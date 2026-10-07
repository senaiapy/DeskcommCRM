## Purpose
Server-to-server authentication with organization-scoped bearer tokens. Admins issue and revoke tokens in Configurações › Tokens de API (`app/app/settings/api-tokens/page.tsx`) through `GET|POST /api/v1/settings/api-tokens` and `POST /api/v1/settings/api-tokens/{id}/revoke`; tokens live in `public.api_tokens` (only the SHA256 hash is stored); `lib/mcp/auth.ts` (`resolveApiToken`, `validateBearerToken`) validates a `dsk_` bearer, and `lib/api/auth-dual.ts` (`resolveAuthDual`, `tetoDeEscritaDoToken`) lets an individual `/api/v1` route accept either the session cookie or the bearer. Because `proxy.ts` only understands cookies, a bearer-enabled route must also be listed in `lib/auth/public-paths.ts`.

## ADDED Requirements

### Requirement: Token issuance by admins
`POST /api/v1/settings/api-tokens` SHALL require role `admin`, accept `{ name (2–100 chars), scopes (≥1 string), expires_in_days? (1–365) }`, generate a plaintext of the form `dsk_<8 hex>_<32 random bytes base64url>`, store only its SHA256 digest in `api_tokens.token_hash` with the `prefix`, and return 201 with the plaintext exactly once, auditing `token.created`.

#### Scenario: Admin creates a token
- **WHEN** an org admin posts a valid body
- **THEN** the response is 201 whose `data.plaintext` starts with `dsk_`, the row in `api_tokens` holds `token_hash` but no plaintext, and `api_audit_log` gets `token.created`

#### Scenario: Manager tries to create
- **WHEN** a `manager` posts to the same route
- **THEN** the response is 403 and no row is inserted

#### Scenario: Listing never returns secrets
- **WHEN** an admin calls `GET /api/v1/settings/api-tokens`
- **THEN** each item carries `id, name, prefix, scopes, last_used_at, expires_at, revoked_at, created_at` and no hash or plaintext

### Requirement: Active-token ceiling per organization
The trigger `trg_teto_de_tokens_ativos` (migration 0415) SHALL refuse an INSERT into `api_tokens` when the organization already has 50 tokens with `revoked_at` null and not expired, raising SQLSTATE `PT409`, which the issuance route returns as 409 `api_token_teto_atingido`.

#### Scenario: 51st active token
- **WHEN** an organization holding 50 active tokens issues another
- **THEN** the response is 409 `api_token_teto_atingido` and no row is created

### Requirement: Revocation
`POST /api/v1/settings/api-tokens/{id}/revoke` SHALL require role `admin`, look the token up filtered by the session's `organization_id`, set `revoked_at` and `revoked_by`, audit `token.revoked`, and answer 200 with `already_revoked: true` without change when the token was already revoked.

#### Scenario: Token of another organization
- **WHEN** an admin revokes a token id belonging to a different organization
- **THEN** the response is 404 `not_found`

#### Scenario: Repeated revoke
- **WHEN** the same token is revoked twice
- **THEN** the second call returns 200 with `already_revoked: true`

### Requirement: Bearer validation
`resolveApiToken()` SHALL reject a value not starting with `dsk_`, look up `api_tokens` by the SHA256 of the plaintext, reject a row with `revoked_at` set or `expires_at` in the past, reject a token whose organization is suspended, and update `last_used_at` only on success.

#### Scenario: Revoked token
- **WHEN** a request presents `Authorization: Bearer dsk_...` for a revoked token
- **THEN** the response is 401 and `last_used_at` is unchanged

#### Scenario: Suspended organization
- **WHEN** a valid token belongs to an organization whose status is suspended
- **THEN** the response is 403 with code `org_suspended`

### Requirement: Role and actor derived from scopes
The token's role SHALL be the highest valid `role:<name>` entry in `api_tokens.scopes` (defaulting to `agent`), and the scope `actor:ai_agent` SHALL make the actor an AI agent instead of an integration.

#### Scenario: Token without role scope
- **WHEN** a token's scopes contain no `role:` entry
- **THEN** it is treated as role `agent`

### Requirement: Dual authentication per route
A route using `resolveAuthDual()` SHALL, when an `Authorization: Bearer` header is present, validate the token, require the route's `scope` and minimum role (403 `forbidden_role` otherwise) and take `organization_id` from the token row, never from the request; without a bearer it SHALL fall back to `requireRole()` on the session cookie.

#### Scenario: Token lacking scope
- **WHEN** a valid token without the route's required scope calls a bearer-enabled route
- **THEN** the response is 403 `forbidden_role`

#### Scenario: Body tries to choose the tenant
- **WHEN** a bearer request body contains a different `organization_id`
- **THEN** the handler acts on the token's organization only

### Requirement: Bearer routes must be public to the proxy
Every path that accepts a bearer SHALL have a pattern in `PUBLIC_PATHS` of `lib/auth/public-paths.ts` (e.g. `/api/v1/contacts`, `/api/v1/messages`, `/api/v1/leads/{uuid}`, `/api/v1/agenda/agendamentos`), because `proxy.ts` answers 401 `unauthenticated` to any non-public path without a session cookie.

#### Scenario: Bearer on a non-listed route
- **WHEN** a bearer-only request hits an `/api/v1` path absent from `PUBLIC_PATHS`
- **THEN** the proxy answers 401 `unauthenticated` before the handler runs

### Requirement: Failed-token throttling and write ceilings
`validateBearerToken()` SHALL refuse with 429 before any lookup when the failure counters exceed `TOKEN_FAILURE_LIMITS` (30 failures per IP or 5 per presented value in 300 s), and `tetoDeEscritaDoToken()` SHALL cap bearer writes at 30 per token and 600 per organization per 60-second window per route bucket, answering 429 `rate_limited` with `Retry-After: 60`.

#### Scenario: Token write burst
- **WHEN** one token sends a 31st write to the same bearer-enabled route inside 60 seconds
- **THEN** the response is 429 `rate_limited` with header `Retry-After: 60`

#### Scenario: Session callers are not throttled
- **WHEN** the same route is called with a session cookie
- **THEN** `tetoDeEscritaDoToken()` returns null and no counter is touched
