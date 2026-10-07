## Purpose
The wire contract shared by every route under `/api/v1/`: the success and error envelopes built by `ok()` / `fail()` / `noContent()` in `lib/api/wrappers.ts`, the canonical error-code catalog in `lib/api/errors.ts` (with the `ApiError` class in `lib/api/types.ts`), the `X-Request-Id` header, the Postgres-backed `Idempotency-Key` receipt (`lib/api/idempotency.ts`, table `public.idempotency_keys`), keyset cursor pagination implemented per handler (`app/api/v1/contacts/_handler.ts`, `conversations/_handler.ts`, `leads/_handler.ts`, `audit/route.ts`), and the support-session write guard `requireSupportWrite()` (`lib/impersonate/support.ts`) enforced by the gate `tests/unit/suporte-cobertura-de-efeitos.test.ts`.

## ADDED Requirements

### Requirement: Success envelope
A successful `/api/v1/` response built with `ok()` SHALL have the JSON body `{ "data": <payload> }`, plus `"meta"` only when the handler passes one (cursor pages use `meta.cursor` and `meta.has_more`), with status 200 by default, 201 on creation, and 204 with an empty body via `noContent()`.

#### Scenario: Created resource
- **WHEN** a handler calls `ok(row, { status: 201 })`
- **THEN** the response status is 201 and the body is `{ "data": row }` with no `meta` key

#### Scenario: Paginated list
- **WHEN** `GET /api/v1/contacts` returns a page with more rows available
- **THEN** the body carries `data` as an array and `meta.cursor` plus `meta.has_more: true`

### Requirement: Error envelope
A failed `/api/v1/` response built with `fail(code, message, status)` SHALL have the JSON body `{ "error": { "code", "message", "details"? } }`, where `details` appears only when the handler supplies it.

#### Scenario: Not found
- **WHEN** a handler calls `fail("not_found", "...", 404)`
- **THEN** the response status is 404 and the body is `{ "error": { "code": "not_found", "message": "..." } }`

### Requirement: X-Request-Id on every wrapped response
Every response produced by `ok()`, `fail()` or `noContent()` SHALL carry an `X-Request-Id` header, taken from the handler's `requestId` or else a fresh `randomUUID()`, and the request proxy (`proxy.ts`) SHALL set `x-request-id` on the responses it produces, including its own 401 `unauthenticated` JSON.

#### Scenario: Handler without explicit id
- **WHEN** a handler calls `fail("forbidden", "...", 403)` without `requestId`
- **THEN** the response still carries an `X-Request-Id` header holding a UUID

#### Scenario: Proxy refuses an API call
- **WHEN** an unauthenticated request reaches a non-public `/api/` path
- **THEN** the proxy answers 401 with `{"error":{"code":"unauthenticated",...}}` and an `x-request-id` header

### Requirement: Canonical error codes
Error codes SHALL be drawn from the `ApiErrorCodes` map in `lib/api/errors.ts` (e.g. `invalid_request`, `validation_failed`, `invalid_cursor`, `unauthenticated`, `mfa_required`, `forbidden`, `forbidden_role`, `not_found`, `idempotency_conflict`, `idempotency_in_progress`, `rate_limited`, `internal_error`, `upstream_unavailable`), with `fail()` typed to accept that union while still allowing an arbitrary string.

#### Scenario: Zod validation failure
- **WHEN** a request body fails the route's Zod schema on a route such as `PATCH /api/v1/campaign-templates/{id}`
- **THEN** the response is 422 with `error.code = "validation_failed"`

### Requirement: Idempotency-Key receipts in Postgres
Routes that read `Idempotency-Key` SHALL store the receipt in `public.idempotency_keys`, unique on `(organization_id, key, endpoint)` (constraint `idempotency_keys_organization_id_key_endpoint_key`), with `request_hash` (bytea of the canonical body hash) and a 24-hour TTL (`TTL_MS`), replaying the stored `status_code` and `response_body` for the same key and body.

#### Scenario: Same key, same body, within 24h
- **WHEN** `POST /api/v1/message-templates` is repeated with the same `Idempotency-Key` and identical body
- **THEN** the stored status and body are returned and no second template is created

#### Scenario: Same key, different body
- **WHEN** the same `Idempotency-Key` is reused with a different body inside 24h
- **THEN** the response is 409 `idempotency_conflict`

#### Scenario: Concurrent duplicate
- **WHEN** a second request with the same key and body arrives while the first still holds the reservation row with `status_code` null
- **THEN** the response is 409 `idempotency_in_progress`

#### Scenario: Expired receipt
- **WHEN** the same key is sent after `expires_at`
- **THEN** the request is executed as a new operation

### Requirement: Opaque keyset cursor
List handlers that paginate by cursor SHALL encode the last row's sort value and `id` as base64url JSON and SHALL answer 400 `invalid_cursor` when the `cursor` query parameter cannot be decoded.

#### Scenario: Tampered cursor
- **WHEN** `GET /api/v1/audit?cursor=not-base64-json` is requested
- **THEN** the response is 400 with `error.code = "invalid_cursor"`

### Requirement: Support-session write guard before effects
Every mutating handler (POST/PUT/PATCH/DELETE) under `app/api/v1` SHALL call `requireSupportWrite()` before its effect, which returns 403 `forbidden` when the caller's support session (`fn_support_context`) is `support_readonly` or no longer `active`, and 503 `upstream_unavailable` when the support context cannot be read; the gate `tests/unit/suporte-cobertura-de-efeitos.test.ts` SHALL fail when a mutating handler lacks the call outside its declared exceptions.

#### Scenario: Read-only support session tries to write
- **WHEN** a platform admin inside a `support_readonly` support session sends `POST /api/v1/settings/api-tokens`
- **THEN** the response is 403 `forbidden` and no token is created

#### Scenario: Caller without user cookie
- **WHEN** a mutating route is called by a worker or bearer token with no user session
- **THEN** `requireSupportWrite()` returns null and the route's own authorization decides
