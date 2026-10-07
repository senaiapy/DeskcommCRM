## Purpose
Legal-fee contracts (fixed, success-based or mixed) and their installment calendar for law firms, the first
official module installed through `installable-modules`. Its tables `honorarios_contratos` and
`honorarios_parcelas` exist only after `fn_honorarios_provisionar()` runs; paying an installment writes a
`financial_entries` row in the core cash ledger covered by `finance`. Overlaps `installable-modules` (install
mechanism), `finance` (cash entries), `leads-deals` (optional lead link) and `navigation-shell` (module-gated menu
entry).

## ADDED Requirements

### Requirement: Tables are created only by the provisioner
`public.fn_honorarios_provisionar()` SHALL create, idempotently, `honorarios_contratos` (`organization_id`, optional `lead_id`, `modelo` in `fixo|exito|misto`, `valor_fixo_cents > 0`, `percentual_exito` in (0,100], `repasse_advogado_pct` in [0,100]) and `honorarios_parcelas` (`contrato_id`, `numero > 0`, `vencimento`, `valor_cents > 0`, `financial_entry_id`, `status` in `pendente|pago|atrasado` default `pendente`), with RLS enabled and all privileges revoked from `anon`.

#### Scenario: Module not installed
- **WHEN** an installation never ran `fn_modulo_instalar('honorarios', ...)`
- **THEN** neither `honorarios_contratos` nor `honorarios_parcelas` exists

### Requirement: Routes report a missing module explicitly
`GET|POST /api/v1/honorarios/contratos`, `GET|POST /api/v1/honorarios/contratos/{id}/parcelas` and `POST /api/v1/honorarios/parcelas/{id}/pagar` SHALL translate PostgreSQL error `42P01` (undefined table) into 409 `module_not_installed` telling the user to ask the installation administrator to install the module.

#### Scenario: Paying before install
- **WHEN** `POST /api/v1/honorarios/parcelas/{id}/pagar` runs on an installation without the module and `fn_honorarios_parcela_pagar` raises `42P01`
- **THEN** the response is 409 `module_not_installed`

### Requirement: Reading needs viewer, writing needs manager
The listing routes SHALL require `requireRole("viewer")` and rely on RLS (`organization_id in fn_user_org_ids()`) for tenancy; the create and pay routes SHALL call `requireSupportWrite` and `requireRole("manager")`, and the RLS insert/update policies SHALL also require `fn_role_at_least(organization_id, 'manager')`.

#### Scenario: Agent creates a contract
- **WHEN** a user with role `agent` posts to `/api/v1/honorarios/contratos`
- **THEN** the response is 403 and no row is inserted

### Requirement: Contract model requires its matching value
`POST /api/v1/honorarios/contratos` SHALL insert with `organization_id` from the active session and SHALL return 422 `validation_failed` unless `fixo` carries `valor_fixo_cents`, `exito` carries `percentual_exito` and `misto` carries both, or when `lead_id` is not a `crm_leads` row of the active organization; success audits `honorarios.contrato_criado`.

#### Scenario: Success model without percentage
- **WHEN** the body is `{"modelo": "exito"}`
- **THEN** the response is 422 `validation_failed`

#### Scenario: Lead from another tenant
- **WHEN** `lead_id` belongs to another organization
- **THEN** the response is 422 `validation_failed` with "Lead inválido para esta organização."

### Requirement: Installment creation is unique per number
`POST /api/v1/honorarios/contratos/{id}/parcelas` SHALL take `{numero, vencimento (YYYY-MM-DD), valor_cents}`, insert a `pendente` installment only for a contract of the same organization (RLS), answer 422 `validation_failed` for a foreign contract (23503/42501) or a duplicate `numero` (23505), and audit `honorarios.parcela_criada`.

#### Scenario: Duplicate number
- **WHEN** installment number 2 already exists on the contract
- **THEN** the response is 422 `validation_failed` with "Já existe uma parcela com este número."

### Requirement: Paying an installment is atomic
`POST /api/v1/honorarios/parcelas/{id}/pagar` SHALL call `fn_honorarios_parcela_pagar(p_org, p_parcela, p_account_id, p_account_plan_id)`, which locks the installment `for update`, rejects a non-manager (403 `forbidden_role`), a missing installment (404 `not_found`), an installment already `pago` (422 `validation_failed`) and an account or plan not active in the same organization (422 `validation_failed`), inserts one `financial_entries` row with `direction = 'in'` and `status = 'paid'`, and sets the installment to `pago` with `financial_entry_id`.

#### Scenario: Two concurrent payments
- **WHEN** two requests pay the same pending installment at the same time
- **THEN** exactly one `financial_entries` row is created and the other request gets 422 "Esta parcela já está paga."

#### Scenario: Account of another organization
- **WHEN** `account_id` belongs to another organization
- **THEN** the response is 422 `validation_failed` and no entry is written

### Requirement: Optional idempotency on payment
When `POST /api/v1/honorarios/parcelas/{id}/pagar` receives an `Idempotency-Key`, it SHALL require a UUID (400 `validation_error` otherwise), replay the stored response for the same body, and return 409 `idempotency_conflict` for a different body or 409 `idempotency_in_progress` while the first call runs.

#### Scenario: Retry after timeout
- **WHEN** the client repeats the payment with the same key and body
- **THEN** the original response is returned and no second entry is created

### Requirement: Paid installments are immutable
RLS SHALL forbid updating or deleting a `honorarios_parcelas` row with `status = 'pago'`, forbid inserting or updating a row as `pago` or with `financial_entry_id` set, and forbid deleting a contract that has any paid installment.

#### Scenario: Manager edits a paid installment
- **WHEN** a manager updates `valor_cents` of a `pago` installment through the REST API
- **THEN** no row is changed

### Requirement: Menu entry exists only when installed
The navigation entry `/app/honorarios` in `lib/navigation/catalogo.ts` SHALL declare `modulo: "honorarios"` with `minRole: "viewer"`, and menus that pass the installed-module list SHALL show it only when `modulos_instalados` has `modulo = 'honorarios'` with `estado = 'ativo'`, treating a failed read as no module installed.

#### Scenario: Installation without the module
- **WHEN** a user opens the app on an installation without the module, or with it `suspenso`
- **THEN** the sidebar has no "Honorários" entry
