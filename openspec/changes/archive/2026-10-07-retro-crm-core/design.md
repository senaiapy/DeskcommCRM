## Context

Retro-spec of the CRM core of DeskcommCRM as of 2026-10-07. Seed material: `CLAUDE.md` (Modelagem), `docs/prd/02-prd-customer-360.md`, `docs/prd/04-prd-pipeline-attendance.md`, `docs/specs/02-spec-customer-360.md`, `docs/specs/04-spec-pipeline-attendance.md`, `docs/specs/17-spec-conversa-vira-lead.md`. The code is the authority.

## How the evidence was gathered

- Each capability was traced from `app/app/**` pages and `app/api/v1/**` routes (plus related crons in `app/api/v1/cron/*`) into `lib/**` and into `supabase/baseline.sql`, including the idempotent appendix where constraints are redefined.
- Requirements name the concrete route, table, column, constraint or error code that was read. Claims that could not be backed were dropped: an `owner_strict` setting, a public booking page, a public proposal acceptance link, a task reminder cron, a seat or saved-views feature.
- Specs were written by helper agents per domain and spot-checked (for example, the absence of `encrypt_cpf` anywhere under `supabase/` was confirmed by grep). No tests were run.

## Inconsistencies found (docs vs code)

1. **CPF at rest.** `docs/specs/02` describes `cpf_encrypted` written by `encrypt_cpf()` and read by `decrypt_cpf()`. Neither function exists in `supabase/`; `CPF_ENCRYPTION_KEY` is required by `lib/env.ts` but not used for CPF.
2. **Bearer on contacts.** Only `GET /api/v1/contacts` accepts a `dsk_` bearer (scope `mcp:read`); `POST` is session-only.
3. **Lead visibility.** There is no `owner_strict`; visibility is `organizations.settings.visibility_mode` (`all`, `own_and_unassigned` default, `own`) applied by `fn_can_view_lead`.
4. **Idempotency-Key.** `POST /api/v1/leads`, `/tasks`, `/pipelines`, `/contacts`, `/proposals` do not read it; `agenda/agendamentos` and `honorarios/.../pagar` do.
5. **Products.** The table is `catalog_products` with `preco_cents` and `moeda`, not `products` with `price_cents`/`currency`.
6. **Board.** `GET /pipelines/{id}/board` says "open leads" but returns won and lost leads too.
7. **Error codes outside the catalog.** `proposal_context_stale`, `agenda_listagem_janela_invalida`, `agenda_listagem_cursor_invalido`, `agenda_ainda_nao_aconteceu`, `appointment_do_colega`, `internal` are emitted but not in `lib/api/errors.ts`; `validation_error` and `validation_failed` both appear; `bulk_too_large` is unreachable because Zod caps `lead_ids` at 50 first.
8. **Schema read in two places.** `crm_proposals` and other tables are created in the dump part of `baseline.sql` with older NOT NULLs and status lists, then widened in the appendix.
9. **Imports.** The B2B import answers 201 and records `import_batches`; the contacts CSV import answers 200 and records no batch, although the table allows `kind = 'contacts'`.
10. **Agenda.** `calendar_appointments.source` allows `public_page`, but no public booking route exists; `cron/agenda-google-sync` answers an unauthorized call with a raw 401 without `X-Request-Id`, unlike other crons.

## Declared debts

- **Defect:** creating or importing a contact with a CPF writes `cpf_hash` without `cpf_encrypted`, violating `contacts_cpf_consistency`; `POST /api/v1/contacts` then answers 500 and the CSV import reports the line as an error (`lib/contacts/cpf.ts:4-7` admits the RPC is not provisioned). `cpf_decrypted` is always null.
- `POST /api/v1/financeiro/lancamentos` does not set `currency`, so manual entries default to `BRL`; `honorarios_*` tables have no `currency` column.
- Finance routes have no module/capability gate; several update by `id` relying on RLS only.
- Pipeline creation, won/lost stage re-marking and lead clone are multi-step writes without a transaction; `midpoint()` yields NaN on tied neighbours and there is no rebalance.
- No task due/overdue reminder cron; overdue is computed on read. `crm_stages.expected_duration_hours` has no CHECK.
- B2B import never marks a batch `failed` and runs synchronously in the request; `contact-tags` reads at most 1000 contacts and 200 tags; no DELETE for `company_people` or `people`.
- The timeline's `contact_id … and lead_id is null` branch can match nothing while `crm_lead_activities.lead_id` is NOT NULL.

## Goals / Non-Goals

- Goal: an accurate, diffable CRM-core baseline per canonical id.
- Non-goal: fixing any item above; the CPF defect deserves its own change first.
