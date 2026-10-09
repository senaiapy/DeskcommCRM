# event-bus-workers Specification

## Purpose
The asynchronous backbone: `emit_event` writes `event_log` rows (database triggers never call HTTP), a drain
claims and dispatches them to registered handlers both from the `event-log-drain` cron and from an in-process
loop of the `worker` service, and the `scheduler` service calls every `/api/v1/cron/*` route with the shared
cron secret. The agent turn queue in `job_queue` and its handlers are covered by `ai-agents-runtime`; per-contact
follow-up scheduling is covered by `followup-flows` and `jobs-scheduler`; individual cron bodies are covered by
the capability that owns them (e.g. `lgpd-privacy`, `data-retention`).

## Requirements

### Requirement: Events are emitted into event_log
`public.emit_event(p_event_type, p_entity_kind, p_entity_id, p_payload, p_metadata, p_organization_id)` SHALL insert one `event_log` row with `status = 'pending'`, resolve the organization from the caller's membership when `p_organization_id` is null (raising `emit_event: organization_id obrigatorio` when none), reject an authenticated caller who is not at least `viewer` in that organization (`caller_not_authorized_for_org`), and stamp `metadata.emitted_at`; `event_type` SHALL match `^[a-z][a-z0-9_]*\.[a-z][a-z0-9_]*$`.

#### Scenario: Member of another organization
- **WHEN** an authenticated user calls `emit_event` with the id of an organization they do not belong to
- **THEN** the call raises `caller_not_authorized_for_org` and no row is inserted

#### Scenario: Malformed event type
- **WHEN** `p_event_type` is `ContactAnonymized`
- **THEN** the insert fails the `event_type_format` check

### Requirement: Triggers never call HTTP
No function or trigger in `supabase/baseline.sql` or `supabase/migrations/` SHALL call `net.http_post` or `net.http_get`; side effects SHALL be performed by `event_log` handlers outside the transaction.

#### Scenario: Schema scan
- **WHEN** the baseline and migrations are searched for `net.http_post` or `net.http_get`
- **THEN** there is no occurrence

### Requirement: Drain claims, dispatches and closes each event
`drainEventLog` SHALL select up to 50 `event_log` rows with `status = 'pending'` and `next_attempt_at` null or due, claim each with an optimistic `update ... where status = 'pending'`, run every registered handler whose `events` include the `event_type` and whose `key` is not in `consumed_by`, append successful (`ok`/`skipped`) handler keys to `consumed_by`, and set `status = 'done'` when no handler errored, keeping skip reasons in `last_error`.

#### Scenario: Two drains race
- **WHEN** the cron and the worker loop select the same pending row
- **THEN** only the one whose claim update returns the row dispatches it

#### Scenario: Handler already consumed
- **WHEN** a row is retried and `consumed_by` already contains `lgpd-export-worker.v1`
- **THEN** that handler is not called again

### Requirement: Retry with exponential backoff and dead letter
A handler error SHALL increment `attempts`, store `last_error` and reschedule `next_attempt_at` at 2^attempts minutes; at 5 attempts the row SHALL become `dead` and `avisarEventoMorto` SHALL open an `agent_inbox_items` item of kind `event_dead`; a `retry` result SHALL reschedule at `retry_at` without counting an attempt; rows stuck in `processing` for more than 10 minutes SHALL be returned to `pending` or marked `dead`.

#### Scenario: Fifth failure
- **WHEN** a handler fails for the fifth time on the same event
- **THEN** the row is `dead` and an open `event_dead` item exists in the organization's inbox

#### Scenario: Worker crashed mid-dispatch
- **WHEN** a row has been `processing` with `updated_at` older than 10 minutes
- **THEN** the next drain moves it back to `pending` (or `dead` at the attempt limit)

### Requirement: Stopped organizations are fail-closed
Before claiming a batch the drain SHALL read `organizations.status` for the batch's organizations, treat an organization that is not operating (or not returned) as stopped so handlers declared `naOrgParada: "pula"` record `skipped` with `org_nao_operante`, and postpone the whole batch when that read fails.

#### Scenario: Suspended tenant
- **WHEN** a `message.received` event belongs to a suspended organization
- **THEN** the web-push handler is recorded as `skipped` with detail `org_nao_operante` and no push is sent

### Requirement: Cron entry point for the drain
`GET|POST /api/v1/cron/event-log-drain` SHALL authorize with `autorizaCron`, call `ensureHandlersRegistered()` and return the drain summary (`scanned`, `done`, `failed`, `dead`, `retried`), or 500 `internal_error` when the drain throws.

