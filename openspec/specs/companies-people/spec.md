# companies-people Specification

## Purpose
Companies and people are the optional B2B half of the CRM: legal entities (with CNPJ and BrasilAPI enrichment), the people who work at them, and the link between them, with a contact (WhatsApp identity) optionally pointing at a person. The whole area is the installation module `crm_b2b` (flag `MODULO_CRM_B2B` in `platform_config`, `lib/instalacao/modulos.ts`), off by default. Entry points: pages `app/app/companies`, `app/app/companies/[id]`, `app/app/people`, `app/app/people/[id]` (gated by `app/app/companies/layout.tsx`); API routes `app/api/v1/companies` (GET, POST), `app/api/v1/companies/[id]` (GET, PATCH, DELETE), `app/api/v1/companies/lookup` (GET), `app/api/v1/companies/[id]/enrich` (POST), `app/api/v1/people` (GET, POST), `app/api/v1/people/[id]` (GET, PATCH), `app/api/v1/company-people` (POST), `app/api/v1/company-people/[id]` (PATCH) and `app/api/v1/contacts/[id]/person` (PATCH). Logic in `lib/crm-b2b/` (`companies-handler.ts`, `people-handler.ts`, `enrich.ts`, `normalize.ts`, `route-helpers.ts`, `schemas.ts`). Tables: `companies`, `people`, `company_people`, column `contacts.person_id`.

## Requirements

### Requirement: B2B module gate
Every route and page of the B2B area SHALL answer 404 `not_found` (pages: `notFound()`) when `moduloLigado(…, "crm_b2b")` is false, before any role check.

#### Scenario: Module disabled
- **WHEN** `platform_config` does not hold `MODULO_CRM_B2B = 'ligado'` and a user calls `GET /api/v1/companies`
- **THEN** the response is 404 with code `not_found`

#### Scenario: Module flag unreadable
- **WHEN** reading `platform_config` fails
- **THEN** the module is treated as disabled (fail closed) and the routes answer 404

### Requirement: Role gates for B2B routes
B2B routes SHALL require `requireRole("viewer")` for reads, `requireRole("manager")` for creating companies, people and company links, for company deletion, CNPJ lookup and enrichment, and `requireRole("agent")` for PATCH on companies, people, company links and `contacts/[id]/person`, with `requireSupportWrite()` before every mutation.

#### Scenario: Agent cannot create a company
- **WHEN** a user with role `agent` calls `POST /api/v1/companies`
- **THEN** the request is rejected by `requireRole("manager")` and no `companies` row is inserted

#### Scenario: Agent can edit a person
- **WHEN** a user with role `agent` calls `PATCH /api/v1/people/{id}` with a valid body
- **THEN** the person row is updated and returned

### Requirement: Tenant isolation and role floors in RLS
The tables `companies`, `people` and `company_people` SHALL have RLS policies that allow SELECT for members (`fn_user_org_ids()`), INSERT and DELETE only for `fn_role_at_least(organization_id, 'manager')`, and UPDATE for `fn_role_at_least(organization_id, 'agent')`.

#### Scenario: Cross-org read
- **WHEN** a member of organization A selects `companies` of organization B
- **THEN** no row is returned

### Requirement: CNPJ normalization and uniqueness
Company handlers SHALL store `normalized_cnpj` as 14 digits (check `companies_normalized_cnpj_digits`) and reject a second company with the same `normalized_cnpj` in the organization with 409 `conflict` (unique index `companies_org_normalized_cnpj_uidx`).

#### Scenario: Invalid CNPJ
- **WHEN** `POST /api/v1/companies` carries a `cnpj` that `normalizeCnpj` rejects
- **THEN** the response is 422 `validation_failed`

#### Scenario: Duplicate CNPJ
- **WHEN** a company with the same normalized CNPJ already exists in the organization
- **THEN** the response is 409 `conflict` and no row is inserted

#### Scenario: Company created
- **WHEN** a manager posts a valid company
- **THEN** the response is 201, the row has `enrichment_status = 'pending'`, and an audit row `companies.created` is written

### Requirement: BrasilAPI CNPJ lookup and enrichment
`GET /api/v1/companies/lookup?cnpj=` SHALL query BrasilAPI without writing any row and return `cnpj`, `normalized_cnpj`, `already_registered` and `fields`, mapping a not-found CNPJ to 404 and any other BrasilAPI failure to 502.

#### Scenario: Lookup of an already registered CNPJ
- **WHEN** the CNPJ already exists in `companies` of the organization and BrasilAPI answers
- **THEN** the response is 200 with `already_registered = true`

#### Scenario: BrasilAPI refuses
- **WHEN** BrasilAPI answers HTTP 403 or 429
- **THEN** the response is 502 with `details.dica` explaining the refusal

#### Scenario: Manual enrichment
- **WHEN** a manager calls `POST /api/v1/companies/{id}/enrich`
- **THEN** `companies.enrichment_status`, `enriched_at`/`enrichment_error` and `brasilapi_raw` are updated, an audit row `companies.enriched` is written, and the company is returned

### Requirement: Company deletion refuses linked people
`DELETE /api/v1/companies/{id}` SHALL count `company_people` rows for the company and answer 409 `conflict` with `details.linked_people` when any exist, because the FK cascade would otherwise silently delete them.

#### Scenario: Company with linked people
- **WHEN** the company has 2 rows in `company_people`
- **THEN** the response is 409 `conflict` with `details.linked_people = 2` and the company is not deleted

#### Scenario: Company without links
- **WHEN** the company has no linked people
- **THEN** the row is deleted, an audit row `companies.deleted` is written and the response carries `deleted: true`

### Requirement: Company–person link integrity
The `company_people` table SHALL be unique on `(company_id, person_id)` and the trigger `trg_company_people_same_org` SHALL reject a link whose `organization_id` differs from the company's or the person's.

#### Scenario: Duplicate link
- **WHEN** `POST /api/v1/company-people` links a person already linked to the same company
- **THEN** the response is 409 `conflict`

#### Scenario: Cross-org link
- **WHEN** a link is inserted with a company and a person from different organizations
- **THEN** the trigger raises and the API answers 422 `validation_failed`

### Requirement: Contact to person association
`PATCH /api/v1/contacts/{id}/person` SHALL set or clear `contacts.person_id` (body `{ person_id: uuid | null }`), and the trigger `trg_contacts_person_same_org` SHALL reject a person from another organization.

#### Scenario: Unknown person
- **WHEN** `person_id` does not exist in `people` of the organization
- **THEN** the response is 404 `not_found`

#### Scenario: Clearing the association
- **WHEN** the body is `{ "person_id": null }`
- **THEN** `contacts.person_id` becomes null and the contact is returned

### Requirement: People records
`POST /api/v1/people` SHALL insert into `people` with a non-blank `full_name` (check `people_full_name_nao_vazio`) and return 201, optionally creating the `company_people` link in the same call.

#### Scenario: Person created with company
- **WHEN** a manager posts a person with a `company_id`
- **THEN** the response is 201 and a `company_people` row links the new person to the company (an existing identical link is tolerated)

#### Scenario: Person not found
- **WHEN** `GET /api/v1/people/{id}` targets an id not in the organization
- **THEN** the response is 404 `not_found`
