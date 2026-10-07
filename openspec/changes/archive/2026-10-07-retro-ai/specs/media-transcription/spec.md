## Purpose
Media transcription turns inbound media into something the system can keep and the AI can read: `media-persist-worker` copies the channel media into the private Storage bucket `whatsapp-media` and records it on `messages`, then requests derivation; `media-derive-worker` produces a model-agnostic text (`messages.media_derived_text`: audio transcription, image vision description, PDF text, opt-in video frames) and records a terminal `media_derived_status`. Both are `event_log` consumers. Channel ingestion that emits `media.persist_requested` is covered by `whatsapp-waha`; the drain, retries and dead letters by `event-bus-workers`; signed URLs and bucket handling by `file-storage`; the expiry of media by `data-retention`; anonymization cascades by `lgpd-privacy`.

## ADDED Requirements

### Requirement: Persist consumer contract
The `media_persist_v1` consumer SHALL handle `media.persist_requested` events (also for organizations that are stopped), load the message by `message_id` filtered by the event's `organization_id`, and skip without side effects when `metadata.media_status='expired'`, when `media_url` is null or when `media_storage_path` is already set.

#### Scenario: Already stored
- **WHEN** a `media.persist_requested` event arrives for a message that already has `media_storage_path`
- **THEN** the handler returns `skipped` with detail `already stored` and uploads nothing

### Requirement: Media stored in the private bucket
The persist worker SHALL download the media through the channel adapter, upload it to bucket `whatsapp-media` at `<organization_id>/<conversation_id>/<message_id>.<ext>`, and set `messages.media_storage_path`, `media_size_bytes`, `media_mime` and `metadata.media_status='stored'`.

#### Scenario: Voice note persisted
- **WHEN** an inbound audio message is persisted successfully
- **THEN** the object exists under the organization's prefix in `whatsapp-media` and the message row has `media_storage_path` and `media_size_bytes` filled

### Requirement: Persist failures end with a cause
On the last attempt allowed by the drain (5 attempts) a failed download or upload SHALL set `metadata.media_status='failed'`, `media_derived_status='failed'` and `metadata.media_derived_motivo` with the reason, while earlier attempts SHALL return `error` so the drain retries.

#### Scenario: Provider never returns the file
- **WHEN** the download fails on the fifth attempt
- **THEN** the message has `metadata.media_status='failed'` and a non-empty `metadata.media_derived_motivo`

### Requirement: Derivation is requested only for non-group conversations
After storing, the persist worker SHALL emit `media.derive_requested` (entity `message`) only when the conversation is not `is_group`, and when that emit or the group lookup fails it SHALL record `media_derived_status='failed'` with the reason instead of leaving the column null.

#### Scenario: Group media
- **WHEN** a media message of a group conversation is persisted
- **THEN** no `media.derive_requested` event is emitted for it

### Requirement: Anonymized message is not re-filled
If the message was anonymized while the persist worker ran, the worker SHALL remove the just-uploaded object from `whatsapp-media` and return `message_redacted`, and the derive worker SHALL write `media_derived_text` only where `body` is distinct from the anonymized marker, returning `message_redacted` when zero rows matched.

#### Scenario: Anonymized during transcription
- **WHEN** the contact is anonymized between reading the message and saving its transcription
- **THEN** `media_derived_text` stays empty and the handler returns `skipped` with detail `message_redacted`

### Requirement: Derive consumer contract and terminal skip
The `media_derive_v1` consumer SHALL handle `media.derive_requested` (skipped for stopped organizations), return early when `media_derived_status` is already `ready`, only derive types in `TIPOS_DERIVAVEIS` (`audio`, `image`, `document`, `video`), and SHALL write `media_derived_status='skipped'` when the media expired by retention, has no `media_storage_path`, or is a video while no published agent version of the organization has `video_frames_enabled=true`.

#### Scenario: Video without opt-in
- **WHEN** a video message is derived for an organization with no published version having `video_frames_enabled`
- **THEN** `media_derived_status` becomes `skipped` and no frames are sent to any model

### Requirement: Successful derivation
On success the derive worker SHALL set `messages.media_derived_text` to the derived text and `media_derived_status='ready'`, filtered by the message id and organization.

#### Scenario: Transcribed audio
- **WHEN** an inbound voice note is transcribed
- **THEN** the message row has `media_derived_status='ready'` and the transcription in `media_derived_text`

### Requirement: Transcription provider ladder and environment
Audio transcription SHALL be resolved by the transcription ladder (`decidirTranscricao`), whose first step is the service configured by `TRANSCRIPTION_BASE_URL`, `TRANSCRIPTION_API_KEY`, `TRANSCRIPTION_MODEL` and `TRANSCRIPTION_LANGUAGES`, and a `TRANSCRIPTION_BASE_URL` refused by the outbound destination check SHALL fail without sending the audio or the key to it.

#### Scenario: Internal address not allowed
- **WHEN** `TRANSCRIPTION_BASE_URL` points to a destination not authorized as internal
- **THEN** the transcription call is refused with a message naming `TRANSCRIPTION_BASE_URL` and no audio leaves the server

### Requirement: Unreadable media is explicit to the agent and the team
When the media cannot be read (audio with no transcriber, an empty audio transcription, or a derivation failure on the last of 5 attempts) the derive worker SHALL set `media_derived_text` to the marker `[o cliente enviou uma mídia que não consegui interpretar]`, `media_derived_status='failed'` and `metadata.media_derived_motivo`, and open at most one `agent_inbox_items` row of kind `midia_nao_lida` (severity `warn`) per organization while one is open.

#### Scenario: Audio without any transcription key
- **WHEN** a voice note arrives and the ladder finds no transcriber
- **THEN** the message gets the unreadable marker with `media_derived_status='failed'` and the organization has one open `midia_nao_lida` notice

#### Scenario: Second unreadable media
- **WHEN** another media fails while a `midia_nao_lida` notice is still open
- **THEN** no second notice is inserted
