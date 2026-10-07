## Purpose
Voice calls over WhatsApp through the WaCalls service (compose profile `voz`): a second linked WhatsApp device is paired per
organization by QR, and agents dial, answer, reject and hang up 1:1 calls recorded in `voice_calls` (`provider` = wacalls).
The feature is opt-in per organization (`org_voice_calls`) on top of the installation capability (`WACALLS_API_BASE_URL`).
WhatsApp text messaging through WAHA is covered by the messaging capabilities (`whatsapp-channel`); SIP telephony through
Asterisk, which shares the `voice_calls` table, is covered by `voip-asterisk`.

## ADDED Requirements

### Requirement: Installation capability gate
The WaCalls client (`getWacallsClient` in lib/wacalls/client.ts) SHALL be unavailable when `WACALLS_API_BASE_URL` or `WACALLS_API_TOKEN` is empty, and every `/api/v1/voice/*` effect route SHALL then answer 503 `wacalls_not_configured` (or `voice_indisponivel_na_instalacao`).

#### Scenario: Install without the voz profile
- **WHEN** `WACALLS_API_BASE_URL` is empty and an agent calls `POST /api/v1/voice/calls`
- **THEN** the response is 503 with error code `wacalls_not_configured` and nothing is dialed

### Requirement: Per-organization opt-in with explicit risk acceptance
`PUT /api/v1/voice/opt-in` SHALL require role `admin`, SHALL reject `enabled: true` without `riscoAceito: true` with 422 `voice_risco_nao_aceito`, and SHALL upsert `org_voice_calls` (`enabled`, `risco_aceito_em`, `risco_aceito_por`) keyed by the session's active `organization_id`.

#### Scenario: Enabling without accepting the risk
- **WHEN** an admin sends `{ "enabled": true }` without `riscoAceito`
- **THEN** the response is 422 `voice_risco_nao_aceito` and `org_voice_calls` is unchanged

#### Scenario: Reading the state
- **WHEN** a manager calls `GET /api/v1/voice/opt-in`
- **THEN** the response reports the organization choice and the installation capability separately, with `podeEditar` true only for admins

### Requirement: Disabling unpairs the device first
When `PUT /api/v1/voice/opt-in` receives `enabled: false` and the WaCalls client exists, the route SHALL log out and delete the WaCalls session and archive the `channel_sessions` row (`archived_at`) before writing `enabled=false`, returning 502 `wacalls_error` without writing if the unpair fails.

#### Scenario: Unpair failure keeps the feature on
- **WHEN** the WaCalls logout fails while an admin disables voice
- **THEN** the response is 502 `wacalls_error` and `org_voice_calls.enabled` stays true

### Requirement: Consent guard on pairing and dialing
`POST /api/v1/voice/sessions/pair` and `POST /api/v1/voice/calls` SHALL call `exigirVozLigada` (lib/voice/guarda.ts), refusing with 422 `voice_desligada_na_organizacao` when the organization has not opted in and 503 `voice_estado_indeterminado` when the choice cannot be read.

#### Scenario: Organization never opted in
- **WHEN** an admin calls `POST /api/v1/voice/sessions/pair` for an organization without an enabled `org_voice_calls` row
- **THEN** the response is 422 `voice_desligada_na_organizacao` and no WaCalls session is created

### Requirement: QR pairing relay over SSE
`GET /api/v1/voice/events` SHALL require role `admin` (platform admin allowed), proxy the WaCalls `/api/events` stream, and forward only `qr` (as a PNG data URL), `paired` and `expired` events of the session that belongs to the active organization (by stored session id or by the deterministic name from `nomeDaSessaoDeVoz`).

#### Scenario: Another organization pairing at the same time
- **WHEN** WaCalls emits an `auth-state` QR for a session whose id and name do not belong to the caller's organization
- **THEN** the event is not forwarded to the caller's stream

### Requirement: Outbound call creation
`POST /api/v1/voice/calls` SHALL require role `agent`, read the destination phone from `contacts` scoped by the active `organization_id` (never from the body), and insert a `voice_calls` row with `direction='outbound'`, `status='starting'`, `created_by` and `owner_user_id` set to the caller, returning 201.

