## Purpose
The AI flywheel is the improvement loop over real agent turns: an LLM judge grades recent turns (`flywheel_judge_verdicts`), a distiller turns a failing verdict into a proposal (`flywheel_distiller_proposals`) that only becomes behaviour after a human applies it, the runtime records near-misses as golden-set candidates (`golden_candidates`), organization-wide style adjustments are toggled in `org_guardrail_layers`, and the Evolution panel (`/app/ai/evolution`) aggregates what these produced. The agent turn itself and agent version publishing are covered by `ai-agents-runtime`; organization memory and skills by `ai-routers-skills-memory`; the purge of golden candidates by `data-retention`.

## ADDED Requirements

### Requirement: Scheduled flywheel round in the agent worker
The agent worker SHALL run `runFlywheelLoop` only when `FLYWHEEL_INTERVAL_MS` is greater than 0 (default 21600000 ms), sleeping one interval before the first round, judging at most `FLYWHEEL_BATCH_LIMIT` (default 10) recent `job_queue` rows of kind `inbound_turn` with status `done` that have an `agent_turn` `llm_calls` row, and SHALL log and swallow a failed round without stopping the worker.

#### Scenario: Flywheel disabled
- **WHEN** the worker starts with `FLYWHEEL_INTERVAL_MS=0`
- **THEN** no flywheel round runs and no `flywheel_judge` model call is made

#### Scenario: Round failure
- **WHEN** a round throws because no LLM model is configured
- **THEN** the error is logged as `flywheel: rodada agendada falhou` and the next round still runs after the interval

### Requirement: Judge verdict persistence
Each judged turn SHALL be written to `flywheel_judge_verdicts` with `dataset='live'`, `dimension='memory_hygiene'`, `trace_id` equal to the job id, `verdict` in `yes|no|unknown` (any other model output stored as `unknown`), the alternating `option_order` and the judge provider and model, using `on conflict (dataset, trace_id, dimension) do nothing` so a turn is judged once.

#### Scenario: Turn already judged
- **WHEN** a later round collects a job id that already has a `live`/`memory_hygiene` verdict
- **THEN** no new verdict row is inserted and no distiller call is made for it

### Requirement: Distiller proposals behind a human gate
Only a newly inserted verdict `no` SHALL trigger a `flywheel_distiller` model call whose result is inserted into `flywheel_distiller_proposals` as type `org_memory_entry` (target `org`) when the model returns scope `org`, otherwise `playbook_bullet` (target `tenant`), with `applied_at` null so that nothing changes agent behaviour until a person applies it.

#### Scenario: Failing verdict
- **WHEN** the judge returns `no` for a new turn
- **THEN** one pending proposal row exists with `evidence.trace_ids` containing the job id and `applied_at` null

### Requirement: Proposal listing
`GET /api/v1/ai/agents/{id}/proposals` SHALL require role `agent`, answer 400 `invalid_request` for a non-UUID id and 404 `not_found` when the agent is not in the active organization, and filter `flywheel_distiller_proposals` by `status` `pending` (`applied_at is null`), `applied` or `all` (default).

#### Scenario: Pending only
- **WHEN** a user requests `?status=pending`
- **THEN** only proposals of the organization with `applied_at` null are returned

### Requirement: Applying a proposal
`POST /api/v1/ai/agents/{id}/proposals/{pid}/apply` SHALL require role `admin`, insert an `org_memory_entries` row (`source='flywheel'`, `status='active'`) for an `org_memory_entry` proposal or create and publish a new agent version for a `playbook_bullet` proposal, set `applied_at`/`applied_by` (and `applied_version_id` for versions), audit `ai.flywheel_proposal_applied`, and answer 404 `proposal_not_found`, 409 `proposal_already_applied`, 422 `proposal_type_unsupported`, 422 `agent_not_published` or 422 `publish_failed`.

#### Scenario: Second apply
- **WHEN** an admin applies a proposal whose `applied_at` is already set
- **THEN** the response is HTTP 409 `proposal_already_applied` and no version or memory entry is created

#### Scenario: Publish vetoed
- **WHEN** publishing the new version for a `playbook_bullet` fails
- **THEN** the response is HTTP 422 `publish_failed` and the proposal stays pending

### Requirement: Golden-set candidates
The agent runtime SHALL, only when `GOLDEN_CANDIDATES_ENABLED` is `true` (default), insert label-only rows into `golden_candidates` with `fonte` `skill_match_miss` (skill and motivo) or `stage_classifier_divergence` (suggested and confirmed stage), one per job via `on conflict do nothing`, and a failed insert SHALL be logged without failing the turn.

#### Scenario: Stage disagreement
- **WHEN** the stage classifier suggests a stage different from the one the model confirmed and the knob is on
- **THEN** one `golden_candidates` row with `fonte='stage_classifier_divergence'` exists for that job and contains no customer message text

#### Scenario: Knob off
- **WHEN** `GOLDEN_CANDIDATES_ENABLED=false`
- **THEN** no `golden_candidates` row is written for any turn

### Requirement: Organization style adjustments
`GET /api/v1/ai/style-adjustments` SHALL require role `manager` and return every adjustment in `AJUSTES_DE_ESTILO` (currently `sem_travessao_longo`) with its `enabled` flag from `org_guardrail_layers` (missing row means `false`) plus `podeEditar` true only for `admin`, and `PUT` SHALL require role `admin`, accept a strict body `{ajuste, enabled}` (422 `invalid_body` otherwise), upsert on `(organization_id, layer)` and audit `ai.style_adjustment_changed`.

#### Scenario: Manager cannot change
- **WHEN** a `manager` sends `PUT` with `{ajuste:"sem_travessao_longo", enabled:true}`
- **THEN** the response is HTTP 403 `forbidden_role` and `org_guardrail_layers` is unchanged

#### Scenario: Unknown adjustment
- **WHEN** an admin sends `{ajuste:"emoji", enabled:true}`
- **THEN** the response is HTTP 422 `invalid_body`

### Requirement: Evolution panel aggregation
`GET /api/v1/ai/evolution` SHALL require role `manager`, accept optional `from`/`to` as `YYYY-MM-DD` (422 `validation_failed` otherwise), default to the last 30 UTC days and clamp the range to 90 days, read each source (`org_memory_entries`, applied `flywheel_distiller_proposals`, `skill_pointers`, `skill_activations`, `ai_router_decisions`, `knowledge_searches`, `lead_state_transitions`, `llm_calls`, counts of `messages`, `agent_inbox_items` kind `handoff` and `event_log` `ai.handoff_triggered`) filtered by the active organization and capped at 50000 rows, and return a zeroed block instead of failing when one source errors.

#### Scenario: Range too long
- **WHEN** a manager requests `from=2026-01-01&to=2026-10-01`
- **THEN** the aggregation covers only the 90 days ending on 2026-10-01

#### Scenario: One source unavailable
- **WHEN** reading `knowledge_searches` fails
- **THEN** the response is still HTTP 200 with that block zeroed and the failure logged with the request id
