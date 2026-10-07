# whatsapp-waha Specification

## Purpose
WhatsApp over WAHA (QR / pairing-code sessions) is the primary messaging channel. Connections live in `channel_sessions` and are managed through `app/api/v1/channel-sessions/*` (create, QR, pairing code, reconnect, archive/delete, groups, AI access, history toggle). WAHA delivers events to `POST /api/v1/webhooks/waha/[token]` (per-session, canonical) or `POST /api/v1/webhooks/waha` (global, resolved by `body.session`), authenticated by `lib/waha/webhook-auth.ts` (HMAC SHA512), contract-checked by `lib/waha/envelope.ts`, cached by `lib/waha/sessao-do-webhook.ts`, archived in `webhook_events_log` and ingested by `lib/waha/ingest.ts` (`dispatchWahaEvent`). Groups are opt-in via `channel_session_groups` and ingested by `lib/grupos/ingest.ts`. Outbound media is stored in the private Supabase Storage bucket `whatsapp-media`. Anti-ban pacing is computed by `lib/agent-engine/pacing/engine.ts` with defaults in `lib/agent-engine/pacing/defaults.ts`, overridable per number in `channel_knobs`. The cron `app/api/v1/cron/recover-stuck-messages` fails outbound messages stuck in `sending`. Env vars: `WAHA_API_BASE_URL`, `WAHA_API_KEY`, `WAHA_WEBHOOK_BASE_URL`, `WAHA_HMAC_SECRET`, `WAHA_WEBHOOK_REQUIRE_SIGNATURE`, `WAHA_BYO_ENCRYPTION_KEY`.

## Requirements

### Requirement: Webhook HMAC authentication is fail-closed
The WAHA webhook routes SHALL verify the `X-Webhook-Hmac` header as a hex HMAC-SHA512 of the raw body compared with `timingSafeEqual`, using the decrypted per-session secret (or `WAHA_HMAC_SECRET` when the session secret is shorter than 16 chars), and SHALL respond 401 `unauthenticated` with an audit row `webhook.hmac_invalid` when a signature is present but does not match, or when no signature is sent while signature is required (`WAHA_WEBHOOK_REQUIRE_SIGNATURE=true` or the installation setting).

#### Scenario: Wrong signature is rejected
- **WHEN** a POST to `/api/v1/webhooks/waha/[token]` carries an `X-Webhook-Hmac` that does not match the body
- **THEN** the response is 401 with reason `bad_signature`
- **AND** an audit row with action `webhook.hmac_invalid` is written for the session's organization

#### Scenario: Unsigned event when signature is not required
- **WHEN** an event arrives without `X-Webhook-Hmac` and signature is not required
- **THEN** the event is processed and archived in `webhook_events_log` with `valid_signature = false`

#### Scenario: Unsigned event when signature is required
- **WHEN** `WAHA_WEBHOOK_REQUIRE_SIGNATURE=true` and an event arrives without signature
- **THEN** the response is 401 with reason `signature_required`

### Requirement: Webhook tenant resolution and archival
The per-session route `POST /api/v1/webhooks/waha/[token]` SHALL resolve the `channel_sessions` row by its path token (404 `not_found` for unknown tokens or tokens shorter than 8 chars), and every authenticated event SHALL be archived in `webhook_events_log` with `organization_id`, `channel_session_id`, `provider = 'waha'`, headers without `authorization`/`cookie`, and the raw body, before ingestion.

#### Scenario: Unknown token
- **WHEN** a POST is made to `/api/v1/webhooks/waha/unknown-token`
- **THEN** the response is 404 with error code `not_found`

#### Scenario: Global route with unregistered session
- **WHEN** `POST /api/v1/webhooks/waha` receives a `session` name with no non-archived `channel_sessions` row
- **THEN** the response is 200 with `accepted: false` and `reason: "session_not_registered"`

