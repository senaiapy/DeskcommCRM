# health Specification

## Purpose
Liveness and dependency health of a DeskcommCRM installation: the public `GET /api/v1/health` probe of Supabase, Redis (REST) and WAHA, how the compose healthcheck and the self-host kit consume it, and the platform-admin per-tenant health view. The update flow that gates on this probe is covered by `selfhost-install-update`; the per-number deliverability circuit in `lib/agent-engine/health/` and the `channel-health` cron belong to `whatsapp-waha` and are not specified here.

## Requirements

### Requirement: Public health endpoint probing three dependencies
`GET /api/v1/health` SHALL require no authentication and SHALL run the Supabase, Redis and WAHA checks in parallel, each bounded by a 3000 ms timeout, returning `{ data: { status, version, timestamp, checks: { supabase, redis, waha } } }`.

#### Scenario: External monitor polls the endpoint
- **WHEN** an unauthenticated client calls `GET /api/v1/health`
- **THEN** the response carries `data.status`, `data.version`, `data.timestamp` and one entry per dependency with `status` and `latency_ms`

### Requirement: Aggregate status and HTTP code
The health route SHALL report `unhealthy` with HTTP 503 when any check is `down`, `degraded` with HTTP 200 when none is down but any is `degraded`, and `healthy` with HTTP 200 otherwise.

#### Scenario: Redis down
- **WHEN** the Redis REST endpoint refuses the connection
- **THEN** `checks.redis.status` is `down`, `data.status` is `unhealthy` and the HTTP status is 503

#### Scenario: WAHA not configured
- **WHEN** `WAHA_API_BASE_URL` is empty and the other checks pass
- **THEN** `checks.waha` is `degraded` with `reason: "nao_configurado"` and the HTTP status is 200

### Requirement: Supabase probe hits the same address the server uses
The Supabase check SHALL call `GET <SUPABASE_SERVER_URL or NEXT_PUBLIC_SUPABASE_URL>/rest/v1/organizations?select=id&limit=1` with the anon key and `Accept-Profile: public`, and SHALL treat 200, 401 and 403 as `ok`.

#### Scenario: RLS blocks the anon read
- **WHEN** PostgREST answers 401 to the anon probe
- **THEN** `checks.supabase.status` is `ok`

### Requirement: Redis and WAHA probes distinguish configuration from availability
The Redis check SHALL return `degraded`/`nao_configurado` when `UPSTASH_REDIS_REST_URL` or `UPSTASH_REDIS_REST_TOKEN` is unset, `down`/`configuracao_invalida` when the value is malformed, and otherwise POST `["PING"]` to the REST root; the WAHA check SHALL call `GET <WAHA_API_BASE_URL>/api/sessions` with `X-Api-Key: WAHA_API_KEY` and map 401/403 to `reason: "credencial_recusada"`.

#### Scenario: Quoted Redis URL in .env
- **WHEN** `UPSTASH_REDIS_REST_URL` holds a value that fails `validarConfigRedisRest`
- **THEN** `checks.redis` is `down` with `reason: "configuracao_invalida"` and no network call is made

### Requirement: Targets and raw errors only for the internal secret
The health route SHALL omit `target` and replace any `error` text with `erro_ao_consultar` unless the request has `?verbose=1` and a Bearer (or `x-cron-secret`) equal to `INTERNAL_CRON_SECRET` or `INTERNAL_SECRET` by constant-time comparison, while always returning `reason`.

#### Scenario: Anonymous verbose request
- **WHEN** `GET /api/v1/health?verbose=1` is sent without the internal secret
- **THEN** no check contains `target` and failing checks show `error: "erro_ao_consultar"` with their `reason`

### Requirement: Reported version comes from the image
The health route SHALL report `data.version` from the `APP_VERSION` environment variable baked by the `Dockerfile` build argument, and `"desconhecido"` when it is absent.

#### Scenario: Image built without APP_VERSION
- **WHEN** `APP_VERSION` is unset in the running process
- **THEN** `data.version` is `"desconhecido"`

### Requirement: Container healthcheck is a TCP probe, not the dependency probe
The `app` service in `docker-compose.prod.yml` SHALL use a Node TCP connect to `127.0.0.1:3000` as its Docker healthcheck (interval 30s, retries 5, start_period 40s), and `caddy` SHALL depend on `app` being `service_healthy`, so a down WAHA or Redis does not mark the app unhealthy.

#### Scenario: WAHA container stopped
- **WHEN** WAHA is down but the Next server listens on port 3000
- **THEN** Docker reports `app` healthy and Caddy keeps serving

### Requirement: Kit tools read the probe from inside the network
`wait_app_healthy` in `hostgator-setup-kit/_common.sh` SHALL poll the in-container `/api/v1/health` (default 20 attempts, 3 s apart) and succeed on `healthy` or `degraded`, and `hostgator-setup-kit/healthcheck.sh` SHALL bound `docker compose ps` to 45 s and the in-container fetch to 30 s.

#### Scenario: Degraded after update
- **WHEN** `/api/v1/health` reports `degraded` because Redis is not configured
- **THEN** `wait_app_healthy` returns success and the update is not rolled back

### Requirement: Platform-admin tenant health view
`GET /api/v1/admin/tenants/[id]/health` SHALL return 403 `forbidden` unless `requirePlatformAdmin` passes, SHALL read `channel_sessions`, `tenant_integrations`, `ai_budgets` and `api_audit_log` for the organization id taken from the path, SHALL classify each block as `ok`, `warning` or `critical`, and SHALL audit `platform_admin.tenant_health_viewed` with `bypassedRls: true`.

#### Scenario: Tenant with only failed sessions
- **WHEN** every `channel_sessions` row of the tenant is `FAILED` or `STOPPED`
- **THEN** `waha.overall_status` is `critical`

### Requirement: The diagnostic shows the update agent's last failure on one line
When the update agent's log changed in the last 120 minutes, `hostgator-setup-kit/healthcheck.sh` SHALL print the last log entry starting at its `<timestamp> [agent]` header, joining its non-empty lines into one line cut at 300 characters, instead of a fixed line offset.

#### Scenario: Proxy error body ending in a newline
- **WHEN** the agent's last POST got `no available server` followed by a blank line and the HTTP code
- **THEN** the diagnostic shows time, URL, body and code on one line instead of an empty line
