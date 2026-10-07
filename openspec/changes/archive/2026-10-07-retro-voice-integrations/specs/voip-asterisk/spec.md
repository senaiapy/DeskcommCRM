## Purpose
SIP telephony through a self-hosted Asterisk (compose profile `telefonia`, services `asterisk` and `voice-agent`): inbound
DIDs registered in `phone_numbers` are routed to an organization and answered by an OpenAI Realtime voice agent over
AudioSocket, and managers originate outbound calls through ARI. Calls are stored in the shared `voice_calls` table with
`provider='sip'`. WhatsApp calls through WaCalls, which share that table, are covered by `voice-whatsapp-calls`; voice
agent prompt/configuration in `ai_agents` is covered by the AI agent capabilities.

## ADDED Requirements

### Requirement: SIP call listing
`GET /api/v1/calls` SHALL require role `manager` and return only `voice_calls` rows with `provider='sip'` of the active `organization_id`, excluding personal contacts, ordered by `started_at` descending, with `status` translated by `mapStatusParaApi` (`connected`→`in_progress`, `ended`+`timeout`→`no_answer`, `ended`+`contact_blocked`→`canceled`).

#### Scenario: WhatsApp calls are not listed
- **WHEN** the organization has both a `provider='wacalls'` and a `provider='sip'` call
- **THEN** `GET /api/v1/calls` returns only the SIP call

#### Scenario: Unanswered call
- **WHEN** a SIP call row has `status='ended'` and `end_reason='timeout'`
- **THEN** it is returned with `status` `no_answer`

### Requirement: Outbound origination through ARI
`POST /api/v1/calls` SHALL require role `manager`, insert a `voice_calls` row (`provider='sip'`, `direction='outbound'`, `status='ringing'`, `asterisk_channel_id` = a new UUID, `handled_by` `human` or `ai` from `mode`) and originate via ARI `POST /ari/channels` into context `voice-agent-out` with variable `AUDIOSOCKET_UUID`, returning 201 with `callId` and `channelId`.

#### Scenario: ARI rejects the originate
- **WHEN** the ARI request fails
- **THEN** the row is updated to `status='ended'`, `end_reason='failed'` and the response is 502 `originate_failed`

#### Scenario: Personal contact
- **WHEN** the body's `contactId` (or the `contact_id` of `leadId`) is a contact with `is_personal` true
- **THEN** the response is 403 `forbidden` and nothing is originated

### Requirement: Trunk resolution
`POST /api/v1/calls` SHALL use `voip_trunk_settings.endpoint_name` of the active organization when `is_active` is true, fall back to the `VOIP_TRUNK_ENDPOINT` environment variable otherwise, and answer 422 `trunk_not_configured` when neither exists.

#### Scenario: No trunk at all
- **WHEN** the organization has no active `voip_trunk_settings` row and `VOIP_TRUNK_ENDPOINT` is unset
- **THEN** the response is 422 `trunk_not_configured`

### Requirement: Trunk settings with encrypted password
`PUT /api/v1/voip/trunk` SHALL require role `admin` and upsert `voip_trunk_settings` by `organization_id` with `endpoint_name` `org-<organization_id>-trunk-endpoint`, storing the password AES-GCM encrypted (`password_encrypted`, `password_iv`, `password_tag`, `password_last4`), while `GET /api/v1/voip/trunk` (role `manager`) SHALL read only the `voip_trunk_settings_safe` view without secret columns.

#### Scenario: First save without password
- **WHEN** an admin saves a trunk for an organization that has none and omits `password`
- **THEN** the response is 422 `password_required`

#### Scenario: Invalid port
- **WHEN** the body has `port` 70000
- **THEN** the response is 422 `validation_failed`

### Requirement: Phone number registry
`POST /api/v1/phone-numbers` SHALL require role `manager`, take `trunk_endpoint` from `VOIP_TRUNK_ENDPOINT` (500 `voip_nao_configurado` when unset), and insert into `phone_numbers` with the active `organization_id`, answering 409 `numero_ja_cadastrado` because `phone_numbers.number` is globally unique; `GET` and `PATCH /api/v1/phone-numbers/{id}` SHALL also require `manager`.