#### Scenario: Blocked, personal or anonymized contact
- **WHEN** the target contact has `is_blocked` or `is_personal` true
- **THEN** the response is 403 `forbidden`, and an anonymized contact gets 422 `contact_anonymized`, with no call started

#### Scenario: Organization not paired
- **WHEN** voice is enabled but no unarchived `channel_sessions` row with `provider='wacalls'` exists
- **THEN** the response is 409 `wacalls_not_paired`

#### Scenario: Event bridge wrote the row first
- **WHEN** the insert hits the unique constraint `(organization_id, wacalls_call_id)` with code 23505
- **THEN** the existing row is updated with `direction='outbound'`, `contact_id`, `created_by` and `owner_user_id` and the response is still 201

### Requirement: Answer ownership
`POST /api/v1/voice/calls/{id}/accept` SHALL require role `agent` and SHALL answer 409 `voice_call_taken` when `voice_calls.owner_user_id` belongs to another user or WaCalls reports the call already taken; on success it SHALL set `owner_user_id` to the caller only where it is still null.

#### Scenario: Losing the race to a colleague
- **WHEN** two agents accept the same ringing call and the second one reaches WaCalls after the first
- **THEN** the second receives 409 `voice_call_taken`, not a connected status

### Requirement: Only the person on the line hangs up
`DELETE /api/v1/voice/calls/{id}` SHALL require role `agent`, answer 403 `voice_call_not_yours` unless the caller is `owner_user_id` (or `created_by` when no owner), and answer 204 without a new audit entry when the call `status` is already `ended`.

#### Scenario: Colleague tries to hang up
- **WHEN** an agent who is not the owner calls `DELETE /api/v1/voice/calls/{id}` on a connected call
- **THEN** the response is 403 `voice_call_not_yours` and the call continues

### Requirement: Worker event bridge persists call state
The agent worker (workers/agent-worker/main.ts) SHALL run `runVoiceCallsBridgeLoop` (lib/wacalls/events-bridge.ts) against `${WACALLS_API_BASE_URL}/api/events`, upserting `voice_calls` on `call-status` by `(organization_id, wacalls_call_id)` and, on `call-ended`, setting `status='ended'`, `end_reason`, `ended_at` and `duration_ms` (only when `answered_at` is set).

#### Scenario: Call ends after being answered
- **WHEN** WaCalls emits `call-ended` for a call that has `answered_at`
- **THEN** the row has `status='ended'`, `ended_at` set and `duration_ms` equal to ended minus answered time, and an `event_log` row `voice_call.ended` is inserted with status `done`

### Requirement: AI silenced during a connected call
On a `call-status` of `connected` with a known contact, the bridge SHALL set `conversations.bot_silenced_until` to now + 2 hours (never shortening a longer silence) with its own `last_handoff_reason`, and on `call-ended` SHALL clear it only where `last_handoff_reason` still equals that reason.

#### Scenario: Human handoff during the call survives hang-up
- **WHEN** an attendant takes over the conversation (changing `last_handoff_reason`) during the call and the call then ends
- **THEN** `bot_silenced_until` is not cleared by the bridge

### Requirement: Missed inbound call creates an inbox alert
On `call-ended` of an inbound call without `answered_at` whose contact is not blocked, the bridge SHALL insert an `agent_inbox_items` row with `kind='voice_call_missed'`, `severity='warn'` and `ref_kind='contact'` when the contact is known.

#### Scenario: Outbound call not answered
- **WHEN** an outbound call ends without being answered
- **THEN** no `voice_call_missed` inbox item is created

#### Scenario: Blocked contact calls in
- **WHEN** a blocked contact's inbound call ends unanswered
- **THEN** the `voice_calls` row is kept but no inbox item is created

### Requirement: Call history excludes personal contacts
`GET /api/v1/voice/calls/history` SHALL require role `agent`, filter `voice_calls` by the active `organization_id`, exclude rows whose `contact_id` is a personal contact, order by `started_at` descending, and return 500 `internal_error` (never an empty list) when the query fails.

#### Scenario: Invalid id filter
- **WHEN** the `id` query parameter is not a UUID
- **THEN** the response is 400 `invalid_request`