#### Scenario: Valid tick
- **WHEN** the scheduler calls the route with the bearer secret
- **THEN** the response is 200 with the summary counts

### Requirement: Cron authentication is shared and fail-closed
`autorizaCron` SHALL accept `Authorization: Bearer <secret>` or `x-cron-secret: <secret>`, compare it in constant time against `INTERNAL_CRON_SECRET` and `INTERNAL_SECRET`, and refuse when no header is sent or neither variable is set; every `/api/v1/cron/*` route using it SHALL answer 403 `forbidden` on refusal.

#### Scenario: Secrets not configured
- **WHEN** both `INTERNAL_CRON_SECRET` and `INTERNAL_SECRET` are empty
- **THEN** every cron call is refused with 403 even if a header is sent

### Requirement: Scheduler container drives the crons
The `scheduler` image (`Dockerfile.scheduler`, Alpine `crond`) SHALL exit at start when `INTERNAL_SECRET` is empty, write one crontab line per route in `docker/scheduler/entrypoint.sh` that calls `http://app:3000/api/v1/cron/<route>` with `curl -m<timeout>` and `Authorization: Bearer $INTERNAL_SECRET`, including `event-log-drain` every minute, `storage-redaction` every 5 minutes, `lgpd-sla-watcher` daily at 12:00, `data-retention` daily at 04:40 and `media-retention` daily at 05:20.

#### Scenario: Missing secret
- **WHEN** the scheduler starts without `INTERNAL_SECRET`
- **THEN** it prints an explanation to stderr and exits with status 1

### Requirement: Worker process runs the drain loop and health endpoints
The `worker` image (`Dockerfile.worker`, `tsx workers/agent-worker/main.ts`) SHALL run `runEventLogDrainLoop` with `EVENT_LOG_DRAIN_INTERVAL_MS` (default 2000), `EVENT_LOG_DRAIN_IDLE_INTERVAL_MS` (default 10000) and `EVENT_LOG_DRAIN_BATCH_SIZE` (default 50), serve `GET /healthz` and `GET /metrics` on `HEALTH_PORT` (default 8787) with `job_queue` counts and `event_log_drain` readiness (503 `degraded` when the database is down), and on SIGTERM/SIGINT stop claiming and wait up to `SHUTDOWN_GRACE_MS` (default 30000).

#### Scenario: Database unreachable
- **WHEN** `/healthz` is called while Postgres is down
- **THEN** the response is 503 with `db: "error"` and still includes `event_log_drain`

#### Scenario: Unknown path
- **WHEN** `GET /foo` is sent to the health port
- **THEN** the response is 404 `{error: "not_found"}`

### Requirement: The scheduler keeps the cron secret out of the crontab
`docker/scheduler/entrypoint.sh` SHALL write `Authorization: Bearer $INTERNAL_SECRET` to `$CRON_AUTH_DIR/header` (default `/run/deskcomm-cron/header`, directory mode 700, file mode 600) and every generated crontab line SHALL send it with `curl -H @<that file>`, so the secret never appears in the crontab text.

#### Scenario: Crontab inspected
- **WHEN** an operator prints the scheduler's crontab
- **THEN** the lines reference the header file and do not contain the value of `INTERNAL_SECRET`

### Requirement: The message webhook drains only its own follow-up handlers when a worker drains
When `EVENT_LOG_WORKER_DRAINS` is truthy (`1|true|on|yes|sim`; default `true` in `docker-compose.prod.yml` and `docker-compose.local.yml`, empty in `.env.example`), the in-request drain after an inbound message SHALL call `drainEventLog` with `limit: 10` and `escopo` limited to the inbound organization and the follow-up trigger handlers (`FOLLOWUP_GATILHO_RETORNO_HANDLER_KEY`, `FOLLOWUP_GATILHO_LEAD_HANDLER_KEY`), returning each event to `pending` with the scoped keys appended to `consumed_by` (counted as `deixados_ao_worker`, never as a failed attempt) so the worker loop runs the remaining handlers; when the variable is falsy the request drains everything as before.

#### Scenario: Production stack
- **WHEN** a WhatsApp message arrives on an installation running the `worker` service with default settings
- **THEN** the webhook request runs only the follow-up trigger handlers of that organization and the other consumers of `message.received` are run by the worker loop

#### Scenario: Scoped handler fails
- **WHEN** a scoped follow-up handler errors inside the request
- **THEN** the event's `attempts` is not incremented and no backoff is applied
