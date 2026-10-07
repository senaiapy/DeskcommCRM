## Why

DeskcommCRM's CRM core (contacts, B2B companies and people, pipelines and the Kanban board, leads/deals, tasks and the timeline, tags, proposals, the product catalog, finance, agenda, imports) is described in `CLAUDE.md`, `docs/prd/02`, `docs/prd/04`, `docs/specs/02` and `docs/specs/04`, and part of that prose no longer matches the code. This change writes a retro-spec of what the code does today, under the shared ids of `openspec-capability-map.md`, so the CRM core can be compared with TEMPLATE_CRM, TEMPLATE_CRM_V1 and TEMPLATE_ERP_v1.

## What Changes

- Documentation only: eleven new capability specs. No code, schema, env var or route changes.
- Where the docs and the code disagree, the specs follow the code and `design.md` lists the drift.
- **BREAKING:** none in the product. What breaks is reliance on doc claims the code does not meet, mainly CPF encryption at rest (`encrypt_cpf`/`decrypt_cpf` do not exist), the `owner_strict` visibility setting (it is `visibility_mode`), and Idempotency-Key on every create POST.

## Capabilities

### New Capabilities
- `contacts`: contacts CRUD, list filters, merge, duplicates, CPF hashing, unique phone/email/CPF, timeline, Customer 360 summary.
- `companies-people`: the `crm_b2b` module's companies, people and links, CNPJ normalization and lookup.
- `pipelines-kanban`: pipelines, stages, won/lost stage rules, `midpoint()` positions, board endpoint.
- `leads-deals`: lead create, patch, move, win, lose, clone, bulk actions, visibility, risk watcher, reactivation, scoring.
- `tasks-activities`: `crm_tasks`, task plans, `crm_lead_activities` timeline and its open type vocabulary.
- `tags`: `text[]` tags with GIN indexes on contacts, leads and conversations; tag vocabulary and colors.
- `proposals`: deal proposals with versions, numbering, send, decision and expiry crons.
- `products-catalog`: `catalog_products` with `preco_cents`/`moeda`, import and photos.
- `finance`: comandas, commissions, `financial_entries` (receivables and payables), recurring entries, loyalty, billing report, honorarios.
- `agenda-scheduling`: appointments, types, availability, idempotent booking, reminders, pending expiry, Google Calendar connection as used by the agenda.
- `imports`: B2B spreadsheet import batches and the contacts CSV import.

### Modified Capabilities
- None.

## Impact

- Files added: `openspec/changes/retro-crm-core/**` only.
- No seed id dropped. `receivables-payables` is folded into `finance` (same table, `financial_entries`, split by direction). Google Calendar is described only inside `agenda-scheduling`; a standalone `google-calendar` id is left to the integrations change.
