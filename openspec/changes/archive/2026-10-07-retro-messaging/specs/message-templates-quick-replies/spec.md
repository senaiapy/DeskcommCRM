## Purpose
Quick replies ("Respostas rápidas") are saved attendant scripts, personal or shared with the team, inserted by the inbox composer with `{{nome}}` / `{{primeiro_nome}}` interpolation; campaign templates are reusable free-text campaign copy (not provider/Meta templates). Key paths: page `app/app/templates/page.tsx` (title "Respostas rápidas") with `app/app/templates/_components/TemplatesClient.tsx` and `TemplateFormDialog.tsx`; API `app/api/v1/message-templates/route.ts` (GET, POST) and `app/api/v1/message-templates/[id]/route.ts` (PATCH, DELETE); API `app/api/v1/campaign-templates/route.ts` (GET, POST) and `app/api/v1/campaign-templates/[id]/route.ts` (PATCH, DELETE); schemas `lib/schemas/templates.ts` and `lib/campanhas/schemas.ts`; interpolation `lib/inbox/template-vars.ts`; tables `message_templates` (migration 0060) and `campaign_templates`.

## ADDED Requirements

### Requirement: Quick reply storage and visibility
The `message_templates` table SHALL store `organization_id`, `owner_user_id` (null means shared with the organization), `title`, `body`, `shortcut` and `created_by_user_id`, with RLS policy `message_templates_select` exposing only rows of `fn_user_org_ids()` that are shared or owned by `auth.uid()` (or any row to a platform admin).

#### Scenario: Another agent's personal script is invisible
- **WHEN** agent A lists quick replies and agent B owns a personal row (`owner_user_id = B`) in the same organization
- **THEN** B's row is not returned to A

#### Scenario: Shared script is visible to all members
- **WHEN** a row has `owner_user_id IS NULL` in the caller's organization
- **THEN** it is returned by `GET /api/v1/message-templates`

### Requirement: Quick reply write permissions
The RLS policy `message_templates_write` SHALL allow writing a personal row only when `owner_user_id = auth.uid()` and the caller is `agent`+, and a shared row only when the caller is `manager`+, and `POST /api/v1/message-templates` with `shared = true` from a caller below manager SHALL return 403 `forbidden`.

#### Scenario: Agent tries to create a shared script
- **WHEN** an agent posts `{ "title": "Oi", "body": "Olá", "shared": true }`
- **THEN** the response is 403 `forbidden` and no row is inserted

#### Scenario: Agent creates a personal script
- **WHEN** an agent posts `{ "title": "Oi", "body": "Olá {{primeiro_nome}}" }`
- **THEN** the response is 201 with `owner_user_id` equal to the caller and an audit row `template.created` is written

### Requirement: Quick reply list
`GET /api/v1/message-templates` SHALL require role `agent`, filter by the active organization and order by `updated_at` descending, returning 500 `internal_error` on query failure.

#### Scenario: Viewer is denied
- **WHEN** a `viewer` calls `GET /api/v1/message-templates`
- **THEN** the `requireRole("agent")` gate rejects the request

### Requirement: Quick reply creation validation and idempotency
`POST /api/v1/message-templates` SHALL call `requireSupportWrite`, validate `title` (1–80), `body` (1–4096) and optional `shortcut` (1–40) returning 422 `validation_failed` on failure, reject a non-UUID `Idempotency-Key` with 400 `validation_error`, and on key reuse return 409 `idempotency_conflict` (different body) or 409 `idempotency_in_progress` (same body still running).

#### Scenario: Replay with the same key and body
- **WHEN** the same POST is retried with the same `Idempotency-Key` after it completed
- **THEN** the response is 201 with the originally created template and no second row is inserted

#### Scenario: Same key, different body
- **WHEN** the key is reused with a different body
- **THEN** the response is 409 `idempotency_conflict`

### Requirement: Quick reply update and delete
`PATCH /api/v1/message-templates/{id}` SHALL require role `agent` and at least one of `title`, `body`, `shortcut` (422 `validation_failed` otherwise), and both PATCH and `DELETE /api/v1/message-templates/{id}` SHALL return 404 `not_found` when the row is absent or not writable under RLS, DELETE returning 204 and writing audit `template.deleted`.

#### Scenario: Empty patch
- **WHEN** the PATCH body is `{}`
- **THEN** the response is 422 `validation_failed`

#### Scenario: Agent deletes a shared script
- **WHEN** an agent deletes a row with `owner_user_id IS NULL`
- **THEN** RLS removes nothing and the response is 404 `not_found`

### Requirement: Template variable interpolation
`interpolateTemplate` in `lib/inbox/template-vars.ts` SHALL replace `{{nome}}` with the contact's full name and `{{primeiro_nome}}` with its first word, and SHALL keep the literal placeholder for unknown variables or empty values.

#### Scenario: Contact without name
- **WHEN** the body `Olá {{primeiro_nome}}` is interpolated for a contact with empty `name`
- **THEN** the result is `Olá {{primeiro_nome}}`

### Requirement: Campaign templates entity
The `campaign_templates` table SHALL store `organization_id`, `name`, `body`, `created_by` and `updated_by` with non-blank `name`/`body` checks and the unique constraint `campaign_templates_nome_unico (organization_id, name)`, and its routes SHALL require role `manager` and write through the admin client filtered by the active `organization_id`.

#### Scenario: Duplicate name
- **WHEN** a manager posts `POST /api/v1/campaign-templates` with a `name` already used in the organization
- **THEN** the response is 409 `campanha_conteudo_invalido`

#### Scenario: Create campaign template
- **WHEN** a manager posts `{ "name": "Black Friday", "body": "Oferta para {{nome}}" }`
- **THEN** the response is 201 with `id, name, body, created_at, updated_at, created_by`

#### Scenario: Agent lists campaign templates
- **WHEN** an `agent` calls `GET /api/v1/campaign-templates`
- **THEN** the `requireRole("manager")` gate rejects the request

### Requirement: Campaign template update and delete
`PATCH` and `DELETE /api/v1/campaign-templates/{id}` SHALL call `requireSupportWrite`, require role `manager`, and return 404 `campanha_nao_encontrada` when no row with that `id` exists in the active organization, DELETE returning 200 with `{ id }`.

#### Scenario: Delete unknown template
- **WHEN** a manager deletes an id that does not exist in the organization
- **THEN** the response is 404 `campanha_nao_encontrada`
