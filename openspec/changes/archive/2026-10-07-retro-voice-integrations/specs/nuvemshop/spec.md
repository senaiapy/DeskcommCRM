## Purpose
Connection of an organization to a Nuvemshop store: OAuth install with a signed `state`, encrypted token storage in
`tenant_integrations`, automatic subscription of eight store webhooks, an HMAC-verified operational webhook receiver that
logs and re-emits events, and the three Nuvemshop LGPD webhooks that open `lgpd_requests` with their SLA. The processing of
data-subject requests after they are opened (redaction worker, export, SLA alarms) is covered by `lgpd`; the event_log
drain by the platform queue capability; products/orders display by the commerce capabilities.

## ADDED Requirements

### Requirement: Installation credentials gate
`getConfig` (lib/nuvemshop/config.ts) SHALL return null unless `NUVEMSHOP_APP_ID`, `NUVEMSHOP_CLIENT_ID` and `NUVEMSHOP_CLIENT_SECRET` are all set, and the OAuth callback SHALL then redirect to `/app/integrations/nuvemshop?error=not_configured`.

#### Scenario: Missing client secret
- **WHEN** `NUVEMSHOP_CLIENT_SECRET` is empty and the callback is hit
- **THEN** the browser is redirected with `error=not_configured` and nothing is stored

### Requirement: Admin-only connect with signed state
The `connectNuvemshop` server action SHALL refuse support read-only sessions and non-admins of the active organization with `forbidden`, and SHALL redirect to the Nuvemshop authorize URL with a `state` HMAC-SHA256 signed by `INTERNAL_SECRET` that carries the organization, user and auth session and expires after 10 minutes.

#### Scenario: Manager tries to connect
- **WHEN** a user with role `manager` invokes `connectNuvemshop`
- **THEN** the action returns `{ ok: false, error: "forbidden" }` and no state is issued

### Requirement: OAuth callback validation order
`GET /api/v1/integrations/nuvemshop/callback` SHALL verify `state` (timing-safe) before reading `code`, audit `nuvemshop.oauth_failed` with the reason on every failure, and redirect with `error=invalid_state`, `missing_code`, the token-exchange error, `encrypt_failed` or `db_upsert_failed`.

#### Scenario: Expired state
- **WHEN** the callback receives a `state` older than 10 minutes
- **THEN** the browser is redirected to `/app/integrations/nuvemshop?error=invalid_state` and the code is not exchanged

### Requirement: Encrypted integration row
On a successful exchange the callback SHALL encrypt the access token and the app client secret with `fn_encrypt_oauth` and upsert `tenant_integrations` on `(organization_id, provider)` with `provider='nuvemshop'`, `status='healthy'`, `store_metadata.store_id`, `oauth_access_token_encrypted` and `webhook_secret_encrypted`, taking `organization_id` from the verified `state`.

#### Scenario: Reconnecting the same store
- **WHEN** an organization that already has a Nuvemshop row completes OAuth again
- **THEN** the existing `tenant_integrations` row is updated instead of a second one being created

### Requirement: Webhook subscription on connect
After the upsert the callback SHALL register `order/created`, `order/updated`, `order/paid`, `order/cancelled`, `product/created`, `product/updated`, `product/deleted` and `app/uninstalled` at `<NEXT_PUBLIC_APP_URL>/api/v1/webhooks/nuvemshop/<event-with-dash>`, best-effort, storing each result in `tenant_integrations.webhook_subscriptions` and auditing `nuvemshop.connected` with registered and failed lists.

#### Scenario: One webhook registration fails
- **WHEN** the store API rejects the `product/deleted` subscription
- **THEN** the connection still completes with `?ok=1` and `webhook_subscriptions["product/deleted"]` has `id: null` and the error

### Requirement: Operational webhook receiver
`POST /api/v1/webhooks/nuvemshop/{event}` SHALL answer 404 `not_found` for an unknown slug, 400 `invalid_request` for invalid JSON or missing `store_id`, resolve the tenant by `tenant_integrations.store_metadata->>store_id`, and answer 401 `unauthenticated` when `x-linkedstore-hmac-sha256` does not match the HMAC-SHA256 of the raw body with the decrypted secret.

#### Scenario: Forged signature
- **WHEN** a request for a known store carries a wrong `x-linkedstore-hmac-sha256`
- **THEN** the response is 401, `nuvemshop.webhook_invalid_signature` is audited and nothing is logged in `webhook_events_log`

#### Scenario: Unknown store
- **WHEN** the `store_id` matches no Nuvemshop integration
- **THEN** the response is 200 with `accepted: false` and reason `tenant_not_found`

### Requirement: Idempotent webhook log and event emission
A valid operational webhook SHALL be de-duplicated by `webhook_events_log.external_id` `<event>:<store_id>:<id>` per organization, inserted with `status='received'` and `valid_signature=true` without `authorization`/`cookie` headers, and emitted through `emit_event` as `nuvemshop.<event_with_underscore>`.

#### Scenario: Duplicate delivery
- **WHEN** the same `order/paid` for the same store and order id arrives twice
- **THEN** the second response is `accepted: true, idempotent: true` and no second event is emitted

### Requirement: LGPD webhooks open requests with SLA
`POST /api/v1/webhooks/nuvemshop/customer-data-request`, `customer-redact` and `store-redact` SHALL verify the same HMAC (401 `unauthenticated`), answer 404 `not_found` when no integration matches the store, de-duplicate on `webhook_events_log` unique violation (23505), and create `lgpd_requests` with `source='nuvemshop'` and `requestType` `data_request` (7 business days), `redact` (15) or `store_redact` (15), emitting `lgpd.data_request_received` or `lgpd.redact_received`.

#### Scenario: Customer data request
- **WHEN** a correctly signed `customer-data-request` arrives for a connected store
- **THEN** an `lgpd_requests` row with request type `data_request` and a due date 7 business days ahead is created and the response carries `request_id`

### Requirement: Disconnect marks the integration
The `disconnectNuvemshop` server action SHALL require an admin of the active organization, answer `not_connected` when no row exists, and set `tenant_integrations.status='disconnected'` with `status_reason='user_disconnected'`, auditing `nuvemshop.disconnected`.

#### Scenario: Disconnect without integration
- **WHEN** an admin of an organization without a Nuvemshop row disconnects
- **THEN** the action returns `{ ok: false, error: "not_connected" }`
