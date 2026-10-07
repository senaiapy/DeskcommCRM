## Purpose
Routing distributes unassigned conversations among available attendants. Key paths: the cron `app/api/v1/cron/routing-worker/route.ts` (GET/POST, scheduled every minute by `docker/scheduler/entrypoint.sh`) running `lib/routing/worker.ts`; pure decision logic in `lib/routing/decide.ts`, eligibility in `lib/routing/eligibility.ts` and `lib/routing/eligibles.ts`; per-channel responsibles in `lib/routing/channel-policies.ts`; configuration routes `app/api/v1/settings/routing/route.ts` (GET, PATCH) and `app/api/v1/settings/routing/channels/route.ts` (GET, PATCH); attendant routes `app/api/v1/attendants/availability/route.ts` (GET), `app/api/v1/attendants/availability/[user_id]/route.ts` (PATCH) and `app/api/v1/attendants/presence/route.ts` (POST). Data: `organizations.settings.routing` and `settings.visibility_mode`, `attendant_availability`, `channel_routing_policies`, `channel_routing_responsibles`, `conversation_assignment_events`, and `event_log` rows of type `conversation.routing_requested` emitted by `fn_request_channel_routing`.

## ADDED Requirements

### Requirement: Routing request emission skips groups and owned conversations
The database SHALL emit `conversation.routing_requested` into `event_log` only through `fn_request_channel_routing`, invoked by trigger `trg_conversation_routing_requested` (AFTER INSERT on `conversations` with `assigned_to_user_id IS NULL` and status `open`/`pending`) and `trg_service_reopened_routing` (terminal status back to `open`/`pending`), and SHALL skip conversations with `is_group = true`, an assignee, or a status outside `open`, `pending`, `claimed`, `ai_handling`, keeping at most one pending/processing event per conversation (unique index `event_log_routing_active_unique`).

#### Scenario: Group conversation created
- **WHEN** a conversation with `is_group = true` is inserted without an assignee
- **THEN** no `conversation.routing_requested` row is written to `event_log`

#### Scenario: Duplicate request while pending
- **WHEN** routing is requested again for a conversation that already has a `pending` routing event
- **THEN** no second event is inserted and the existing event's `next_attempt_at` is set to now

### Requirement: Routing worker cron authentication and audit
`GET`/`POST /api/v1/cron/routing-worker` SHALL require a Bearer equal to `INTERNAL_CRON_SECRET` or `INTERNAL_SECRET` (403 `forbidden` otherwise), return 200 with `batch_size`, `outcomes` and `errors`, return 500 `internal_error` if the worker throws, and write audit `routing.worker_run` only when `batch_size > 0` or errors occurred.

#### Scenario: Empty tick
- **WHEN** the cron runs with no pending routing events
- **THEN** the response is 200 with `batch_size: 0` and no `api_audit_log` row is written

#### Scenario: Missing secret
- **WHEN** the cron is called without a valid Bearer secret
- **THEN** the response is 403 `forbidden`

### Requirement: Routing modes and decision
`decideRouting` SHALL skip (`already_assigned`) when the conversation has an owner, skip (`manual_mode`) when `settings.routing.mode` is `manual` (the default), pick by `selectRoundRobin` (least recently assigned first, never-assigned first, tie broken by `userId`) in mode `round_robin`, pick the lowest `currentLoad` with the same round-robin tie-break in mode `load`, and requeue with `backoff_seconds` (at least 900 seconds once `attempts >= max_retries`) when no attendant is eligible.

#### Scenario: Round robin picks the attendant idle longest
- **WHEN** mode is `round_robin` and eligible attendants were last assigned at 10:00 and 09:00
- **THEN** the attendant last assigned at 09:00 receives the conversation

#### Scenario: Load mode
- **WHEN** mode is `load` and eligible attendants have current loads 3 and 1
- **THEN** the attendant with load 1 receives the conversation

#### Scenario: No eligible attendant
- **WHEN** mode is `round_robin` and nobody is eligible
- **THEN** the event returns to `status = 'pending'` with `next_attempt_at` = now + `backoff_seconds` and outcome `requeued_no_eligible`

### Requirement: Attendant eligibility
An attendant SHALL be eligible only when they are an active (`revoked_at IS NULL`) `agent`, `manager` or `admin` member, have `attendant_availability.is_available = true`, have fewer conversations in status `open`, `pending`, `claimed` or `ai_handling` than `capacity`, are inside a `schedule.windows` window in `schedule.timezone` (empty windows meaning 24/7), and, when the conversation's channel has a `channel_routing_policies` row, are listed in `channel_routing_responsibles`.

