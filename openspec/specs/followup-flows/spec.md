# followup-flows Specification

## Purpose
Visual follow-up flows: a graph edited as a draft on `followup_flow_pointers`, published into immutable `followup_flow_versions`, executed per contact as `followup_enrollments` by the clock cron `followup-flow-worker`, plus one-off return promises ("retornos", `cron_jobs` with `job_kind = 'followup_turn'`). AI turns that a flow requests are run by the agent-worker described in `ai-agents-runtime`; the semantic promise guardrail on outbound text is also described there.

## Requirements

### Requirement: Flow graphs are validated against a closed node vocabulary
`flowGraphSchema`, applied to `draft_graph` by `PATCH /api/v1/ai/followup-flows/{id}`, SHALL accept only node types `trigger`, `wait`, `condition`, `ai_classify`, `match_reply`, `repeat`, `collect`, `skill`, `action`, `internal_task`, `move_lead`, `edit_lead_tag` and `end`, and edge conditions `always`, `class_match`, `cond_result` and `branch`.

#### Scenario: Unknown node type
- **WHEN** a manager saves a `draft_graph` containing a node of a type outside the list
- **THEN** the response is 422 `validation_failed` and the draft is not stored

### Requirement: Flow management requires manager and unique names
`POST /api/v1/ai/followup-flows`, `PATCH /api/v1/ai/followup-flows/{id}`, `/duplicate`, `/publish`, `/disable` and `/rollback` SHALL require role `manager`, listing and detail SHALL require role `viewer`, and a duplicate `(organization_id, name)` SHALL answer 409 `conflict`.

#### Scenario: Name clash
- **WHEN** a manager renames a flow to the name of another flow in the organization
- **THEN** the response is 409 `conflict`

### Requirement: Publishing is atomic through a service-role RPC
`POST /api/v1/ai/followup-flows/{id}/publish` SHALL validate `draft_graph` with `validateFlowForPublish`, answer 422 `validation_failed` with `details.errors` when it is missing or invalid, and publish via `fn_publish_followup_flow_version`, which inserts the version and sets `active_version_id` and `status = 'active'` in one call.

#### Scenario: Valid draft
- **WHEN** a manager publishes a valid draft
- **THEN** a new `followup_flow_versions` row exists and the pointer is `active` with that version

### Requirement: Disable and rollback only move the pointer
`POST /api/v1/ai/followup-flows/{id}/disable` SHALL set `status = 'disabled'` idempotently, and `POST /api/v1/ai/followup-flows/{id}/rollback` SHALL set `active_version_id` to an existing version of the same organization (404 `not_found` otherwise) without creating a version.

#### Scenario: Disable twice
- **WHEN** a manager disables an already disabled flow
- **THEN** the response is 200 with `status: 'disabled'` and no write happens

### Requirement: An enrollment needs an active flow and is unique per pointer and contact
`POST /api/v1/ai/followups/enrollments` SHALL require role `manager`, answer 422 `flow_not_enrollable` for a pointer with `surface = 'atendimento'`, 422 `flow_not_active` for a pointer that is not `active` or lacks `active_version_id`, and 409 `conflict` when the unique index `idx_followup_enrollments_one_live` already holds a live enrollment for that `(pointer_id, contact_id)`.

#### Scenario: Enroll into a draft flow
- **WHEN** a manager enrolls a contact into a flow with `status = 'draft'`
- **THEN** the response is 422 `flow_not_active`

### Requirement: Enrollment states keep clock coherence
`followup_enrollments.status` SHALL be one of `active`, `waiting_reply`, `paused_handoff`, `completed`, `cancelled`, `dead`, with `next_eval_at` required for `active` and `waiting_reply`; manual cancel SHALL set `status = 'cancelled'`, `cancel_reason = 'manual'` and `next_eval_at = null`, answering 409 `already_terminal` for a terminal enrollment.

#### Scenario: Cancel a completed enrollment
- **WHEN** a manager cancels an enrollment already `completed`
- **THEN** the response is 409 `already_terminal`

### Requirement: The clock cron advances due enrollments and runs the silence sweep
`GET|POST /api/v1/cron/followup-flow-worker` SHALL authorize with `INTERNAL_CRON_SECRET` or `INTERNAL_SECRET` (403 `forbidden` otherwise), run `fn_appointment_confirmation_sweep`, then `runFollowupTick`, then `runSilenceSweep` in an isolated try/catch, then send pending fixed texts, auditing `followup.worker_run` only when the tick had an effect.

#### Scenario: Silence sweep throws
- **WHEN** `runSilenceSweep` raises an error
- **THEN** the cron still returns the tick summary and only logs the sweep error

### Requirement: An enrollment that exhausts attempts becomes dead and is announced
When a step reaches a dead result the engine SHALL first insert an `agent_inbox_items` warning naming the flow and enrollment and then set `status = 'dead'` with `claimed_until = null`.

#### Scenario: Repeated step failure
- **WHEN** a step fails up to `max_attempts`
- **THEN** the enrollment is `dead` and the Central shows a warning for it

### Requirement: Armed automatic flows without an agent raise and close a warning
`GET /api/v1/cron/followup-sem-agente` SHALL open one `agent_inbox_items` row of kind `followup_sem_agente` (severity `warn`) per active automatic flow whose published graph needs AI but no published agent lists the pointer in `followup.flow_pointer_ids`, and SHALL resolve that row when the flow gains an agent.

#### Scenario: Agent attached later
- **WHEN** an agent is published with the flow pointer after the warning opened
- **THEN** the next cron run sets the warning `status = 'resolved'`

### Requirement: Return promises are cancellable by staff
A return promise SHALL be a `cron_jobs` row with `kind = 'at'` and `job_kind = 'followup_turn'`, and `POST /api/v1/ai/followups/promises/{id}/cancel` SHALL require role `manager`, take the organization from the session, answer 404 `not_found` or 409 `already_terminal`, and write a timeline activity so the agent does not reschedule it.

#### Scenario: Cancel a fired promise
- **WHEN** a manager cancels a promise that already fired
- **THEN** the response is 409 `already_terminal`
