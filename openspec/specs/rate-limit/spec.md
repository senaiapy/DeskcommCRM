# rate-limit Specification

## Purpose
Fixed-window request counters that protect authentication, bearer integrations, public endpoints and expensive AI calls. The single primitive is `checkRateLimit(bucket, limit, windowSec)` / `peekRateLimit()` in `lib/ai/dispatcher/rate-limit.ts`, backed by the Upstash-compatible Redis REST endpoint (`UPSTASH_REDIS_REST_URL`, `UPSTASH_REDIS_REST_TOKEN`, served by the `srh` container in the self-host stack, shape-checked by `lib/redis-config.ts`) with an in-process memory fallback. Policies on top: `lib/auth/rate-limit.ts` (`AUTH_LIMITS`, `TOKEN_FAILURE_LIMITS`, used by `app/actions/auth/*`), `lib/mcp/rate-limit.ts` (`TETO_POR_TOKEN`, `TETO_POR_ORGANIZACAO`, `TETO_DE_ESCRITA`), `lib/api/auth-dual.ts` (`tetoDeEscritaDoToken`), and per-route budgets in routes such as `app/api/v1/messages/route.ts`, `app/api/v1/ai/knowledge/busca/route.ts`, `app/api/v1/ai/cases/[id]/chat/route.ts`, `app/api/v1/tenants/provision/route.ts`, `app/api/v1/webhooks/in/[token]/route.ts`.

## Requirements

### Requirement: Fixed-window counter in Redis
`checkRateLimit()` SHALL increment the Redis key `<bucket>:<floor(now/windowSec)>`, set its expiry to `windowSec` on the first hit, and report `allowed = count <= limit` together with `count`, `limit` and `window_sec`.

#### Scenario: Limit reached
- **WHEN** a bucket with limit 30 receives its 31st call within the same window
- **THEN** `checkRateLimit()` returns `allowed: false` and `count: 31`

#### Scenario: New window
- **WHEN** the clock crosses into the next `windowSec` slot
- **THEN** the counter starts again from 1 under a new key

### Requirement: Degrade to memory instead of failing
When `UPSTASH_REDIS_REST_URL`/`UPSTASH_REDIS_REST_TOKEN` are malformed or a Redis call throws, the limiter SHALL count in an in-process `Map` and log a warning that the fallback is not safe for multi-instance, with the Redis client built with `retry: false`.

#### Scenario: Redis unreachable
- **WHEN** the Redis REST endpoint refuses connections
- **THEN** requests are still counted per process and none fails because of the limiter

### Requirement: Authentication attempt limits
Server actions for login, signup, password reset and organization recovery SHALL consult `authRateLimited()` with `AUTH_LIMITS` (login 60 per IP — overridable by `AUTH_RATE_LIMIT_LOGIN_IP` — and 5 failures per account per 300 s; signup 20 per IP per hour; reset 30 per IP and 3 per e-mail per hour; org_recovery 5 per IP and 3 per user per hour), skipping the per-IP bucket when no client IP is identifiable.

#### Scenario: Password guessing on one account
- **WHEN** a sixth wrong password for the same e-mail arrives within 5 minutes, from any IP
- **THEN** the login action returns `{ ok: false, error: "rate_limited" }` and audits `auth.login_rate_limited` with only an e-mail hash

#### Scenario: Correct password does not consume the account budget
- **WHEN** a user logs in successfully
- **THEN** the per-account failure counter is not incremented

### Requirement: Bearer and MCP ceilings
Bearer token traffic SHALL be capped per token at 60 calls/min, per organization at 600 calls/min and for writes at 30/min (`lib/mcp/rate-limit.ts`), and bearer writes on REST routes SHALL answer 429 `rate_limited` with `Retry-After` of 60 seconds.

#### Scenario: Integration loops on POST /api/v1/messages
- **WHEN** one `dsk_` token sends more than 30 messages in a minute
- **THEN** the extra requests get 429 `rate_limited` with a `Retry-After` header

### Requirement: Rate-limit headers on 429
Routes that publish per-user budgets (`/api/v1/ai/knowledge/busca`, `/api/v1/ai/cases/{id}/chat`, `/api/v1/tenants/provision`) SHALL return 429 `rate_limited` with `Retry-After`, `X-RateLimit-Limit` and `X-RateLimit-Remaining` headers.

#### Scenario: Knowledge search flood
- **WHEN** a user exceeds the per-user search budget inside the window
- **THEN** the response is 429 with `Retry-After`, `X-RateLimit-Limit` and `X-RateLimit-Remaining: 0`

### Requirement: Identifiers never stored in clear
Authentication bucket keys SHALL embed only an opaque hash of the IP or identifier (`opaque()`), never the raw e-mail, token or IP.

#### Scenario: Inspecting Redis keys
- **WHEN** an operator lists keys starting with `auth:login:`
- **THEN** no key contains an e-mail address or IP literal
