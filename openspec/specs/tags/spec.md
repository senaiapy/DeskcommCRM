# tags Specification

## Purpose
Tags are free-text labels stored as `text[]` columns (no tag table) on `contacts.tags`, `crm_leads.tags` and `conversations.tags`, each with a GIN index, plus an organization vocabulary kept in `organizations.settings.tags` (curated entries with optional `cor`/`descricao`) and `organizations.settings.canonical_conversation_tags`. Entry points: page `app/app/settings/tags` (vocabulary management); API routes `app/api/v1/tags/vocabulario` (GET, POST), `app/api/v1/tags/cores` (GET), `app/api/v1/contact-tags` (GET) and `app/api/v1/conversation-tags` (GET); tag writes on contacts go through `PATCH /api/v1/contacts/{id}`. Logic: `lib/contacts/tag-normalizada.ts`, `lib/tags/cor-da-etiqueta.ts`, `lib/schemas/tags.ts`; database functions `fn_vocabulario_de_tags`, `fn_vocabulario_de_tags_operar` and `fn_tags_de_conversa_em_uso`.

## Requirements

### Requirement: Tag storage as indexed text arrays
Tags SHALL be stored as `text[] NOT NULL DEFAULT '{}'` on `contacts`, `crm_leads` and `conversations`, indexed by `idx_contacts_tags_gin`, `idx_crm_leads_tags_gin` and `idx_conversations_tags_gin`.

#### Scenario: Contact without tags
- **WHEN** a contact is created without `tags`
- **THEN** `contacts.tags` is the empty array `{}`

#### Scenario: Filter by tag uses containment
- **WHEN** `GET /api/v1/contacts?tag=vip&tag=novo` is called (default AND mode)
- **THEN** the query filters with `tags @> {vip,novo}` and only contacts holding both tags are returned

#### Scenario: OR mode
- **WHEN** `GET /api/v1/contacts?tag=vip&tag=novo&modo=ou` is called
- **THEN** contacts holding either tag are returned

### Requirement: Contact tag normalization
Contact schemas (`contactCreateSchema`, `contactPatchSchema`, `contactListQuerySchema`) SHALL normalize every tag with `normalizarTag` (trim, lowercase, cut at 40 characters) and `normalizarTags` SHALL drop empty values and duplicates while preserving first-appearance order.

#### Scenario: Mixed-case duplicates collapse
- **WHEN** a contact is saved with `tags = ["VIP", " vip ", "Novo"]`
- **THEN** the stored array is `["vip", "novo"]`

### Requirement: Tag vocabulary read
`GET /api/v1/tags/vocabulario` SHALL require `requireRole("manager")` and no pending MFA, and return one row per tag from `fn_vocabulario_de_tags(p_org)` with `tag`, `uso_em_contatos`, `uso_em_leads`, `uso_em_conversas`, `em_regras`, `cor`, `descricao` and `no_vocabulario`, plus `meta.total` and `meta.em_regras`.

#### Scenario: MFA pending
- **WHEN** a manager whose session owes the second factor calls the route
- **THEN** the response is 403 with code `mfa_required`

#### Scenario: Agent denied
- **WHEN** a user with role `agent` calls the route
- **THEN** the request is rejected by `requireRole("manager")`

### Requirement: Atomic vocabulary operations
`POST /api/v1/tags/vocabulario` SHALL require `requireSupportWrite()`, `requireRole("manager")` and no pending MFA, validate `acao` in `renomear | juntar | excluir | definir_cor`, and execute a single call to `fn_vocabulario_de_tags_operar(p_org, p_acao, p_tag, p_destino, p_cor)`.

#### Scenario: Rename rewrites every place the tag lives
- **WHEN** a manager renames tag `cliente novo` to `novo-cliente`
- **THEN** in one transaction the tag is replaced in `contacts.tags`, `crm_leads.tags`, `conversations.tags`, the `add_tag` actions of `automation_rules`, `organizations.settings.tags` and `organizations.settings.canonical_conversation_tags`, and an audit row `tag_vocabulary.changed` is written

#### Scenario: Invalid input
- **WHEN** the body fails `vocabularioDeTagsSchema` (e.g. `cor` not a hex colour or tag longer than 60 characters)
- **THEN** the response is 422 `validation_failed`

#### Scenario: Function refuses the role
- **WHEN** `fn_vocabulario_de_tags_operar` raises SQLSTATE `42501` (`insufficient_role`)
- **THEN** the response is 403 `forbidden`

### Requirement: Tag colours readable by every member
`GET /api/v1/tags/cores` SHALL require only `requireRole("viewer")`, without MFA, and return the `{ tag, cor }` pairs derived by `etiquetasComCor` from `organizations.settings` of the active organization.

#### Scenario: Viewer reads colours
- **WHEN** a viewer calls `GET /api/v1/tags/cores`
- **THEN** the response is 200 with the coloured tags and `meta.total`

### Requirement: Contact tag suggestions
`GET /api/v1/contact-tags` SHALL require `requireRole("viewer")` and return the distinct normalized tags found in the 1000 most recently updated tagged contacts of the organization, sorted with `pt-BR` collation and capped at 200 entries.

#### Scenario: Read failure is not an empty list
- **WHEN** the contacts query fails
- **THEN** the response is 500 `internal_error` and the cause is logged with the `requestId`

### Requirement: Conversation tag vocabulary
`GET /api/v1/conversation-tags` SHALL require `requireRole("viewer")` and return the sorted union of `organizations.settings.canonical_conversation_tags` and the tags in use returned by `fn_tags_de_conversa_em_uso(p_org)`.

#### Scenario: Tag in use but not curated
- **WHEN** a conversation carries tag `retorno` that is not in `canonical_conversation_tags`
- **THEN** `retorno` appears in the response

#### Scenario: In-use read fails
- **WHEN** `fn_tags_de_conversa_em_uso` returns an error
- **THEN** the response is 500 `internal_error` instead of the curated list alone

### Requirement: Tags settings page gate
The page `app/app/settings/tags` SHALL redirect to `/403` when the active organization role ranks below `manager`, with no platform-admin bypass.

#### Scenario: Agent opens the page
- **WHEN** a user with role `agent` requests `/app/settings/tags`
- **THEN** the response redirects to `/403`
