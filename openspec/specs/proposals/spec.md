# proposals Specification

## Purpose
Commercial proposals attached to a deal (lead): a draft with priced items, a confirmed document template, a numbered PDF sent to the contact over the conversation channel, and a customer decision. Pages `app/app/proposals` (list, `novo`, `[id]` editor; the layout returns notFound when the capability is off). API `app/api/v1/proposals` (GET/POST), `proposals/[id]` (GET/PATCH/DELETE), `proposals/[id]/{send,decide,revise,modelo,documento,previa,assistant,assistant/apply,preencher-com-conversa}`, settings at `app/api/v1/settings/proposals` and `app/api/v1/settings/proposal-templates[/slug|/importar]`. Logic in `lib/propostas/*` (items, pricing, numbering, versioning, document/PDF rendering, storage, gate `porta.ts`). Tables `crm_proposals`, `crm_proposal_items`, `crm_proposal_counters`, `proposal_templates`; PDFs in the private storage bucket `propostas`. Crons `app/api/v1/cron/{proposal-expiry,proposta-travada,proposal-acceptance-rate,proposal-promised-not-created}`. There is no public acceptance link: the decision is recorded by a team member.

## Requirements

### Requirement: Organization capability gate
Every `/api/v1/proposals*` route SHALL return 404 `not_found` (via `sePropostasDesligadas`) when the organization capability `propostas` is not enabled in `organizations.settings.proposals.enabled`.

#### Scenario: Capability disabled
- **WHEN** a member of an organization whose `settings.proposals.enabled` is not `true` calls `GET /api/v1/proposals`
- **THEN** the response is 404 with error code `not_found`

#### Scenario: Enabling through settings
- **WHEN** a manager sends `PATCH /api/v1/settings/proposals` with `{enabled: true, default_valid_days, default_conditions}`
- **THEN** the values are shallow-merged into `organizations.settings.proposals` and subsequent proposal routes stop returning 404

### Requirement: Draft creation scoped to the organization
`POST /api/v1/proposals` SHALL require role `agent`, verify the `lead_id` belongs to the active organization, and insert a `crm_proposals` row with `status = 'rascunho'`, `numero`/`ano` NULL and `moeda` taken from the organization currency, returning 201 with the new `id`.

#### Scenario: Lead from another organization
- **WHEN** the body carries a `lead_id` not found in the active organization
- **THEN** the response is 404 `not_found` and no row is written

#### Scenario: Lead without contact
- **WHEN** the lead has no `contact_id`
- **THEN** the response is 422 `validation_failed`

#### Scenario: Draft created
- **WHEN** an agent posts a valid body for a lead with a contact
- **THEN** the response is 201 `{id}`, a `crm_lead_activities` entry of type `proposal_drafted` is emitted and `valid_until` defaults to today plus `default_valid_days` in the organization time zone

### Requirement: One open draft per deal
The system SHALL allow at most one `crm_proposals` row with `status = 'rascunho'` per `(organization_id, lead_id)`, enforced by the unique partial index `crm_proposals_rascunho_unico_por_negocio_uidx`.

#### Scenario: Second draft for the same deal
- **WHEN** `POST /api/v1/proposals` is called for a lead that already has a draft
- **THEN** the response is 409 `validation_failed` with `details.rascunho_aberto_id`, and a concurrent insert caught as SQLSTATE 23505 also returns 409

### Requirement: Optimistic editing of drafts only
`PATCH /api/v1/proposals/[id]` SHALL require role `agent` and update only a row whose `status = 'rascunho'` and whose `revision` equals the body `revision`, incrementing `revision` and recomputing `total_cents` and `pricing_status` from the items in the proposal currency.

#### Scenario: Stale revision or non-draft
- **WHEN** the body `revision` differs from the stored one or the proposal is no longer a draft
- **THEN** the response is 409 `proposal_context_stale` and nothing is written

### Requirement: Sending allocates a permanent number and delivers the PDF
`POST /api/v1/proposals/[id]/send` SHALL require role `manager`, accept only a `rascunho` with items, no unpriced item, a confirmed `template_slug` and no missing document fields, allocate `numero`/`ano` from `crm_proposal_counters` through `fn_proposta_aloca_numero`, set `status = 'enviando'`, upload the PDF to bucket `propostas`, send it as a document message, and set `status = 'enviada'` with `sent_at` only when the message status is sent/delivered/read.

#### Scenario: Not a draft
- **WHEN** send is called on a proposal whose status is not `rascunho`
- **THEN** the response is 409 `proposal_context_stale`

#### Scenario: Missing template or unpriced item
- **WHEN** `template_slug` is NULL or `pricing_status = 'missing'`
- **THEN** the response is 422 `validation_failed` and no number is allocated

