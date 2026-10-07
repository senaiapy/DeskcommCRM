## Purpose
The core cash module: sale tabs (comandas) with items, commissions and loyalty points, manual and recurring financial entries (receivables and payables share `financial_entries` with `direction` in/out and `status` pending/paid, so they are folded in here), the billing report, and the optional legal-fees module (honorarios). Pages `app/app/comandas`, `app/app/faturamento`, `app/app/honorarios`. API `app/api/v1/financeiro/{catalogo/[tipo],comandas,comandas/pendentes,comandas/faturar-lote,comandas/[id],comandas/[id]/itens[/itemId],comandas/[id]/finalizar,comandas/[id]/estornar,lancamentos[/id],fidelidade}`, `app/api/v1/reports/financeiro`, `app/api/v1/honorarios/{contratos,contratos/[id]/parcelas,parcelas/[id]/pagar}`. Logic in `lib/financeiro/*` (catalog schemas, comanda math, `comanda-do-ganho.handler.ts` event consumer). Tables `financial_accounts`, `payment_methods`, `account_plans`, `commission_rules`, `recurring_entries`, `sales`, `sale_items`, `commissions`, `financial_entries`, `loyalty_ledger`, and module tables `honorarios_contratos`, `honorarios_parcelas`. Database functions `fn_proximo_numero_de_comanda`, `fn_finalizar_comanda`, `fn_estornar_comanda`, `fn_saldo_de_fidelidade`, `fn_relatorio_financeiro`, `fn_honorarios_parcela_pagar`. Cron `app/api/v1/cron/recurring-entries`. Money is always integer `_cents` plus a 3-letter `currency`. Online payment collection is out of scope here and belongs to the `payment-gateways` capability.

## ADDED Requirements

### Requirement: Financial catalog with soft deactivation
`/api/v1/financeiro/catalogo/[tipo]` SHALL accept `tipo` in `contas`, `formas_de_pagamento`, `planos_de_conta`, `regras_de_comissao`, `recorrencias` (tables `financial_accounts`, `payment_methods`, `account_plans`, `commission_rules`, `recurring_entries`), allow `viewer` to GET, require `manager` for POST/PATCH/DELETE, and implement DELETE as `is_active = false`.

#### Scenario: Unknown catalog type
- **WHEN** a request targets `/api/v1/financeiro/catalogo/clientes`
- **THEN** the response is 404 `not_found`

#### Scenario: Deleting an account
- **WHEN** a manager sends DELETE with `{id}` of an account of the organization
- **THEN** the row stays in `financial_accounts` with `is_active = false` and audit action `financeiro.catalogo_inativado` is written

#### Scenario: Duplicate name
- **WHEN** POST creates an entity whose name already exists in the organization
- **THEN** the response is 409 `conflict`

### Requirement: Opening a comanda
`POST /api/v1/financeiro/comandas` SHALL require role `agent`, allocate `sales.number` from `fn_proximo_numero_de_comanda` per organization, set `currency` from the organization currency (never the body) and insert a `sales` row with `status = 'open'`.

#### Scenario: Appointment already has a comanda
- **WHEN** the body `appointment_id` already has a non-cancelled comanda
- **THEN** the response is 200 with the existing `{id, number, status}` and `ja_existia: true`

#### Scenario: Unique race
- **WHEN** the insert fails with SQLSTATE 23505
- **THEN** the response is 409 `conflict`

### Requirement: Items and commission frozen at insertion
`POST /api/v1/financeiro/comandas/[id]/itens` SHALL accept items only on a comanda with `status = 'open'` and store the commission percentage resolved from active `commission_rules` in `sale_items.commission_percent` at insertion time.

#### Scenario: Item on a finalized comanda
- **WHEN** an item is added to a comanda whose status is not `open`
- **THEN** the response is 409 `conflict`

### Requirement: Atomic and idempotent finalization
`POST /api/v1/financeiro/comandas/[id]/finalizar` SHALL require role `agent` and call `fn_finalizar_comanda`, which in one transaction marks the sale `finalized`, inserts `commissions` per item, inserts a `financial_entries` row with `origin = 'sale'` into the payment method's account, inserts loyalty points, and sets the linked `calendar_appointments.status` to `completed`.

#### Scenario: Repeated finalization
- **WHEN** finalize is called on an already finalized comanda
- **THEN** the response is 200 with `ja_finalizada: true`, no new entries are written and no audit row is added

#### Scenario: Cancelled comanda
- **WHEN** finalize is called on a cancelled comanda
- **THEN** the response is 409 `conflict`

#### Scenario: Payment method without account
- **WHEN** the chosen payment method has no destination account
- **THEN** the response is 422 `validation_failed`

### Requirement: Cancel open, reverse finalized
`PATCH /api/v1/financeiro/comandas/[id]` SHALL change discount, notes or cancel (`status = 'cancelled'`) only while `status = 'open'`, and `POST /api/v1/financeiro/comandas/[id]/estornar` SHALL require role `manager` and, through `fn_estornar_comanda`, only for a `finalized` sale, set `reversed_at`, insert paid `financial_entries` with `origin = 'reversal'` and `reverses_entry_id`, mark `commissions.status = 'reversed'` and insert negative `loyalty_ledger` points.

