## Purpose
The inbox is where attendants list, take ownership of, annotate and answer WhatsApp conversations. Entry points are the page `app/app/inbox/page.tsx` (with `app/app/inbox/[id]/page.tsx` redirecting to `/app/inbox?id=<id>`), the API routes under `app/api/v1/conversations/*` (list, counts, get/patch, claim, release, transfer, close, snooze, notes, drafts, media upload, messages) and `app/api/v1/messages/*` (send, edit/revoke, hide, media signed URL), the shared handlers `app/api/v1/conversations/_handler.ts` and `app/api/v1/messages/_handler.ts`, helpers in `lib/inbox/` and `lib/messaging/`, the realtime hook `hooks/inbox/useConversationsRealtime.ts`, and the cron `app/api/v1/cron/snooze-watcher`. Data lives in `conversations`, `messages`, `conversation_notes` and `conversation_drafts`; media lives in the private Storage bucket `whatsapp-media`. Conversations of contacts marked `contacts.is_personal` leave the operation (docs/specs/21).

## ADDED Requirements

### Requirement: Conversation and message data model
The `conversations` table SHALL constrain `status` to `open`, `pending`, `resolved`, `claimed`, `ai_handling`, `closed` or `archived`, carry `assigned_to_user_id`, `is_group`, `group_chat_id` and `unread_count_for_assignee`, and the `messages` table SHALL constrain `direction` to `inbound`/`outbound` and `status` to `queued`, `received`, `sending`, `sent`, `delivered`, `read` or `failed`.

#### Scenario: Invalid conversation status rejected by the database
- **WHEN** a row in `conversations` is written with `status = 'snoozed'`
- **THEN** the insert/update fails on constraint `conversations_status_check`

#### Scenario: Both tables are published to realtime
- **WHEN** the baseline is applied
- **THEN** `messages` and `conversations` are members of publication `supabase_realtime`

### Requirement: List conversations
`GET /api/v1/conversations` SHALL require an authenticated user with an active organization (401 `unauthenticated` / 403 `no_active_org` otherwise), filter by `organization_id` of the active org, accept the query filters `status`, `exclude_finished`, `assigned_to`, `comando`, `tag`, `modo`, `unread`, `channel_session_id`, `is_group`, `search`, `contact_id`, `cursor` and `limit`, and return 422 `validation_failed` for an invalid query.

#### Scenario: Filter group conversations
- **WHEN** an agent calls `GET /api/v1/conversations?is_group=true`
- **THEN** the response is 200 with only rows where `is_group = true` and `meta.cursor` / `meta.has_more` describe the filtered set

#### Scenario: Invalid query
- **WHEN** the query string carries a value the `listConversationsQuerySchema` rejects
- **THEN** the response is 422 with code `validation_failed` and `details` per field

### Requirement: Personal contacts leave the operation
The conversation list (`listConversationsHandler`) and `GET /api/v1/conversations/counts` SHALL exclude conversations whose contact has `contacts.is_personal = true`, and `sendMessageHandler` SHALL refuse any send to such a contact with 403 `forbidden`.

#### Scenario: Personal conversation hidden from the list
- **WHEN** the contact of a conversation has `is_personal = true` and an agent lists conversations
- **THEN** that conversation is absent from `data` and from the counts

#### Scenario: Send to personal contact refused
- **WHEN** `POST /api/v1/messages` targets a conversation whose contact has `is_personal = true`
- **THEN** the response is 403 `forbidden` and no `messages` row is inserted

#### Scenario: Message history of a personal conversation
- **WHEN** `GET /api/v1/conversations/{id}/messages` is called for a personal contact's conversation
- **THEN** the response is 404 `not_found`

### Requirement: Claim with optimistic lock
`POST /api/v1/conversations/{id}/claim` SHALL require role `agent` and `requireSupportWrite`, assign the conversation to the caller through RPC `fn_conversation_assign` with `p_reason = 'claim'` and `p_enforce_expected = true`, and return 409 `state_conflict` when the RPC returns no row (another attendant already owns it).

#### Scenario: Claim a free conversation
- **WHEN** an agent posts `{}` to claim an unassigned conversation
- **THEN** the conversation's `assigned_to_user_id` becomes the caller, an audit row `conversation.claimed` is written and event `conversation.claimed` is emitted

#### Scenario: Claim lost to another attendant
- **WHEN** an agent posts `{ "expected_assignee": null }` but the conversation already has an owner
- **THEN** the response is 409 `state_conflict`

### Requirement: Transfer and release
`POST /api/v1/conversations/{id}/transfer` SHALL require role `agent`, accept `{ to_user_id, reason? }`, reject a target that is not an active non-viewer member of the same organization with 422 `unprocessable_entity`, and reassign through `fn_conversation_assign` with `p_reason = 'transfer'` and no optimistic lock; `POST /api/v1/conversations/{id}/release` SHALL return 409 `state_conflict` when the caller is not the assignee.

#### Scenario: Transfer to a viewer
- **WHEN** an agent transfers a conversation to a user whose `user_organizations.role` is `viewer`
- **THEN** the response is 422 `unprocessable_entity` and the assignee is unchanged

#### Scenario: Successful transfer
- **WHEN** an agent transfers a conversation to an active agent of the same org
- **THEN** the response is 200 with the updated conversation and an audit row `conversation.transferred` is written

### Requirement: Close and reopen through the service status RPC
`POST /api/v1/conversations/{id}/close` and `PATCH /api/v1/conversations/{id}` with `status` SHALL change status only through RPC `fn_service_status` with `p_expected` set to `expected_revision` (or the current `service_revision`), returning 409 `conflict` when the RPC raises `PT409` or `40001`, and 404 `not_found` when the conversation is not visible in the active org.

