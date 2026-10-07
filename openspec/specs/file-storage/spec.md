# file-storage Specification

## Purpose
Binary storage of conversation media in the private Supabase Storage bucket `whatsapp-media`: how inbound media is persisted, how operators upload outbound media, how the browser reads media through short-lived signed URLs, and how objects are deleted through `storage_redaction_queue` drained by the `storage-redaction` cron. Which messages become due for deletion and which personal data is redacted are covered by `data-retention` and `lgpd-privacy`; message ingestion and sending by `whatsapp-waha` and `inbox-conversations`.

## Requirements

### Requirement: Private media bucket with a 50 MB object limit
The schema SHALL create the Storage bucket `whatsapp-media` with `public = false` and `file_size_limit = 52428800` (migration `0055_whatsapp_media_bucket`), with no `storage.objects` policy for `anon` or `authenticated`, and `MAX_MEDIA_BYTES` in `lib/messaging/media/types.ts` SHALL equal that limit.

#### Scenario: Browser tries to read the bucket directly
- **WHEN** an authenticated browser session requests an object of `whatsapp-media` without a signed URL
- **THEN** Storage refuses it because the bucket is private and has no policy for that role

### Requirement: Object paths are namespaced by organization and conversation
Inbound media persisted by `workers/media-persist-worker.ts` SHALL be stored at `<organization_id>/<conversation_id>/<message_id>.<ext>` (via `storagePathFor`, upsert true) and recorded in `messages.media_storage_path`, and outbound uploads SHALL be stored at `<organization_id>/<conversation_id>/out-<uuid>.<ext>` with upsert false.

#### Scenario: Message already persisted
- **WHEN** the persist worker receives a message whose `media_storage_path` is already set
- **THEN** it skips the message without uploading again

### Requirement: Media is served through a one-hour signed URL redirect
`GET /api/v1/messages/[id]/media` SHALL return 401 `unauthenticated` without a session, SHALL load the message filtered by the active `organization_id`, and when `media_storage_path` is set SHALL answer 302 to a `whatsapp-media` signed URL valid for 3600 seconds with an `X-Request-Id` header.

#### Scenario: Persisted image opened in the inbox
- **WHEN** an operator of the owning organization requests the media of a message with `media_storage_path`
- **THEN** the response is a 302 redirect to a signed URL that expires after one hour

#### Scenario: Message of another organization
- **WHEN** the message id belongs to a different organization
- **THEN** the response is 404 `not_found`

### Requirement: Media removed by retention answers 410
When a message has neither `media_storage_path` nor `media_url`, the media route SHALL return 410 `media_expired` if `messages.metadata.media_status = 'expired'` (naming `media_retention_days` when present) and 404 `not_found` otherwise, without fetching the file again from the channel.

#### Scenario: Media pruned by policy
- **WHEN** the message metadata has `media_status: "expired"` and `media_retention_days: 365`
- **THEN** the response is 410 `media_expired` with a message citing 365 days

### Requirement: Not-yet-persisted media is proxied through the channel adapter
When only `media_url` is set, the media route SHALL fetch the file through the channel adapter's `fetchInboundMedia` for the message's `channel_session_id` (filtered by `organization_id`), returning 200 with `Cache-Control: private, max-age=60`, 404 `not_found` when the adapter has no inbound media capability and 502 `bad_gateway` when the fetch fails.

#### Scenario: Transport unavailable
- **WHEN** the adapter fetch throws
- **THEN** the response is 502 `bad_gateway`

### Requirement: Outbound media upload requires agent role and enforces size and type
`POST /api/v1/conversations/[id]/media` SHALL call `requireSupportWrite`, SHALL authorize via `resolveAuthDual` with role `agent` (session) or scope `mcp:write` (token) plus the token write ceiling, SHALL return 404 `not_found` for a conversation outside the caller's organization, 413 `payload_too_large` when `Content-Length` exceeds 50 MB plus 1 MB or the file exceeds the limit, 422 `validation_failed` when the multipart `file` is missing and 415 `unsupported_media_type` for a refused MIME type.

#### Scenario: Viewer tries to upload
- **WHEN** a user with role `viewer` posts a file
- **THEN** the request is refused by the role gate and nothing is written to `whatsapp-media`

#### Scenario: Oversized declared body
- **WHEN** the request declares a `Content-Length` of 60 MB
- **THEN** the response is 413 `payload_too_large` before the body is parsed

### Requirement: Storage deletions go through a queue with bounded retries
`storage_redaction_queue` SHALL only admit `status` values `pending`, `deleted`, `failed` and `skipped`, and `drainStorageRedactionQueue` (re-exported by `workers/storage-cleanup-worker.ts`) SHALL remove each `pending` row's `object_path` from its `bucket`, marking it `deleted` on success, `skipped` with `error_message = 'object_not_found'` when the object is already gone, and `failed` after `MAX_ATTEMPTS` = 3 failed attempts.

#### Scenario: Object already deleted
- **WHEN** Storage answers "not found" for a queued object
- **THEN** the queue row becomes `skipped` with `error_message = 'object_not_found'`

#### Scenario: Third consecutive failure
- **WHEN** removal fails for a row whose `attempts` is already 2
- **THEN** the row becomes `failed` and is no longer retried

### Requirement: Storage redaction cron drains in bounded batches
`GET /api/v1/cron/storage-redaction` SHALL return 403 `forbidden` unless `autorizaCron` accepts the `INTERNAL_CRON_SECRET`/`INTERNAL_SECRET` bearer, SHALL drain at most `limit` rows (default 50, capped at 200) and return the drain stats, and the scheduler SHALL call it every 5 minutes with `limit=50`.

#### Scenario: Cron called without the secret
- **WHEN** the route is called without a valid bearer
- **THEN** the response is 403 `forbidden` and no row is processed

### Requirement: Media retention feeds the redaction queue
`GET /api/v1/cron/media-retention` SHALL require `autorizaCron`, SHALL call `fn_enfileirar_midia_vencida` in batches of 500 for up to 10 batches per run, enqueueing into `storage_redaction_queue` the `whatsapp-media` objects of messages older than `organizations.media_retention_days` (default 365, floor 30) only for organizations with `media_retention_enforced` true, plus unreferenced orphan objects, and purging `deleted` queue rows older than 90 days; the scheduler SHALL run it daily at 05:20.

#### Scenario: Organization sets retention below the floor
- **WHEN** `organizations.media_retention_enforced` is true and `media_retention_days` is 10
- **THEN** only media older than 30 days is enqueued

#### Scenario: Enforcement switched off
- **WHEN** `organizations.media_retention_enforced` is false
- **THEN** none of that organization's message media is enqueued for age