### Requirement: Payload contract check
The WAHA webhook routes SHALL validate the envelope with `lib/waha/envelope.ts` and respond 400 `invalid_request` (`invalid_json`) for non-JSON bodies and 400 `validation_failed` with `details.campos` (field names only) for out-of-contract payloads, archiving the content-stage rejection in `webhook_events_log` with `status = 'error'`.

#### Scenario: Payload with a field of the wrong type
- **WHEN** an authenticated event has a `payload` field outside the channel contract
- **THEN** the response is 400 `validation_failed` listing the offending field names
- **AND** the `webhook_events_log` row is written with `status = 'error'`

### Requirement: Idempotent message ingest
Inbound and outbound-from-phone WAHA messages SHALL be inserted into `messages` keyed by the constraint `messages_org_external_id_unique (organization_id, external_id)`, treating Postgres error `23505` as an already-ingested duplicate, and a transient database failure SHALL make the webhook respond 503 `upstream_unavailable` with a `Retry-After` header so WAHA redelivers.

#### Scenario: Redelivered message
- **WHEN** WAHA delivers the same message id twice for the same organization
- **THEN** only one `messages` row exists and the second delivery returns 200 `accepted: true`

#### Scenario: Database unavailable during ingest
- **WHEN** ingestion fails with a transient database error
- **THEN** the response is 503 `upstream_unavailable` with `Retry-After`

### Requirement: Event dispatch
`dispatchWahaEvent` SHALL route `message`/`message.any` (split by `fromMe` into inbound vs outbound-from-phone), `message.ack`, `message.edited`, `message.revoked`, and `session.status`/`state.change` to their handlers, ignoring other event types.

#### Scenario: Ack updates delivery state
- **WHEN** a `message.ack` event arrives for a known message
- **THEN** it is handled by the ack handler and no new `messages` row is created

### Requirement: Session lifecycle endpoints
The channel-session API SHALL require role `admin` (plus `requireSupportWrite` and MFA when owed, 403 `mfa_required`) for `POST /api/v1/channel-sessions` (201 on create, 200 on `Idempotency-Key` replay), `DELETE /api/v1/channel-sessions/[id]`, `POST /api/v1/channel-sessions/[id]/reconnect` and `POST /api/v1/channel-sessions/[id]/pairing-code`, and `GET /api/v1/channel-sessions/[id]/qr` SHALL proxy the WAHA QR image, returning 409 with `x-channel-state: archived|no-session` for archived or session-less channels and 503 when `WAHA_API_BASE_URL`/`WAHA_API_KEY` are not configured.

#### Scenario: Create replay
- **WHEN** an admin repeats `POST /api/v1/channel-sessions` with the same `Idempotency-Key`
- **THEN** the response is 200 with the same channel instead of 201

#### Scenario: QR for archived channel
- **WHEN** `GET /api/v1/channel-sessions/[id]/qr` targets an archived channel
- **THEN** the response is 409 with header `x-channel-state: archived`

### Requirement: Pairing code
`requestChannelPairingCode` (`lib/channels/pairing-code.ts`) SHALL reuse the existing WAHA session (never create, log out or restart it), rate-limit to 1 request per 30 s per organization+channel (429 `rate_limited`, `Retry-After: 30`), reject a session already `WORKING` with 409 `channel_already_connected`, reject a session not in `SCAN_QR_CODE` with 409 `pairing_not_ready`, and return the code formatted as `XXXX-XXXX`, auditing `channel.pairing_code_requested`.

#### Scenario: Already connected number
- **WHEN** an admin requests a pairing code for a session whose WAHA status is `WORKING`
- **THEN** the response is 409 `channel_already_connected`

#### Scenario: Second request within 30 seconds
- **WHEN** a second pairing-code request for the same channel arrives within 30 s
- **THEN** the response is 429 `rate_limited` with `Retry-After: 30`