#### Scenario: Stale revision on close
- **WHEN** an agent posts `{ "expected_revision": 3 }` to close a conversation whose `service_revision` already advanced
- **THEN** the response is 409 with code `conflict`

#### Scenario: Reopen via PATCH
- **WHEN** an agent sends `PATCH /api/v1/conversations/{id}` with `{ "status": "open" }`
- **THEN** `fn_service_status` sets status `open` and an audit row `conversation.released` is written

#### Scenario: PATCH with empty body
- **WHEN** the PATCH body contains neither `status` nor `tags`
- **THEN** the request is rejected by `patchConversationSchema` with a validation error

### Requirement: Snooze
`POST /api/v1/conversations/{id}/snooze` SHALL require role `agent`, accept only `duration_hours` of 1, 3 or 24 (else 422 `validation_failed`), and set `conversations.snooze_until`, `snoozed_at` and `snoozed_by_user_id`; `DELETE` on the same route SHALL clear the three columns and return 204; the cron `GET/POST /api/v1/cron/snooze-watcher` (every 5 minutes, Bearer `INTERNAL_CRON_SECRET`/`INTERNAL_SECRET`, 403 `forbidden` otherwise) SHALL clear expired snoozes.

#### Scenario: Unsupported duration
- **WHEN** an agent posts `{ "duration_hours": 2 }`
- **THEN** the response is 422 `validation_failed`

#### Scenario: Snooze set
- **WHEN** an agent posts `{ "duration_hours": 3 }` for a conversation of the active org
- **THEN** `snooze_until` is now + 3h and an audit row `conversation.snoozed` is written

### Requirement: Internal notes
`GET`/`POST /api/v1/conversations/{id}/notes` SHALL require role `agent`, return 404 `not_found` when the conversation is not in the active org, insert into `conversation_notes` with `created_by_user_id` and `created_by_name`, and reject an `anexo.storage_path` outside `{org}/{conversation}/` with 422 `validation_failed`; `DELETE /api/v1/conversations/{id}/notes/{noteId}` SHALL allow only the author or `manager`+ (403 `forbidden` otherwise) and return 204.

#### Scenario: Note created
- **WHEN** an agent posts a valid note body
- **THEN** the response is 201 with the note and an audit row `conversation.note_added` is written

#### Scenario: Non-author agent deletes a note
- **WHEN** an agent who did not write the note and is below manager deletes it
- **THEN** the response is 403 `forbidden`

### Requirement: Server-side reply drafts
`POST /api/v1/conversations/{id}/drafts` SHALL accept session or Bearer `dsk_` auth (role `agent`, scope `mcp:write`), validate `{ texto, origem, expira_em_horas }`, require a UUID `Idempotency-Key` when present (400 `validation_error` otherwise), and insert into `conversation_drafts` returning 201; draft bodies are constrained to 1–4096 characters.

#### Scenario: Malformed idempotency key
- **WHEN** the request carries `Idempotency-Key: abc`
- **THEN** the response is 400 `validation_error`

#### Scenario: Draft for a conversation outside the org
- **WHEN** the conversation id does not exist in the caller's organization
- **THEN** the response is 404 `not_found`

### Requirement: Outbound media upload and signed retrieval
`POST /api/v1/conversations/{id}/media` SHALL accept a multipart `file`, reject bodies over 50MB with 413 `payload_too_large` and unsupported types with 415, and upload to bucket `whatsapp-media` at `{org}/{conversation}/out-{uuid}.{ext}`; `GET /api/v1/messages/{id}/media` SHALL redirect (302) to a signed URL valid for 3600 seconds when `media_storage_path` exists, and return 410 `media_expired` when retention deleted the media.

#### Scenario: Missing file field
- **WHEN** the multipart body has no `file`
- **THEN** the response is 422 `validation_failed`

#### Scenario: Stored media fetched
- **WHEN** a member requests media of a message with `media_storage_path`
- **THEN** the response is 302 to a signed URL and carries `X-Request-Id`

### Requirement: Send message
`POST /api/v1/messages` SHALL require `requireSupportWrite` and session or Bearer auth with role `agent` and scope `mcp:write`, insert the outbound row in `messages` with `status = 'queued'` and `sent_via` derived from the actor, then mark it `sent` (or `failed` with `error_code`) after the channel adapter call, and return 201.

#### Scenario: Blocked contact
- **WHEN** the conversation's contact has `is_blocked = true`
- **THEN** the response is 403 `forbidden` and no message is sent

#### Scenario: Auto-claim on reply
- **WHEN** a session user sends a message and `organizations.settings.routing.conversation_stays_with_attendant` is true
- **THEN** the conversation is assigned to that user via `fn_conversation_assign` and an audit row `conversation.claimed` is written

#### Scenario: on_behalf_of without scope
- **WHEN** the body has `on_behalf_of_user_id` and the caller is not a token with scope `messages:on_behalf`
- **THEN** the response is 403 `forbidden`

### Requirement: Realtime inbox refresh
The inbox list hook SHALL subscribe to Supabase Realtime `postgres_changes` on table `conversations` filtered by `organization_id=eq.<active org>` on channel `inbox-<orgId>` and refetch the list on any change, with a safety refetch when the channel stops delivering.

#### Scenario: New inbound message changes a conversation
- **WHEN** a `conversations` row of the active org is updated
- **THEN** the realtime channel `inbox-<orgId>` receives the change and the conversation list query is refetched
