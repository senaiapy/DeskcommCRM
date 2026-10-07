# pipelines-kanban Specification

## Purpose
Pipelines ("funis") and their stages are the configuration of the sales board. Each organization has one or more `crm_pipelines` rows (with `vocabulary` jsonb, `settings` jsonb holding `fields`, `lost_reasons`, `canonical_tags`, and exclusive flags `is_default` / `is_client_pipeline`) and each pipeline has ordered `crm_stages` rows (`position` numeric, `is_won` / `is_lost` terminal flags, `expected_duration_hours`, `win_probability`, `is_archived`). Entry points: API `app/api/v1/pipelines/route.ts` (GET, POST), `app/api/v1/pipelines/[id]/route.ts` (PATCH, DELETE), `app/api/v1/pipelines/[id]/stages/route.ts` (GET, POST), `app/api/v1/pipelines/[id]/stages/[stageId]/route.ts` (PATCH, DELETE), `app/api/v1/pipelines/[id]/board/route.ts` (GET), `app/api/v1/pipelines/default/route.ts`, `.../forecast`, `.../stages/win-rates`, `.../agent-mapping`; pages `app/app/kanban` (pipeline list), `app/app/pipelines/[id]` (board), `app/app/crm` (CRM hub); libraries `lib/pipelines/pipeline-editing.ts`, `lib/leads/stage-operations.ts`, `lib/kanban/*` (fractional `midpoint()`, vocabulary, card state). Lead movement between stages is specified in `leads-deals`.

## Requirements

### Requirement: Pipeline configuration is manager-only
Every mutating pipeline and stage route (`POST /api/v1/pipelines`, `PATCH`/`DELETE /api/v1/pipelines/[id]`, `POST /api/v1/pipelines/[id]/stages`, `PATCH`/`DELETE /api/v1/pipelines/[id]/stages/[stageId]`) SHALL call `requireSupportWrite()` and then `requireRole("manager")`, taking `organization_id` from the session and never from the body.

#### Scenario: Agent tries to create a pipeline
- **WHEN** a user whose role in the active organization is `agent` sends `POST /api/v1/pipelines`
- **THEN** the response is the `requireRole` denial and no `crm_pipelines` row is inserted

#### Scenario: Read-only support impersonation
- **WHEN** a platform admin impersonating in read-only support mode calls `PATCH /api/v1/pipelines/[id]`
- **THEN** `requireSupportWrite()` returns its denial before any write happens

### Requirement: Pipeline creation seeds four starter stages
`POST /api/v1/pipelines` SHALL insert the `crm_pipelines` row and then the four `ETAPAS_INICIAIS` stages (`novo`, `em_andamento`, `ganho` with `is_won=true`, `perdido` with `is_lost=true`) at positions 1000..4000, deleting the pipeline again and returning 500 if the stage insert fails.

#### Scenario: New pipeline created
- **WHEN** a manager posts `{ "name": "Clínica" }` with a name that no other pipeline of the org uses
- **THEN** the response is 201 with the re-read pipeline list, the new pipeline has a slug derived from the name, and it owns one won stage and one lost stage

#### Scenario: First pipeline of the org becomes default
- **WHEN** the organization has no non-archived pipeline and a manager creates one
- **THEN** the inserted row has `is_default = true`

#### Scenario: Invalid body
- **WHEN** the body is not JSON
- **THEN** the response is 400 `invalid_request`; and when the name is empty or longer than 80 chars the response is 422 `unprocessable_entity`

### Requirement: DELETE archives a pipeline unless it is a clean accidental one
`DELETE /api/v1/pipelines/[id]` SHALL set `crm_pipelines.is_archived = true` by default, and SHALL only hard-delete when `?definitivo=1` is passed and the pipeline has zero `crm_leads`, refusing with 422 `unprocessable_entity` (with `details.negocios`, `details.fontes_de_webhook`, `details.automacoes`) when it is the only active pipeline, the default pipeline, the target of a `webhook_sources.default_pipeline_id`, or referenced by an active automation rule.

#### Scenario: Archiving the default pipeline
- **WHEN** a manager deletes the pipeline that has `is_default = true`
- **THEN** the response is 422 `unprocessable_entity` and the row is unchanged

#### Scenario: Hard delete of a pipeline with deals
- **WHEN** `DELETE /api/v1/pipelines/[id]?definitivo=1` targets a pipeline that has at least one `crm_leads` row
- **THEN** the response is 422 with `details.negocios` greater than zero and nothing is deleted

#### Scenario: Plain archive
- **WHEN** a non-default, non-sole pipeline with no webhook source or active automation is deleted without `definitivo`
- **THEN** the row stays with `is_archived = true` and an audit `pipeline.archived` is written

### Requirement: Exclusive pipeline flags are unique per organization
The database SHALL allow at most one `crm_pipelines` row with `is_default = true` per organization (`uniq_crm_pipelines_org_default`) and a unique `(organization_id, slug)` (`uniq_crm_pipelines_org_slug`, slug matching `^[a-z0-9_-]{2,40}$`), and `PATCH /api/v1/pipelines/[id]` with `is_default: true` SHALL clear the previous default before setting the new one.

#### Scenario: Electing a new default
- **WHEN** a manager sends `PATCH /api/v1/pipelines/[id]` with `{ "is_default": true }`
- **THEN** the previous default row ends with `is_default = false`, the target with `is_default = true`, and no 23505 is returned

