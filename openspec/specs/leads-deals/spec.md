# leads-deals Specification

## Purpose
Leads ("negócios", deals) are the cards of the sales board: rows of `crm_leads` (`pipeline_id`, `stage_id`, `status` open/won/lost, `lost_reason`, `position_in_stage` numeric, `value_cents` + `currency`, `owner_user_id`, `custom_fields`, `tags`, `closed_at`), with polymorphic links in `crm_lead_links` and an AI-maintained score in `crm_lead_scores`. Entry points: `POST /api/v1/leads`, `PATCH /api/v1/leads/[id]`, `POST /api/v1/leads/[id]/move`, `/win`, `/lose`, `/clone`, `/retomar`, `/reactivation`, `/next-action`, `GET /api/v1/leads/[id]/timeline`, `GET /api/v1/leads/[id]/contatos-relacionados`, `POST /api/v1/leads/bulk`, `GET /api/v1/leads/at-risk`, `GET /api/v1/leads/reactivations`, `GET /api/v1/leads/proposals`; shared handlers in `app/api/v1/leads/_handler.ts`; rules in `lib/leads/*` (`encerramento.ts`, `motivo-da-perda.ts`, `clonar-para-funil.ts`, `reabertura.ts`, `campos-exigidos.ts`, `radar-de-risco.ts`, `risk-worker.ts`, `reactivation.ts`, `score-writer.ts`); cron `app/api/v1/cron/risk-watcher` (every 15 min via `docker/scheduler/entrypoint.sh`); triggers `trg_crm_lead_close_on_stage`, `trg_validate_lost_reason_required`, `trg_lead_so_liga_a_propria_empresa`; pages `app/app/leads/[id]` (deal dossier) and `app/app/radar` (risk radar).

## Requirements

### Requirement: Lead writes require agent role and support-write guard
`POST /api/v1/leads`, `PATCH /api/v1/leads/[id]` and `POST /api/v1/leads/[id]/{move,win,lose,clone,retomar}` and `POST /api/v1/leads/bulk` SHALL call `requireSupportWrite()` and require role `agent` or higher (PATCH accepts either a session or a `dsk_` bearer with scope `mcp:write` via `resolveAuthDual`), with `organization_id` taken from the session or token row, never the body.

#### Scenario: Viewer tries to move a lead
- **WHEN** a `viewer` calls `POST /api/v1/leads/[id]/move`
- **THEN** the `requireRole` denial is returned and `crm_leads` is unchanged

#### Scenario: Integration updates a custom field by token
- **WHEN** a server sends `PATCH /api/v1/leads/[id]` with `Authorization: Bearer dsk_...` holding `mcp:write`
- **THEN** the lead of the token's organization is updated and the response is 200 with the updated row

### Requirement: Lead creation validates stage and pipeline coherence
`POST /api/v1/leads` SHALL require `pipeline_id`, `stage_id` and a 2–200 char `title`, return 422 `stage_pipeline_mismatch` when the stage belongs to another pipeline, append the lead at `MAX(position_in_stage) + 1000` of the stage, default `currency` to the organization currency when omitted, and respond 201 (adding `meta.avisos: ["negocio_aberto_existente"]` when the contact already has an open lead in that pipeline).

#### Scenario: Stage from another pipeline
- **WHEN** the body carries a `stage_id` whose `pipeline_id` differs from `pipeline_id`
- **THEN** the response is 422 `stage_pipeline_mismatch` and no row is inserted

#### Scenario: Duplicate open deal for a contact
- **WHEN** a lead is created for a `contact_id` that already has an open lead in the same pipeline
- **THEN** the lead is still created with 201 and the response carries `meta.negocio_aberto_existente`