#### Scenario: DID owned by another organization
- **WHEN** a manager registers a `number` already present in `phone_numbers`
- **THEN** the response is 409 `numero_ja_cadastrado`

### Requirement: Inbound tenant resolution by dialed number
On ARI `StasisStart` the voice-agent worker (workers/voice-agent/index.ts) SHALL resolve the organization by the dialed extension through `fn_resolve_inbound_number`, and for extension `s` SHALL accept only when exactly one `phone_numbers` row has `is_active=true`, hanging up the channel otherwise.

#### Scenario: Unmapped number
- **WHEN** a call arrives for a number absent from `phone_numbers`
- **THEN** the worker hangs up the channel and inserts no `voice_calls` row

### Requirement: Blocked and personal callers are refused
After resolving the caller contact, the worker SHALL insert an already-ended `voice_calls` row with `end_reason='contact_blocked'` (contact `is_blocked`) or `end_reason='contato_pessoal'` (contact `is_personal`) and hang up, without creating a lead or opening a Realtime session.

#### Scenario: Blocked caller
- **WHEN** a contact with `is_blocked` true calls a registered DID
- **THEN** a row with `status='ended'`, `end_reason='contact_blocked'` and `answered_at` null is stored and the channel is hung up

### Requirement: Inbound call record and lead
For an accepted inbound call the worker SHALL insert `voice_calls` (`provider='sip'`, `direction='inbound'`, `status='ringing'`, a fresh `asterisk_channel_id`), call `garantirLeadDaConversa` with source `voip` when the caller contact is known, set channel variable `AUDIOSOCKET_UUID` and continue the dialplan at `from-trunk-audiosocket`.

#### Scenario: First call from a new number
- **WHEN** an unknown number calls an active DID
- **THEN** a contact is resolved or created, a ringing `voice_calls` row exists, and lead creation is attempted with source `voip`

### Requirement: AudioSocket bridge to OpenAI Realtime
The worker SHALL listen for AudioSocket TCP connections on `AUDIOSOCKET_PORT` (default 9092), match the UUID frame to `voice_calls.asterisk_channel_id`, and end the socket when no row matches, when `organizations.status` is not operational, or when no `ai_agents` row with `channel='voice'` and `is_active=true` exists; otherwise it SHALL open `wss://api.openai.com/v1/realtime` with model `OPENAI_REALTIME_MODEL` (fallback `gpt-realtime`) and set the row to `status='connected'`, `handled_by='ai'`.

#### Scenario: Organization suspended
- **WHEN** an AudioSocket connection arrives for a call whose organization is suspended
- **THEN** the socket is closed and no Realtime session is opened

### Requirement: Call finalization
When the bridge ends the worker SHALL update the row to `status='ended'`, `end_reason='user_ended'`, `ended_at`, `duration_ms` (from `answered_at`) and `transcript`, and on ARI `ChannelDestroyed` SHALL mark the row whose `asterisk_channel_id` equals the destroyed channel id and is still `ringing` as `status='ended'`, `end_reason='timeout'`.

#### Scenario: Outbound call never answered
- **WHEN** the ARI channel of a still-ringing outbound call is destroyed
- **THEN** the row becomes `status='ended'` with `end_reason='timeout'`

### Requirement: Worker startup and network exposure
The voice-agent worker SHALL refuse to start unless `ARI_URL`, `ARI_WS_URL`, `ARI_USERNAME`, `ARI_PASSWORD` and `OPENAI_API_KEY` are set, and the `asterisk` service in docker-compose.prod.yml SHALL publish only UDP 5060 and RTP 10000-10200 (matching asterisk/rtp.conf), keeping ARI port 8088 internal.

#### Scenario: Missing ARI credentials
- **WHEN** the voice-agent container starts without `ARI_PASSWORD`
- **THEN** the worker exits reporting the missing variables
