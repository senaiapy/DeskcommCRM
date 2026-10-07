# imports Specification

## Purpose
Imports bring spreadsheets into the CRM through two independent paths. (1) The B2B importer (part of the optional `crm_b2b` module): `POST /api/v1/imports` accepts CSV or XLSX, maps columns to companies, people and contacts, and records every batch and row in `import_batches` / `import_rows`; `GET /api/v1/imports` and `GET /api/v1/imports/{id}` read them; pages `app/app/imports` and `app/app/imports/[id]` (layout reuses the `app/app/companies` module gate). Logic in `lib/crm-b2b/spreadsheet.ts` (parsing, limits, column mapping) and `lib/crm-b2b/import-process.ts` (row processing and dedupe). (2) The contacts CSV importer: `POST /api/v1/contacts/import`, called from the contacts screen through `hooks/contacts/useImportContacts.ts`, with parsing and header aliases in `lib/contacts/csv.ts`; it writes only `contacts` and returns a per-line summary without a batch table.

## Requirements

### Requirement: B2B import authorization and module gate
`POST /api/v1/imports` SHALL return 404 `not_found` when the `crm_b2b` module is off, and otherwise require `requireSupportWrite()` and `requireRole("manager")`, while `GET /api/v1/imports` and `GET /api/v1/imports/{id}` require `requireRole("viewer")`.

#### Scenario: Agent upload refused
- **WHEN** a user with role `agent` posts a file to `/api/v1/imports`
- **THEN** the request is rejected by `requireRole("manager")` and no `import_batches` row is created

#### Scenario: Module off
- **WHEN** the module `crm_b2b` is disabled
- **THEN** every `/api/v1/imports` route answers 404 `not_found`

### Requirement: B2B upload format and limits
`POST /api/v1/imports` SHALL accept a multipart field `file` named `.csv` or `.xlsx`, at most `IMPORT_MAX_BYTES` (2 MiB) and `IMPORT_MAX_DATA_ROWS` (2000 data rows), answering 422 `validation_failed` otherwise.

#### Scenario: Missing file field
- **WHEN** the multipart body has no `file`
- **THEN** the response is 422 `validation_failed`

#### Scenario: Unsupported extension
- **WHEN** the file is named `dados.pdf`
- **THEN** the response is 422 `validation_failed`

#### Scenario: Too many rows
- **WHEN** the spreadsheet has 2001 data rows
- **THEN** the response is 422 `validation_failed` and no batch is created

### Requirement: Column mapping
The B2B importer SHALL use the JSON `mapping` form field (validated by `importColumnMappingSchema`) when present and otherwise `suggestColumnMapping(headers)`, which matches header aliases to the fields `company_name`, `legal_name`, `trade_name`, `cnpj`, `person_name`, `job_title`, `phone` and `email`.

#### Scenario: Automatic mapping
- **WHEN** the file has headers `Empresa`, `Telefone`, `E-mail` and no `mapping` field
- **THEN** the batch `column_mapping` is the suggested mapping and the response echoes `headers` and `suggested_mapping`

### Requirement: Batch and row bookkeeping
The B2B importer SHALL insert one `import_batches` row (`kind = 'companies_people'`, `status = 'pending'`, then `processing`, then `completed`) and one `import_rows` row per data line (`row_number` = line number counting the header as 1, unique per batch by `import_rows_batch_row_uidx`), and return 201 with `batch_id`, `processed_rows`, `successful_rows`, `failed_rows` and `conflict_rows`.

#### Scenario: Batch completes
- **WHEN** a manager uploads a valid 10-row file
- **THEN** the response status is 201, `import_batches.status` is `completed` with `completed_at` set, `processed_rows = successful_rows + failed_rows + conflict_rows = 10`, and an audit row `imports.companies_people` is written

#### Scenario: Row status recorded
- **WHEN** a row has a phone that cannot be normalized
- **THEN** its `import_rows.status` is `failed` with `error = 'Telefone inválido.'` and the remaining rows are still processed

### Requirement: Dedupe of companies, people and contacts in the B2B importer
`processCompaniesPeopleImport` SHALL reuse an existing company of the organization with the same `normalized_cnpj`, reuse a person already created in the same batch for the same company and normalized name, and reuse a live contact (`is_merged_into IS NULL`) whose `phone_number` matches any `phoneLookupVariants` of the row phone.

#### Scenario: Phone owned by another person
- **WHEN** the row phone belongs to a contact whose `person_id` points to a different person than the row's person
- **THEN** the row is recorded with `status = 'conflict'` and `error = 'Telefone já vinculado a outra pessoa.'`

#### Scenario: New contact created
- **WHEN** no live contact matches the row phone
- **THEN** a `contacts` row is inserted with `source = 'import_csv'` and `source_metadata.import_batch_id` set to the batch id

#### Scenario: Concurrent insert wins
- **WHEN** the contact insert fails with `23505`
- **THEN** the row is recorded as `conflict`, not `failed`

### Requirement: Import batch read
`GET /api/v1/imports/{id}` SHALL return the batch of the active organization and up to 500 of its `import_rows` ordered by `row_number`, optionally filtered by `?status=`.

#### Scenario: Unknown batch
- **WHEN** the id does not belong to the active organization
- **THEN** the response is 404 `not_found`

#### Scenario: Batch list
- **WHEN** a viewer calls `GET /api/v1/imports`
- **THEN** the 50 most recent batches of the organization are returned ordered by `created_at` desc

### Requirement: Import tables are tenant-isolated with manager writes
The tables `import_batches` and `import_rows` SHALL have RLS allowing SELECT to organization members and INSERT, UPDATE and DELETE only when `fn_role_at_least(organization_id, 'manager')`.

#### Scenario: Agent writes directly
- **WHEN** a session with role `agent` inserts into `import_batches`
- **THEN** the RLS check rejects the row

### Requirement: Contacts CSV import
`POST /api/v1/contacts/import` SHALL require `requireSupportWrite()` and `requireRole("agent")`, accept only a `.csv` (or `text/csv` / `application/vnd.ms-excel`) multipart `file` up to `CSV_MAX_BYTES` (5 MiB) and `CSV_MAX_DATA_ROWS` (500 data rows), and insert contacts one row at a time with `source = 'import_csv'`.

#### Scenario: File too large
- **WHEN** the CSV exceeds 5 MiB
- **THEN** the response is 413 with code `validation_failed`

#### Scenario: XLSX refused
- **WHEN** the file is `contatos.xlsx`
- **THEN** the response is 422 `validation_failed` asking to export as CSV UTF-8

#### Scenario: Too many lines
- **WHEN** the CSV has 501 data rows
- **THEN** the response is 422 `validation_failed`

### Requirement: Per-line outcome of the contacts CSV import
The contacts CSV import SHALL return 200 with `total_linhas`, `imported`, `skipped_duplicates` and `errors[]` (each `{ linha, motivo }`), counting rows whose phone (any variant) or `email_normalized` already exists in the organization, or whose insert fails with `23505`, as `skipped_duplicates` instead of errors.

#### Scenario: Mixed file
- **WHEN** a CSV has one valid new contact, one contact whose phone already exists and one line with an invalid document
- **THEN** the response has `imported = 1`, `skipped_duplicates = 1`, one entry in `errors` with its line number, a `contact.created` event for the new contact, and an audit row `contacts.imported`

#### Scenario: Invalid header
- **WHEN** the header row cannot be mapped by `mapHeader`
- **THEN** the response is 422 `validation_failed` with `details.header`