#### Scenario: Attendant at capacity
- **WHEN** an available attendant with `capacity = 5` already owns 5 open conversations
- **THEN** they are not a routing candidate

#### Scenario: Restricted channel with no responsibles
- **WHEN** the conversation's channel has a routing policy with zero responsibles
- **THEN** no attendant is eligible and the event is requeued

### Requirement: Atomic routing claim
The worker SHALL assign through RPC `fn_channel_routing_claim` with `p_reason = 'routing'`, which re-checks membership, channel responsibility, availability, schedule snapshot and capacity under lock, mark the event `done` with outcome `assigned` on success, `assign_lost_race` when the RPC returns `already_assigned` or `conversation_changed`, and requeue on `candidate_revoked`, `candidate_not_allowed` or `capacity_changed`.

#### Scenario: Another attendant claimed first
- **WHEN** the conversation gained an owner between selection and the RPC
- **THEN** the RPC returns `already_assigned`, no reassignment happens and the event is marked `done` with outcome `assign_lost_race`

#### Scenario: Successful assignment adopts open leads
- **WHEN** the RPC returns `assigned`
- **THEN** open `crm_leads` of the contact without `owner_user_id` and `owner_agent_id` get `owner_user_id` set to the attendant

### Requirement: Personal contacts and stale events are not routed
The worker SHALL mark the event `done` without assignment with outcome `skipped_contato_pessoal` when the conversation's contact has `is_personal = true`, `skipped_conv_missing` when the conversation is missing or not in an open status, and SHALL reset events stuck in `processing` for more than 300 seconds back to `pending`.

#### Scenario: Personal contact
- **WHEN** a routing event refers to a conversation whose contact has `is_personal = true`
- **THEN** the event is marked `done` with `metadata.outcome = 'skipped_contato_pessoal'` and the conversation stays unassigned

### Requirement: Routing configuration
`GET`/`PATCH /api/v1/settings/routing` SHALL require role `manager`, read and merge `organizations.settings.routing` (`mode` in `manual`/`round_robin`/`load`, `max_retries` 0–20, `backoff_seconds` 1–3600, `handoff_return_after_minutes`, `manual_reply_silence_minutes`, `conversation_stays_with_attendant`) and `settings.visibility_mode` (`all`, `own_and_unassigned`, `own`) without dropping other `settings` keys, and PATCH SHALL write audit `routing.config_changed`.

#### Scenario: Switch to round robin
- **WHEN** a manager sends `PATCH /api/v1/settings/routing` with `{ "mode": "round_robin" }`
- **THEN** `organizations.settings.routing.mode` becomes `round_robin`, other settings keys are preserved and an audit row `routing.config_changed` is written

#### Scenario: Agent reads routing settings
- **WHEN** an `agent` calls `GET /api/v1/settings/routing`
- **THEN** the `requireRole("manager")` gate rejects the request

### Requirement: Per-channel responsibles
`PATCH /api/v1/settings/routing/channels` SHALL require `requireSupportWrite`, role `manager`, and no pending MFA (403 `mfa_required`), validate `{ channel_session_id, user_ids, reset }` strictly (422 `validation_failed`), and call RPC `fn_set_channel_routing`, mapping `P0002` to 404 `not_found`, `22023` to 422 `validation_failed` and `42501` to 403 `forbidden`.

#### Scenario: Unknown channel
- **WHEN** a manager sets responsibles for a `channel_session_id` that does not exist in the organization
- **THEN** the response is 404 `not_found`

### Requirement: Attendant availability
`PATCH /api/v1/attendants/availability/{user_id}` SHALL require role `agent`, allow only the attendant themself or a `manager`+ (403 `forbidden_role` otherwise), return 404 `not_found` when a manager targets a non-member, validate at least one of `is_available`, `capacity` (1–1000) or `schedule`, upsert `attendant_availability` on `(organization_id, user_id)` and write audit `attendant.availability_changed`.

#### Scenario: Agent edits a colleague
- **WHEN** an agent patches another user's availability
- **THEN** the response is 403 `forbidden_role`

#### Scenario: Attendant goes on duty
- **WHEN** an agent patches their own row with `{ "is_available": true }`
- **THEN** `attendant_availability.is_available` is true and an audit row `attendant.availability_changed` is written

### Requirement: Presence heartbeat does not change availability
`POST /api/v1/attendants/presence` SHALL require `requireSupportWrite` and role `agent`, upsert only `last_heartbeat_at` for the caller's `attendant_availability` row, and write audit `attendant.presence_started` only on the first beat that creates the row; presence SHALL never set `is_available`.

#### Scenario: Repeated heartbeat
- **WHEN** an attendant who already has an availability row posts a heartbeat
- **THEN** only `last_heartbeat_at` changes and no audit row is written