### Requirement: Status is derived from the stage by trigger
`crm_leads.status` SHALL be set only by `fn_crm_lead_close_on_stage` (BEFORE INSERT OR UPDATE OF `stage_id`): entering an `is_won` stage sets `status='won'` and `closed_at`, entering an `is_lost` stage sets `status='lost'`, `closed_at` and `lost_from_stage_id`, and leaving a terminal stage for an open one resets `status='open'` and `closed_at=null`, with CHECKs `crm_leads_status_enum`, `crm_leads_closed_at_consistency` and `crm_leads_currency_iso` (`^[A-Z]{3}$`).

#### Scenario: Card dragged into the won column
- **WHEN** a lead's `stage_id` is updated to a stage with `is_won = true`
- **THEN** the row ends with `status = 'won'` and non-null `closed_at` without the API writing `status`

#### Scenario: Reopening in the default mode
- **WHEN** a lost lead in a pipeline without `settings.reabertura = "novo_negocio"` is moved to an open stage
- **THEN** the row ends with `status = 'open'`, `closed_at = null` and `lost_from_stage_id = null`

### Requirement: Move is same-pipeline with optimistic concurrency
`POST /api/v1/leads/[id]/move` SHALL require `stage_id`, numeric `position_in_stage` and `expected_updated_at`, return 422 `pipeline_immutable_use_clone` (with `details.use = "/api/v1/leads/{id}/clone"`) when the target stage is in another pipeline, filter the UPDATE by `updated_at = expected_updated_at`, and return 409 `lead_stage_changed_concurrent` with `details.current_updated_at` when zero rows match.

#### Scenario: Stale drag
- **WHEN** another user changed the lead after the client read it and the client moves it with the old `expected_updated_at`
- **THEN** the response is 409 `lead_stage_changed_concurrent` and the lead keeps the other user's stage

#### Scenario: Cross-pipeline drag
- **WHEN** `stage_id` belongs to a different pipeline than the lead
- **THEN** the response is 422 `pipeline_immutable_use_clone` and nothing is written

#### Scenario: Successful move
- **WHEN** the move is accepted
- **THEN** a `stage_changed` row is written to `crm_lead_activities`, an event `lead.stage_changed` is emitted via `emit_event`, an audit `lead.moved` is written, and the response is the re-read lead with its new `updated_at`

### Requirement: Closing as lost requires a valid reason
Every write that would close a lead as lost SHALL carry a non-empty `lost_reason` that is either canonical (`CANONICAL_LOST_REASONS`, mirrored by `fn_validate_lost_reason_required`) or listed in `crm_pipelines.settings.lost_reasons`, answering 422 `lost_reason_required` when missing and 422 `lost_reason_invalid` when outside the vocabulary, and the CHECK `crm_leads_lost_reason_required` SHALL forbid `status='lost'` with an empty reason.

#### Scenario: Lose without reason
- **WHEN** `POST /api/v1/leads/[id]/lose` is called with `{}` or an empty `lost_reason`
- **THEN** the response is 422 `lost_reason_required`

#### Scenario: Drag into the lost column without reason
- **WHEN** `POST /api/v1/leads/[id]/move` targets an `is_lost` stage, the body has no `lost_reason` and the lead has none stored
- **THEN** the response is 422 `lost_reason_required` and the stage is unchanged

#### Scenario: Reason outside the vocabulary
- **WHEN** the lose request carries `lost_reason: "achou caro"` that is neither canonical nor in the pipeline's `settings.lost_reasons`
- **THEN** the database trigger raises `lost_reason_invalid` and the API answers 422 `lost_reason_invalid`

### Requirement: Win and lose move the lead to the terminal stage
`POST /api/v1/leads/[id]/win` and `POST /api/v1/leads/[id]/lose` SHALL move the lead to the first non-archived `is_won` / `is_lost` stage of its pipeline through `encerraDemanda`, return 200 with the current row unchanged when the lead already has that status, and return 422 `pipeline_no_won_stage` / `pipeline_no_lost_stage` when the pipeline has no such stage.