#### Scenario: Message failed
- **WHEN** the outgoing message returns status `failed` or any step throws
- **THEN** the proposal returns to `rascunho` keeping `numero`/`ano`, with the reason in `ultima_falha_envio`

#### Scenario: Delivered
- **WHEN** the message status is sent, delivered or read
- **THEN** the row becomes `enviada` with `pdf_path`, `message_id`, `template_snapshot`, `rendered_snapshot`, and `crm_leads.value_cents` is set to the proposal `total_cents`

### Requirement: Numbering never reuses a number
The system SHALL derive proposal numbers only from `crm_proposal_counters(organization_id, ano, ultimo_numero)`, which only increments and is not readable or writable by `anon`/`authenticated`.

#### Scenario: Deleting proposals does not free numbers
- **WHEN** a sent proposal or its deal is deleted and another proposal is sent in the same year
- **THEN** the new proposal receives `ultimo_numero + 1`, never the freed number

### Requirement: Customer decision recorded by the team
`POST /api/v1/proposals/[id]/decide` SHALL require role `agent`, accept `decisao` in (`aceita`,`recusada`) and update only a row with `status = 'enviada'`, writing `decided_at`, `decided_by_user_id` and `decision_reason`.

#### Scenario: Decision on a non-sent proposal
- **WHEN** the proposal is not `enviada`
- **THEN** the response is 409 `proposal_context_stale`

#### Scenario: Accepted
- **WHEN** an agent posts `{decisao: "aceita"}` on a sent proposal
- **THEN** the row becomes `aceita`, a `proposal_accepted` lead activity is emitted, and any scheduled automatic follow-up in `retorno_id` is cancelled best-effort

### Requirement: Revision chain
`POST /api/v1/proposals/[id]/revise` SHALL require role `manager` and, only for an `enviada` proposal, create a new `rascunho` inheriting `numero`/`ano` with `versao + 1` and `substitui_id` pointing to the original; the original becomes `substituida` only when the new version is effectively sent and is still `enviada`.

#### Scenario: Revising a non-sent proposal
- **WHEN** revise is called on a draft or decided proposal
- **THEN** the response is 409 `proposal_context_stale`

#### Scenario: V2 sent after V1 was decided
- **WHEN** V1 was accepted before V2 is sent
- **THEN** V1 keeps status `aceita` because the substitution update filters `status = 'enviada'`

### Requirement: Discarding drafts
`DELETE /api/v1/proposals/[id]` SHALL require role `manager` and set `status = 'cancelada'` only on a `rascunho`, never deleting the row.

#### Scenario: Discarding a sent proposal
- **WHEN** DELETE targets a proposal not in `rascunho`
- **THEN** the response is 409 `proposal_context_stale`

### Requirement: Status vocabulary
`crm_proposals.status` SHALL be constrained by `crm_proposals_status_check` to `rascunho`, `enviando`, `enviada`, `aceita`, `recusada`, `vencida`, `cancelada`, `substituida`, with `total_cents >= 0`, `moeda` matching `^[A-Z]{3}$`, and `numero`/`ano` both NULL or both set.

#### Scenario: Invalid status
- **WHEN** a write sets `status = 'aprovada'`
- **THEN** Postgres rejects it with a check violation

### Requirement: Expiry and stuck-send recovery crons
`/api/v1/cron/proposal-expiry` SHALL, when authorized by `INTERNAL_CRON_SECRET` or `INTERNAL_SECRET`, move `enviada` proposals whose `valid_until` has passed to `vencida`, and `/api/v1/cron/proposta-travada` SHALL return to `rascunho` (keeping `numero`/`ano`) proposals stuck in `enviando` for more than 5 minutes whose message is not `queued`.

#### Scenario: Unauthorized cron call
- **WHEN** either cron is called without a valid cron secret
- **THEN** the response is 403 `forbidden`

#### Scenario: Expired proposal
- **WHEN** an `enviada` proposal has `valid_until` before today
- **THEN** the cron updates it to `vencida` with a `status = 'enviada'` claim and opens an inbox notice

### Requirement: Template management and PDF preview
`/api/v1/settings/proposal-templates` SHALL allow `viewer` to read and require `manager` to create, edit, import or deactivate organization templates in `proposal_templates` (DELETE sets `is_active = false`), and `GET /api/v1/proposals/[id]/previa` SHALL require `agent` and return the rendered PDF with `Content-Type: application/pdf`.

#### Scenario: Preview without template
- **WHEN** the proposal has no confirmed template
- **THEN** `GET /previa` returns 422 `validation_failed`

#### Scenario: Agent deactivates template
- **WHEN** an `agent` calls `DELETE /api/v1/settings/proposal-templates/[slug]`
- **THEN** the response is 403 `forbidden_role` and the template stays active
