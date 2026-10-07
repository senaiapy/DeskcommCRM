## Purpose
Official WhatsApp channel through the Meta WhatsApp Cloud API (provider `meta_cloud`). An admin connects a number on the settings page `app/app/settings/canal-oficial` via `GET/POST /api/v1/channels/official` (credentials validated, token encrypted into `channel_sessions.meta_token_encrypted`) and registers the per-number webhook override via `POST /api/v1/channels/official/webhook`. Meta calls `GET/POST /api/v1/webhooks/meta/[token]` (verify handshake + `X-Hub-Signature-256` HMAC SHA256), parsed by `lib/channels/meta/webhook.ts`/`envelope.ts` and ingested by `lib/channels/meta/ingest.ts`; delivery statuses map through `lib/channels/meta/status-update.ts`. Approved templates are synced into `meta_templates` via `/api/v1/channels/templates`. Graph API version comes from `lib/graph-version.ts` (`META_GRAPH_VERSION`, default `v22.0`) and host from `lib/channels/meta/graph-base.ts` (`META_GRAPH_BASE_URL`). The 24-hour customer-service window is computed by `lib/channels/janela.ts` and enforced in `app/api/v1/messages/_handler.ts` (`janela_fechada`). App credentials: `META_APP_SECRET`, `META_WEBHOOK_VERIFY_TOKEN` (or their encrypted installation-level equivalents, `lib/channels/meta/app.ts`).

## ADDED Requirements

### Requirement: Webhook verification handshake
`GET /api/v1/webhooks/meta/[token]` SHALL return the `hub.challenge` value as `text/plain` with status 200 only when `hub.mode = subscribe` and `hub.verify_token` equals the configured verify token, 403 otherwise, and 404 when the path token matches no Meta session.

#### Scenario: Valid handshake
- **WHEN** Meta calls the GET with `hub.mode=subscribe`, the correct `hub.verify_token` and `hub.challenge=123`
- **THEN** the response is 200 with body `123` and `content-type: text/plain`

#### Scenario: Wrong verify token
- **WHEN** the `hub.verify_token` does not match
- **THEN** the response is 403

### Requirement: Webhook signature verification
`POST /api/v1/webhooks/meta/[token]` SHALL verify `X-Hub-Signature-256` as `sha256=<hex HMAC-SHA256 of the raw body with the app secret>` using a length check plus `timingSafeEqual`, and respond 401 `unauthorized` (`invalid_signature`) when it is missing or does not match.

#### Scenario: Missing signature
- **WHEN** a POST arrives without `X-Hub-Signature-256`
- **THEN** the response is 401 with error code `unauthorized`

### Requirement: Always-200 after authentication
After a valid signature and a contract-valid envelope, the Meta webhook SHALL respond 200 with `{ received, outcomes }` for every event batch (including ignored events and events whose `wabaId` differs from the session's), while non-JSON bodies return 400 `invalid_request` and out-of-contract payloads return 400 `validation_failed`.

#### Scenario: Event of another WABA
- **WHEN** a signed event carries a `wabaId` different from the session's `meta_waba_id`
- **THEN** the event is skipped and the response is still 200

### Requirement: Idempotent inbound ingest
`ingestMetaInbound` and `ingestMetaEcho` SHALL insert into `messages` keyed by `(organization_id, external_id)` and treat Postgres error `23505` as `status: "duplicate"`, and inbound messages SHALL run the shared post-ingest effects (`aplicarEfeitosPosEntrada`: opt-out, lead, agent dispatch).

#### Scenario: Meta redelivers a message
- **WHEN** the same `wamid` is delivered twice
- **THEN** one `messages` row exists and the second outcome is `duplicate`

### Requirement: Delivery status and failure events
Status events SHALL update the matching `messages` row by `(organization_id, external_id)` via `statusUpdate`, and a `failed` status SHALL update only rows not already `failed` and emit a delivery-failure event once per message; `template_status` events SHALL update `meta_templates.status` and `rejected_reason`.

#### Scenario: Repeated failed status
- **WHEN** Meta delivers the same `failed` status twice for one message
- **THEN** the message ends with `status = 'failed'` and only one delivery-failure event is emitted

### Requirement: Connecting the official number
`POST /api/v1/channels/official` SHALL require `requireSupportWrite` and role `admin`, reject a body without `phone_number_id`, `waba_id` and `token` with 422 `invalid_request`, validate the credentials against the Graph API before writing (422 `invalid_request` with the reason on failure), refuse with 422 when the token cannot be encrypted, and store the session with `provider = 'meta_cloud'`, `meta_phone_number_id`, `meta_waba_id` and `meta_token_encrypted`.

#### Scenario: Invalid credentials
- **WHEN** an admin posts a token the Graph API rejects
- **THEN** the response is 422 `invalid_request` and no `channel_sessions` row is written

#### Scenario: Non-admin caller
- **WHEN** a user with role `agent` calls `POST /api/v1/channels/official`
- **THEN** the request is refused by `requireRole("admin")` with 403

### Requirement: 24-hour window enforcement for integrations
`POST /api/v1/messages` SHALL reject a non-template message from an `api_token` or `ai_agent` actor with 422 `janela_fechada` (details include `ultima_mensagem_do_cliente`, `use: "template"`, `codigo_plataforma: "131047"`) when the conversation's channel provider does not allow free-form outside the window and `last_inbound_at` is null or older than 24 hours.

#### Scenario: Free text after 25 hours
- **WHEN** an API token posts a text message on a `meta_cloud` conversation whose last inbound was 25 hours ago
- **THEN** the response is 422 with error code `janela_fechada` and no message row is created

#### Scenario: Template outside the window
- **WHEN** the same token posts with `type = "template"`
- **THEN** the window check does not block the request

### Requirement: Graph API version and host
Graph API calls SHALL use the version from `META_GRAPH_VERSION` when it is non-empty and `v22.0` otherwise (`graphVersion()`), and the host from `META_GRAPH_BASE_URL` with fallback `https://graph.facebook.com`.

#### Scenario: Empty version variable
- **WHEN** `META_GRAPH_VERSION` is set to an empty string
- **THEN** `graphVersion()` returns `v22.0`

### Requirement: Template listing and sync permissions
`GET /api/v1/channels/templates` SHALL require role `agent` and return the organization's `meta_templates` rows, while `POST` (sync from Meta) and `PATCH` (saved media values) SHALL require `requireSupportWrite` and role `admin`, returning 400 `invalid_request` (`no_meta_channel` / `missing_meta_token`) when the organization has no usable Meta channel.

#### Scenario: Sync without Meta channel
- **WHEN** an admin calls `POST /api/v1/channels/templates` in an organization without a `meta_cloud` session
- **THEN** the response is 400 `invalid_request` with message `no_meta_channel`