#### Scenario: Cancelling a finalized comanda
- **WHEN** PATCH with `cancel: true` targets a finalized comanda
- **THEN** the response is 409 `conflict`

#### Scenario: Reversing an open comanda
- **WHEN** estornar is called on an open comanda
- **THEN** the response is 409 `conflict`

#### Scenario: Agent tries to reverse
- **WHEN** a user with role `agent` calls estornar
- **THEN** the response is 403

### Requirement: Manual entries (receivables and payables)
`POST /api/v1/financeiro/lancamentos` SHALL require role `agent` and insert a `financial_entries` row with `origin = 'manual'`, positive `amount_cents`, `direction` in (`in`,`out`) and `status` in (`pending`,`paid`), setting `paid_at` only when the status is `paid`.

#### Scenario: Marking as paid twice
- **WHEN** `PATCH /api/v1/financeiro/lancamentos/[id]` marks an already paid entry
- **THEN** the response is 200 with `ja_pago: true`

#### Scenario: Deleting a paid or non-manual entry
- **WHEN** DELETE targets an entry with `status = 'paid'` or `origin <> 'manual'`
- **THEN** the response is 409 `conflict` and the row remains

### Requirement: Paid entries are immutable in the database
The trigger `trg_financial_entries_imutavel` SHALL reject any update that changes `amount_cents`, `account_id`, `direction` or `entry_date` of a `financial_entries` row whose `paid_at` is not null, raising `lancamento_pago_imutavel` with SQLSTATE 42501.

#### Scenario: Editing a paid amount directly
- **WHEN** an UPDATE changes `amount_cents` on a paid entry, bypassing the API
- **THEN** Postgres raises `lancamento_pago_imutavel`

### Requirement: Recurring entries generated daily as pending
`/api/v1/cron/recurring-entries` SHALL, when authorized by `INTERNAL_CRON_SECRET` or `INTERNAL_SECRET`, insert for each active `recurring_entries` template a `financial_entries` row for the current month with `status = 'pending'`, `origin = 'recurring'` and `recurring_entry_id`, clamping `day_of_month` to the last day of the month, relying on the unique `(recurring_entry_id, entry_date)` index for idempotency.

#### Scenario: Second run on the same day
- **WHEN** the cron runs twice in the same month
- **THEN** the second insert hits SQLSTATE 23505, is counted as already existing, and no duplicate entry is created

#### Scenario: Missing cron secret
- **WHEN** the cron is called without a valid secret
- **THEN** the response is 403 `forbidden`

### Requirement: Loyalty ledger
`/api/v1/financeiro/fidelidade` SHALL return a contact's balance from `fn_saldo_de_fidelidade` plus the last 100 `loyalty_ledger` rows to `viewer`, and on POST (role `agent`) insert a signed, non-zero `points` movement with a `reason`, with no guard against a negative balance.

#### Scenario: Balance without contact
- **WHEN** GET is called without `contact_id`
- **THEN** the response is 422 `validation_failed`

#### Scenario: Zero points
- **WHEN** POST sends `points: 0`
- **THEN** the response is 422 `validation_failed`

### Requirement: Billing report aggregated in the database
`GET /api/v1/reports/financeiro` SHALL require role `viewer`, accept a period of at most 400 days, and return the aggregation computed by `fn_relatorio_financeiro`.

#### Scenario: Period too long
- **WHEN** the requested period exceeds 400 days or starts after it ends
- **THEN** the response is 422 `validation_failed`

### Requirement: Comanda opened when a deal is won (opt-in)
The `lead.won` event consumer `comandaDoGanhoHandler` SHALL open a comanda for the won lead only when the lead's pipeline has `crm_pipelines.settings.comanda_no_ganho = true`, reading the organization from the `event_log` row and deduplicating through `crm_lead_links`.

#### Scenario: Switch off
- **WHEN** a lead is won in a pipeline without `comanda_no_ganho = true`
- **THEN** the handler returns `skipped` with reason `comanda_no_ganho_desligada` and no `sales` row is created

### Requirement: Legal-fees module (honorarios)
The honorarios routes SHALL return 409 `module_not_installed` while the module tables do not exist (SQLSTATE 42P01), require `manager` to create contracts and installments, and `POST /api/v1/honorarios/parcelas/[id]/pagar` SHALL pay an installment atomically via `fn_honorarios_parcela_pagar`, inserting a paid `financial_entries` row (`direction = 'in'`, `origin = 'manual'`) and setting `honorarios_parcelas.status = 'pago'` with `financial_entry_id`, honoring an optional UUID `Idempotency-Key`.

#### Scenario: Paying twice
- **WHEN** an installment already `pago` is paid again without an idempotency key
- **THEN** the response is 422 `validation_failed`

#### Scenario: Retry with the same key
- **WHEN** the same request is repeated with the same `Idempotency-Key`
- **THEN** the stored receipt is returned and no second financial entry is created

#### Scenario: Same key, different body
- **WHEN** an `Idempotency-Key` is reused for another installment
- **THEN** the response is 409 `idempotency_conflict`