#### Scenario: Winning an already won lead
- **WHEN** `POST /api/v1/leads/[id]/win` is called on a lead with `status = 'won'`
- **THEN** the response is 200 with the existing row and no write happens

#### Scenario: Pipeline without a lost stage
- **WHEN** a lead whose pipeline has no non-archived `is_lost` stage is lost
- **THEN** the response is 422 `pipeline_no_lost_stage`

### Requirement: Clone moves a deal to another pipeline
`POST /api/v1/leads/[id]/clone` SHALL create the copy in the target pipeline through `createLeadHandler` first and then close the origin as lost with the caller's `lost_reason` or `moved_to_another_pipeline`, returning 201, and SHALL refuse before any write with 422 `pipeline_unchanged` (same pipeline), `lead_not_open` (closed origin outside `novo_negocio` mode), `stage_pipeline_mismatch` (stage not in target), `stage_destino_terminal`, `pipeline_without_initial_stage`, `pipeline_no_lost_stage` (origin cannot close) or `lost_reason_invalid`, and 404 `pipeline_not_found`.

#### Scenario: Clone to the same pipeline
- **WHEN** `pipeline_id` equals the lead's current pipeline
- **THEN** the response is 422 `pipeline_unchanged` and no lead is created

#### Scenario: Successful clone
- **WHEN** an open lead is cloned to another pipeline without `stage_id`
- **THEN** a new lead exists in the first open stage of the target, the origin has `status = 'lost'` with `lost_reason = 'moved_to_another_pipeline'`, and the response is 201

### Requirement: Lead visibility follows the organization visibility mode
RLS on `crm_leads` SHALL use `fn_can_view_lead(organization_id, owner_user_id)`: platform admins, `viewer`, `manager` and `admin` see the whole organization, while an `agent` sees own leads plus, per `organizations.settings.visibility_mode`, all leads (`all`), unassigned leads (`own_and_unassigned`, the default) or nothing else (`own`), and `crm_lead_activities` / `crm_lead_links` SELECT SHALL inherit this through the parent lead.

#### Scenario: Agent in default mode
- **WHEN** an `agent` of an org with no `visibility_mode` set reads `crm_leads`
- **THEN** rows owned by another user are not returned, while own rows and rows with `owner_user_id IS NULL` are

#### Scenario: Agent edits another agent's lead
- **WHEN** an `agent` in `own` mode updates a lead owned by someone else
- **THEN** the `crm_leads_update` policy filters the row and the API returns 404 `not_found`

### Requirement: Lead links stay inside the organization
`fn_lead_so_liga_a_propria_empresa` SHALL reject a `crm_leads` insert or update whose `contact_id` is not a contact of the same organization (PT404) or whose `owner_user_id` is not an active non-viewer member of it (PT422), and `crm_lead_links.target_kind` SHALL be one of `order`, `conversation`, `message`, `appointment`, `contact`, `lead`, `external`.

#### Scenario: Owner from another organization
- **WHEN** a lead is updated with an `owner_user_id` that has no active membership in the lead's organization
- **THEN** the database raises PT422 and the row keeps its previous owner

#### Scenario: Related contacts
- **WHEN** `GET /api/v1/leads/[id]/contatos-relacionados` is called
- **THEN** only `crm_lead_links` rows with `target_kind='contact'` and `link_kind='related'` whose contact belongs to the lead's organization are returned

### Requirement: Bulk actions are capped and assign is manager-only
`POST /api/v1/leads/bulk` SHALL accept `action` in `move`, `assign`, `tag`, `delete` with 1–50 `lead_ids`, scope every write to the active organization, require role `manager` for `assign`, and answer 422 `invalid_owner` when the new owner is not an active non-viewer member (422 `owner_validation_unavailable` when the service role is not configured).

#### Scenario: Agent bulk-assigns
- **WHEN** an `agent` sends `{ "action": "assign", ... }`
- **THEN** the manager `requireRole` denial is returned and no owner changes