### Requirement: Group opt-in and isolation
Group messages (`@g.us` chats) SHALL only be ingested when a `channel_session_groups` row with `enabled = true` exists for (organization, session, group_chat_id); such messages create/reuse a contact with `kind = 'whatsapp_group'`, store the sender (from `p.participant`/`p.author`) in `messages.metadata.group_sender` validated by `lib/messaging/remetente-de-grupo.ts`, skip opt-out/lead/AI post-ingest effects, and the trigger `fn_emit_message_event` SHALL emit `message.group_received` instead of `message.received`.

#### Scenario: Disabled group is discarded
- **WHEN** a message arrives from a group with no enabled `channel_session_groups` row
- **THEN** no `messages` row is written

#### Scenario: Enabled group message
- **WHEN** a message arrives from an enabled group
- **THEN** a `messages` row is written in the group conversation and the emitted event is `message.group_received`

### Requirement: Group toggle API
`GET` and `PUT /api/v1/channel-sessions/[id]/groups` SHALL require role `manager` (PUT also `requireSupportWrite`), the PUT body SHALL match `group_chat_id` `^[\d-]+@g\.us$` (400 `validation_error` otherwise), and only the service role may write `channel_session_groups` (RLS grants members SELECT only; insert/update/delete are revoked from `authenticated`), with audits `channel.group_enabled` / `channel.group_disabled`.

#### Scenario: Unknown session
- **WHEN** a manager PUTs a group toggle for a session that does not exist in the organization
- **THEN** the response is 404 `sessao_nao_encontrada`

### Requirement: Anti-ban pacing defaults and per-number knobs
`decidePacing` SHALL veto sends outside the send window (`outside_window`, default 07:00–22:00 in the tenant timezone `America/Sao_Paulo`) and, where ban risk applies, enforce warm-up/daily caps (`warmup_cap`, `daily_cap` from `channel_sessions.daily_message_limit`) and a minimum gap of `throttle_ms` (default 1200 ms) plus random jitter up to `jitter_max_ms` (default 800 ms), with NULL columns in `channel_knobs (organization_id, channel_session_id)` falling back to `PACING_DEFAULTS`.

#### Scenario: Send at 23:00 local time
- **WHEN** a dispatch is evaluated at 23:00 in the tenant timezone with default knobs
- **THEN** the decision is `allow: false` with code `outside_window` and `nextAllowedAt` at the next 07:00 plus jitter

#### Scenario: Back-to-back sends
- **WHEN** the previous send on the same number happened 200 ms ago
- **THEN** the decision is `allow: true` with `waitMs` of at least 1000 ms

### Requirement: Outbound media via Storage
`POST /api/v1/conversations/[id]/media` SHALL upload the outbound file to the private bucket `whatsapp-media` under `{organization_id}/{conversation_id}/out-{uuid}.{ext}` and return `storage_path`, `media_mime` and `media_size_bytes`, rejecting files above 50 MB with 413 `payload_too_large` and a missing `file` field with 422 `validation_failed`.

#### Scenario: Oversized upload
- **WHEN** a file larger than 50 MB is posted
- **THEN** the response is 413 `payload_too_large`

### Requirement: Recover stuck outbound messages
The cron `GET/POST /api/v1/cron/recover-stuck-messages` (Bearer `INTERNAL_CRON_SECRET`/`INTERNAL_SECRET`, 403 `forbidden` otherwise) SHALL mark outbound `messages` with `status = 'sending'` older than 5 minutes as `status = 'failed'` with `error_code = 'send_timeout'`, emit `message.failed`, and insert one `agent_inbox_items` row of kind `message_send_stuck` per affected organization per run, without resending anything.

#### Scenario: Message stuck for six minutes
- **WHEN** the cron runs and an outbound message has been `sending` for 6 minutes
- **THEN** that message becomes `failed` with `error_code = 'send_timeout'`
- **AND** an `agent_inbox_items` row with `kind = 'message_send_stuck'` is created for its organization

#### Scenario: Missing cron secret
- **WHEN** the cron is called without a valid Bearer secret
- **THEN** the response is 403 `forbidden`
