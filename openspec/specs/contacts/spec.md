# contacts Specification

## Purpose
Contacts are the tenant-scoped person records of the CRM (one row per customer identity, keyed by phone, e-mail and CPF hash). Entry points: pages `app/app/contacts` (list) and `app/app/contacts/[id]` (contact dossier / customer 360); API routes `app/api/v1/contacts` (GET list, POST create), `app/api/v1/contacts/[id]` (GET, PATCH, DELETE), `app/api/v1/contacts/[id]/timeline` (GET), `app/api/v1/contacts/[id]/crm-summary` (GET), `app/api/v1/contacts/[id]/vinculos` (GET), `app/api/v1/contacts/[id]/personal` (POST, DELETE), `app/api/v1/contacts/duplicates` (GET) and `app/api/v1/contacts/merge` (POST). Logic lives in `app/api/v1/contacts/_handler.ts` (shared with MCP tools), `lib/contacts/*` (`cpf.ts`, `duplicados.ts`, `tag-normalizada.ts`) and `lib/channels/phone-variants.ts`. Data: table `contacts` (RLS through `fn_user_org_ids()`), RPCs `fn_mesclar_contatos` and `fn_apagar_contato_com_historico`.

## Requirements

### Requirement: Dual authentication on contact listing
`GET /api/v1/contacts` SHALL accept either a session cookie checked by `requireRole("viewer")` or an `Authorization: Bearer dsk_…` token validated by `validateBearerToken`, taking `organization_id` from the token row and never from the request.

#### Scenario: Bearer token without read scope
- **WHEN** a valid `dsk_` token lacking the `mcp:read` scope calls `GET /api/v1/contacts`
- **THEN** the response is 403 with error code `forbidden_role`

#### Scenario: Invalid bearer token
- **WHEN** a missing, revoked or expired `dsk_` token is presented
- **THEN** the response is 401 and no contact row is read

#### Scenario: Create is session-only
- **WHEN** `POST /api/v1/contacts` is called
- **THEN** authorization goes through `requireSupportWrite()` and `requireRole("agent")` (session), with no bearer branch in the handler

### Requirement: Contact list only shows live operational persons
`listContactsHandler` SHALL filter `contacts` by the caller's `organization_id`, `kind = 'person'`, `is_merged_into IS NULL` and `is_personal = false` (or only `is_personal = true` when `?pessoais=true`), paginating with an opaque base64url cursor.

#### Scenario: WhatsApp group placeholders and merge tombstones are hidden
- **WHEN** the organization has a contact with `kind = 'whatsapp_group'` and another with `is_merged_into` set
- **THEN** neither appears in the `GET /api/v1/contacts` response

#### Scenario: Invalid cursor
- **WHEN** `?cursor=` cannot be decoded
- **THEN** the response is 400 with error code `invalid_cursor`

#### Scenario: Search below the floor
- **WHEN** `?search=` (after removing parentheses) fails `buscaValeConsulta`
- **THEN** the response is 200 with an empty list and `has_more = false` without querying the database

#### Scenario: Search by CPF digits
- **WHEN** `?search=` contains exactly 11 digits
- **THEN** the OR filter also matches `cpf_hash` equal to the sha256 of those digits

### Requirement: Contact creation
`POST /api/v1/contacts` SHALL validate the body against `contactCreateSchemaDoPais` (document rule taken from `organizations.country`), store `phone_number` in canonical form via `canonicalPhoneBR`, and return 201 with `{ contact, action: "created" }`.

#### Scenario: Valid contact created
- **WHEN** an agent posts a valid body with `phone_number`
- **THEN** a `contacts` row is inserted with `organization_id` of the active org and `created_by_user_id` of the caller, a `contact.created` event is emitted via `emit_event`, an audit row `contact.created` is written, and the response status is 201

#### Scenario: Invalid body
- **WHEN** the body fails schema validation
- **THEN** the response is 422 with error code `validation_error` and `details.fieldErrors`

### Requirement: Duplicate phone on create returns the existing contact
`createContactHandler` SHALL map a `23505` insert error whose phone resolves to a live contact of the same organization into HTTP 409 `contact_exists` with `details.contact_id`.

#### Scenario: Phone already registered
- **WHEN** a contact is created with a phone already held by a live contact of the organization
- **THEN** the response is 409 with code `contact_exists` and `details.contact_id` equal to the existing contact id

#### Scenario: Other unique conflict
- **WHEN** the insert fails with `23505` on e-mail or CPF and the phone lookup finds no live contact
- **THEN** the response is 500 `internal_error`

### Requirement: Identity uniqueness and format constraints
The `contacts` table SHALL enforce partial unique indexes `uniq_contacts_org_phone (organization_id, phone_number)`, `uniq_contacts_org_email (organization_id, email_normalized)` and `uniq_contacts_org_cpf (organization_id, cpf_hash)`, each restricted to `is_merged_into IS NULL`, plus the E.164 check `contacts_phone_e164_format`.

#### Scenario: E-mail normalization
- **WHEN** a contact is saved with `email = ' Ana@X.com '`
- **THEN** the generated column `email_normalized` holds `ana@x.com` and a second live contact with `ana@x.com` in the same organization is rejected with `23505`

#### Scenario: Merged tombstone releases identity
- **WHEN** a contact has `is_merged_into` set
- **THEN** its phone, e-mail and CPF hash no longer block a new live contact in the same organization