#### Scenario: Too many ids
- **WHEN** the body carries 51 `lead_ids`
- **THEN** the response is 422 and no lead is touched

### Requirement: Risk radar and reactivation proposals
`app/api/v1/cron/risk-watcher` (authorized by `INTERNAL_CRON_SECRET`/`INTERNAL_SECRET`, 403 `forbidden` otherwise) SHALL store each open lead's risk bucket in `crm_lead_risk_states`, emit `lead_cooled` / `lead_reactivated` activities on bucket crossings, create a pending `crm_lead_reactivations` proposal when a lead cools, and expire overdue proposals, while `GET /api/v1/leads/at-risk` (agent+, `limit` 1–200, 422 `validation_failed` on bad query) lists cooled open leads and `POST /api/v1/leads/[id]/reactivation` decides a proposal with 409 `reactivation_not_pending` when it is no longer pending.

#### Scenario: Deciding an expired proposal
- **WHEN** a user posts `{ "decision": "accept", "proposal_id": <id> }` for a proposal whose status is `expired`
- **THEN** the response is 409 `reactivation_not_pending` and no `cron_jobs` follow-up is scheduled

#### Scenario: Cron without secret
- **WHEN** the risk-watcher route is called without a valid cron bearer
- **THEN** the response is 403 `forbidden`

### Requirement: Lead score lives outside crm_leads
The AI probability score SHALL be upserted by `recalculaScoreDoLead` into `crm_lead_scores` (one row per `lead_id`, `ai_probability`, `ai_probability_band`, `ai_probability_band_since` that only moves when the band changes) from the agent inbound turn, never by trigger and never as a column of `crm_leads`.

#### Scenario: Same band recalculated
- **WHEN** a lead's score is recalculated and the band is unchanged
- **THEN** `ai_probability` and `updated_at` change but `ai_probability_band_since` keeps its previous value, and `crm_leads.updated_at` is untouched

### Requirement: Agent-created deal in "own" mode belongs to the agent
`POST /api/v1/leads` SHALL, when the caller's role is `agent`, the body names neither `owner_user_id` nor `owner_agent_id`, and the active organization's `settings.visibility_mode` (read with the admin client from the trusted active organization) is `own`, set `owner_user_id` to the caller before creating the lead.

#### Scenario: Agent creates a deal without owner in "own" mode
- **WHEN** an `agent` of an organization with `visibility_mode = 'own'` posts a valid lead without owner fields
- **THEN** the response is 201 and the lead has `owner_user_id` equal to the caller

#### Scenario: Default mode
- **WHEN** the same request comes from an organization with `visibility_mode = 'own_and_unassigned'`
- **THEN** the lead is created without owner

### Requirement: Visibility refusal on a lead write is a permission error
Lead write handlers in `app/api/v1/leads/_handler.ts` SHALL map a Postgres `42501` whose message names `"crm_leads"` (the RLS check of `fn_can_view_lead`) to HTTP 403 `forbidden` with a translated message, while any other `42501` stays HTTP 500.

#### Scenario: Agent assigns a colleague in "own" mode
- **WHEN** an `agent` in `visibility_mode = 'own'` creates a lead with `owner_user_id` of another member
- **THEN** the response is 403 `forbidden` instead of 500 and no row is inserted

### Requirement: The MCP move tool forwards the loss reason
The MCP tool `crm_move_lead_stage` SHALL accept an optional `lost_reason` (string, at most 500 characters) and SHALL pass it to the move handler, so that an agent moving a deal into a lost stage is checked against the funnel's loss vocabulary instead of having the reason dropped.

#### Scenario: An agent marks a deal as lost through MCP
- **WHEN** an agent calls `crm_move_lead_stage` with a lost-stage target and `lost_reason: "price"`
- **THEN** the deal moves with `status = 'lost'` and `lost_reason = 'price'`, the same outcome as `POST /api/v1/leads/[id]/move`