### Requirement: Stage terminal flags are mutually exclusive and single per pipeline
`crm_stages` SHALL reject a row with both `is_won` and `is_lost` (`crm_stages_won_lost_mutex`) and SHALL allow at most one non-archived won stage and one non-archived lost stage per pipeline (`uniq_crm_stages_pipeline_won`, `uniq_crm_stages_pipeline_lost`), with `PATCH /api/v1/pipelines/[id]/stages/[stageId]` releasing the old flag holder before marking the new one.

#### Scenario: Moving the won flag to another stage
- **WHEN** a manager patches stage B with `{ "is_won": true }` while stage A is the current won stage
- **THEN** stage A ends with `is_won = false`, stage B with `is_won = true`, and the response returns the re-read pipeline

#### Scenario: Editing an archived stage
- **WHEN** the target stage has `is_archived = true`
- **THEN** the PATCH returns 409 `state_conflict` and no stage is updated

### Requirement: Archiving a stage requires a destination for its deals
`DELETE /api/v1/pipelines/[id]/stages/[stageId]` SHALL archive the stage (`is_archived = true`, never a SQL delete) and, when the stage still holds `crm_leads`, SHALL require `?destino=<stageId>`, moving the leads to the destination before archiving, otherwise returning 422 `unprocessable_entity` with `details.negocios` and `details.precisa_destino`.

#### Scenario: Archiving a stage with deals and no destination
- **WHEN** a manager deletes a stage that holds 2 leads without `destino`
- **THEN** the response is 422 with `details.negocios = 2` and the stage stays active

#### Scenario: Archiving with destination
- **WHEN** the same request carries `?destino=<other stage id>`
- **THEN** the 2 leads now have `stage_id = <other stage id>` and the archived stage has `is_archived = true`

### Requirement: Ordering uses fractional midpoint positions
Reordering of pipelines (`PATCH /api/v1/pipelines/[id]` with `depois_de`) and stages (`PATCH .../stages/[stageId]` with `depois_de`) SHALL compute the new `position` with `midpoint(prev, next)` from `lib/kanban/fractional-indexing.ts` (STEP 1000 at the ends, arithmetic mean between neighbours), returning 409 `state_conflict` when the neighbours share the same position (midpoint is NaN) and 422 `unprocessable_entity` when `depois_de` is not an active sibling.

#### Scenario: Move to first place
- **WHEN** a stage is patched with `{ "depois_de": null }` and the first active stage has `position = 1000`
- **THEN** the stage is stored with `position = 0`

#### Scenario: Tied neighbours
- **WHEN** the two neighbours around the requested slot have equal `position`
- **THEN** the response is 409 `state_conflict` and no position is written

### Requirement: Stage creation and validation ranges
`POST /api/v1/pipelines/[id]/stages` SHALL append the stage at `midpoint(highest existing position, null)` (archived stages included in the count) with a name of 1–80 chars and an optional integer `expected_duration_hours` between 1 and 8760, and stage PATCH SHALL accept `win_probability` only as an integer 0–100 or null, returning 422 `unprocessable_entity` otherwise.

#### Scenario: Stage created at the end
- **WHEN** a manager posts `{ "name": "Proposta" }` to a pipeline whose highest stage position is 3000
- **THEN** the response is 201 and the new stage has `position = 4000`

#### Scenario: Out-of-range duration
- **WHEN** the body carries `expected_duration_hours: 0`
- **THEN** the response is 422 `unprocessable_entity`

### Requirement: Board snapshot is served through the API under RLS
`GET /api/v1/pipelines/[id]/board` SHALL authenticate with `supabase.auth.getUser()` (401 `unauthenticated` otherwise) and return the pipeline, its non-archived stages ordered by `position`, and its `crm_leads` ordered by `position_in_stage`, using the session client so `crm_leads_select` (`fn_can_view_lead`) filters what the caller sees, with 404 `resource_not_found` when the pipeline is not visible.

#### Scenario: Pipeline of another organization
- **WHEN** a signed-in user requests the board of a pipeline outside their organizations
- **THEN** the response is 404 `resource_not_found`

#### Scenario: Agent in own-only visibility
- **WHEN** an `agent` whose organization has `settings.visibility_mode = 'own'` loads the board
- **THEN** the returned leads include only rows with `owner_user_id` equal to that agent

### Requirement: Default pipeline lookup
`GET /api/v1/pipelines/default` SHALL require role `agent` or higher and return the organization's non-archived pipeline ordered by `is_default` desc then `position` (so the default when one exists) with its non-archived stages, or 404 `resource_not_found` when the organization has no pipeline configured.

#### Scenario: Org without pipelines
- **WHEN** an agent of an organization with no `crm_pipelines` row calls the route
- **THEN** the response is 404 `resource_not_found`

### Requirement: Pipeline vocabulary and field settings have safe defaults
`crm_pipelines.vocabulary` SHALL default to a jsonb with the `lead`, `deal`, `won`, `lost` and `stage` labels and `crm_pipelines.settings` SHALL default to `{ fields: [], canonical_tags: [], lost_reasons: [], identity_resolution: {...} }`, and the UI SHALL resolve missing vocabulary keys through `resolveVocabulary()` (defaults Lead / Negócio / Ganho / Perdido).

#### Scenario: Pipeline without custom vocabulary
- **WHEN** a pipeline row has a vocabulary object without the `won` key
- **THEN** `resolveVocabulary()` returns `won = "Ganho"`
