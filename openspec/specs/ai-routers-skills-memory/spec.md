# ai-routers-skills-memory Specification

## Purpose
Three configuration surfaces that shape which agent answers and what it knows on every turn: intent routers per WhatsApp number (`ai_routers`, `ai_router_members`) with the Jev fast classifier and its decision log (`jev_router_decisions`), situational skills versioned by pointer (`skill_versions`, `skill_pointers`), and organization memory (`org_memory_versions`, `org_memory_pointers`, `org_memory_entries`). The turn runtime that consumes them is covered by `ai-agents-runtime`; human (non-AI) conversation routing in `lib/routing` and the `routing-worker` cron are outside this capability.

## Requirements

### Requirement: One active router per WhatsApp number
`POST /api/v1/ai/routers` SHALL require role `admin`, verify `channel_session_id` belongs to the session organization (404 `channel_session_not_found`), store `config.context_message_count` between 0 and 16, and answer 409 `router_already_exists` when the partial unique index on active routers per `channel_session_id` rejects the insert.

#### Scenario: Second router on the same number
- **WHEN** an admin creates a router for a number that already has an active router
- **THEN** the response is 409 `router_already_exists`, not 500

### Requirement: Router roles are split between read, test and write
Router listing and detail SHALL require role `agent`, `POST /api/v1/ai/routers/{id}/test` SHALL require role `manager`, and create, `PATCH`, `DELETE` and `PUT /api/v1/ai/routers/{id}/members` SHALL require role `admin`, with members refused by 422 `validation_failed` when an agent is outside the organization.

#### Scenario: Member from another organization
- **WHEN** an admin puts an agent id of another organization in a router's members
- **THEN** the response is 422 and the member list is unchanged

### Requirement: Turn routing resolves sticky, classified, fallback, then generic
At runtime the router SHALL keep the sticky agent unless a different intent arrives with `confidence >= min_confidence`, choose the member of a matched intent with `confidence >= min_confidence`, otherwise use `fallback_agent_id`, and record the outcome in `ai_router_decisions`.

#### Scenario: Classifier failure with a sticky agent
- **WHEN** the classifier returns null while the conversation has an eligible sticky agent
- **THEN** the sticky agent keeps the turn

### Requirement: Router tests never write production telemetry
`POST /api/v1/ai/routers/{id}/test` SHALL classify the sample message with the same `min_confidence` rule as the runtime without writing `ai_router_decisions`, `jev_router_decisions` or `conversations`, recording only the model cost in `llm_calls`.

#### Scenario: Test click
- **WHEN** a manager tests a router with a sample message
- **THEN** the response shows the intent, confidence and `min_confidence`, and no decision row is created

### Requirement: Jev requires a validated key and explicit LGPD consent
`PATCH /api/v1/ai/jev` SHALL require role `admin`, store its switch in `organizations.settings.jev`, refuse turning it on with 422 `jev_exige_chave_validada` when no active validated Jev credential exists, refuse the first activation without `aceite_lgpd: true` with 422 `jev_exige_aceite`, and record the consent with timestamp and user.

#### Scenario: First activation without consent
- **WHEN** an admin with a validated key sets `ligado: true` without `aceite_lgpd`
- **THEN** the response is 422 `jev_exige_aceite` and the setting stays off

### Requirement: On-demand routing needs the regular AI
`PATCH /api/v1/ai/jev` SHALL refuse `modo_roteador: 'sob_demanda'` with 422 `jev_sem_ia_de_sempre` when the organization has no regular classifier configured, and SHALL refuse `estado: 'decidindo'` for observe-only tasks with 422 `jev_tarefa_so_observa`.

#### Scenario: Organization without the regular AI
- **WHEN** an admin selects `sob_demanda` with no regular LLM key
- **THEN** the response is 422 `jev_sem_ia_de_sempre` and `modo_roteador` stays `comparacao`

### Requirement: Router decisions are logged once per message and reviewable
The runtime SHALL insert one `jev_router_decisions` row per `(organization_id, router_id, message_id)`, ignoring conflicts and never failing the turn on a logging error; `GET /api/v1/ai/routing-results` SHALL require role `manager` and `PATCH /api/v1/ai/routing-results/{id}` SHALL require role `admin` and accept `agent_id_esperado` only with `revisao = 'incorreto'` and only for a member of that router.

#### Scenario: Review with an outside agent
- **WHEN** an admin marks a decision `incorreto` with an expected agent that is not a router member
- **THEN** the response is 422 `invalid_body`

### Requirement: Skills are immutable versions moved by pointer
Editing a skill via `PUT /api/v1/ai/skills/{name}` SHALL require role `manager` and insert a new `skill_versions` row, and `POST /api/v1/ai/skills/{name}/restore` SHALL move the organization's `skill_pointers` row back to an existing version of the same name and organization without creating a version.

#### Scenario: Restore an old version
- **WHEN** a manager restores version 2 of a skill currently at version 4
- **THEN** the pointer references version 2 and no new `skill_versions` row exists

### Requirement: Platform skills are forked on install and packages are size-limited
`POST /api/v1/ai/skills/{name}/install` SHALL require role `manager`, answer 404 `not_found` when no platform skill (`skill_pointers.organization_id is null`) has that name, and copy it into the organization catalog; `POST /api/v1/ai/skills/import` SHALL answer 413 `skill_upload_too_large` above 5 MB.

#### Scenario: Install unknown skill
- **WHEN** a manager installs a name absent from the platform catalog
- **THEN** the response is 404 and nothing is copied

### Requirement: Organization memory is published as a new version
`POST /api/v1/ai/memory` SHALL require role `admin`, insert `org_memory_versions` with the next `version_number`, upsert `org_memory_pointers` on `organization_id`, audit `ai.org_memory_published`, and answer 500 distinguishing a failed version insert from a pointer that did not move.

#### Scenario: Agent reads the published memory
- **WHEN** a new memory version is published
- **THEN** the next turn's prompt contains that version's content plus the `org_memory_entries` with `status = 'active'`

### Requirement: Memory entries are toggled, not deleted
`POST /api/v1/ai/memory/entries` and `PATCH /api/v1/ai/memory/entries/{id}` SHALL require role `manager`, and the patch SHALL accept only `status` in `active` or `archived`, answering 404 for an entry outside the organization.

#### Scenario: Archive an entry
- **WHEN** a manager patches an entry to `archived`
- **THEN** the entry stops appearing in the agent prompt and the row remains stored