#### Scenario: Non-E.164 phone rejected
- **WHEN** a row is written with `phone_number` not matching `^\+\d{8,15}$`
- **THEN** the database rejects it with the `contacts_phone_e164_format` check

### Requirement: CPF is never stored in plaintext
Contact handlers SHALL store the CPF only as `cpf_hash` (sha256 hex of the 11 normalized digits, `lib/contacts/cpf.ts`) and `cpf_encrypted` (from the `encrypt_cpf` RPC), and the check `contacts_cpf_consistency` SHALL require both columns to be null or non-null together.

#### Scenario: Hash without ciphertext rejected
- **WHEN** a contact is inserted with `cpf_hash` set and `cpf_encrypted` null (e.g. `encrypt_cpf` RPC unavailable)
- **THEN** the insert violates `contacts_cpf_consistency` and the API answers 500 `internal_error`

#### Scenario: Decrypt request below manager
- **WHEN** a user with role below `manager` calls `GET /api/v1/contacts/{id}` with header `X-Decrypt-Purpose`
- **THEN** the response has `cpf_decrypted = null` and `cpf_decrypt_denied = true`

### Requirement: Contact update
`PATCH /api/v1/contacts/{id}` SHALL require `requireSupportWrite()` and `requireRole("agent")`, merge `consent` with the previous value, and audit `contact.updated` with `old_`/`new_` pairs for changed `email`, `phone_number`, `name` and `display_name`.

#### Scenario: Anonymized contact is read-only
- **WHEN** the target contact has `is_anonymized = true`
- **THEN** the response is 403 with code `lgpd_anonymization_irreversible`

#### Scenario: Empty patch
- **WHEN** the body contains no updatable field
- **THEN** the response is 400 with code `invalid_request`

#### Scenario: Tag added
- **WHEN** the patch adds a tag not present before
- **THEN** an event `contact.tag_added` is emitted with `payload.added_tags`

### Requirement: Contact deletion refuses linked records
`DELETE /api/v1/contacts/{id}` SHALL require `requireRole("agent")`, refuse with 409 `state_conflict` when RESTRICT-linked rows exist, and otherwise delete through `fn_apagar_contato_com_historico` returning 204.

#### Scenario: Linked records block deletion
- **WHEN** the contact still has rows in a RESTRICT-linked table
- **THEN** the response is 409 `state_conflict` with `details.vinculos` and `details.por_tabela`, and an audit row `contact.delete_blocked` is written

#### Scenario: Successful deletion
- **WHEN** no RESTRICT link exists and the RPC returns true
- **THEN** the response is 204, a `contact.deleted` event is emitted and an audit row `contact.deleted` is written

### Requirement: Contact merge
`POST /api/v1/contacts/merge` SHALL require `requireRole("manager")` and run `fn_mesclar_contatos(p_organization_id, p_contato_principal, p_contatos_secundarios)` with the session client, mapping its known raise labels to HTTP errors.

#### Scenario: Successful merge
- **WHEN** a manager posts `primary_contact_id` and 1–20 `secondary_contact_ids`
- **THEN** the response is 200 with `contato_id`, `contatos_mesclados`, `repontado` and `nao_repontado`, and an audit row `contact.merged` is written

#### Scenario: Secondary unavailable
- **WHEN** a secondary contact is anonymized or already merged
- **THEN** the response is 404 `not_found`

#### Scenario: Primary inside secondaries
- **WHEN** `primary_contact_id` appears in `secondary_contact_ids`
- **THEN** the request fails schema validation with 422

### Requirement: Duplicate detection
`GET /api/v1/contacts/duplicates` SHALL scan at most 2000 live (`is_merged_into IS NULL`), non-anonymized contacts of the active organization and return duplicate groups with `chave`, `motivos`, `principal_sugerido` and `contatos`.

#### Scenario: Scan truncated
- **WHEN** the organization has more than 2000 live contacts
- **THEN** `meta.varreu_tudo` is false and `meta.contatos_varridos` is 2000

#### Scenario: Unauthenticated
- **WHEN** there is no session
- **THEN** the response is 401 `unauthenticated`

### Requirement: Contact timeline
`GET /api/v1/contacts/{id}/timeline` SHALL return `crm_lead_activities` rows attached either directly to the contact (`contact_id`) or to any of its `crm_leads`, deduplicated, ordered by `performed_at` desc then `id` desc, with `limit` clamped to 1..100 (default 50).

#### Scenario: Contact not visible
- **WHEN** the contact id is not visible to the session under RLS
- **THEN** the response is 404 `not_found`

#### Scenario: Filter by type
- **WHEN** `?type=order_created&type=message_inbound` is passed
- **THEN** only activities of those types are returned

### Requirement: Personal contact flag
`POST /api/v1/contacts/{id}/personal` and `DELETE /api/v1/contacts/{id}/personal` SHALL require `requireRole("manager")` and set or clear `contacts.is_personal`, without deleting conversations, messages or leads.

#### Scenario: Marking a contact personal
- **WHEN** a manager posts to the personal route for a non-personal contact
- **THEN** `is_personal` becomes true and the response is 200 with `contact` and `effects` (cancelled follow-ups, campaign exits, closed conversations, removed RAG chunks)

#### Scenario: Agent cannot mark
- **WHEN** a user with role `agent` calls the route
- **THEN** the request is rejected by `requireRole` and `is_personal` is unchanged
