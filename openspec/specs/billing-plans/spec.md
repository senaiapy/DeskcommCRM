# billing-plans Specification

## Purpose
Installation-level billing of the organizations served by one installation ("cobrança do revendedor"), shipped switched off. Data and rules live in migration `supabase/migrations/20261007130926_0583_cobranca_planos_e_assinaturas.sql` (tables `public.cobranca_planos` and `public.cobranca_assinaturas`, functions `fn_cobranca_ligada`, `fn_limite_do_plano`, `fn_trava_assentos_do_plano`, `fn_trava_canais_do_plano`, `fn_trial_na_criacao_da_org`, `fn_cobranca_liberar_suspensoes`, and the billing branch of `fn_suspender_organizacao` / `fn_reativar_organizacao`). App code: `lib/cobranca/*` (`limites.ts` translates the PT402 refusals, `faixa.ts` the trial banner, `painel.ts` the read-only panel, `dono.ts`/`uso.ts` the owner routes), `lib/schemas/cobranca-plano.ts`, the installation switch `cobranca` in `lib/instalacao/modulos.ts` (`platform_config.MODULO_COBRANCA`). Owner surfaces: `/admin/cobranca`, `POST /api/v1/admin/cobranca/planos`, `PATCH /api/v1/admin/cobranca/planos/[id]`, `POST|PATCH|DELETE /api/v1/admin/tenants/[id]/assinatura`, `POST /api/v1/admin/tenants/[id]/assinatura/prazo`, the `plano_id` field of `POST /api/v1/admin/tenants`. Organization surfaces: `/app/settings/billing` and the trial banner in `app/app/layout.tsx`. No payment provider is wired yet. Overlaps `platform-admin-console`, `team-invites`, `whatsapp-waha` and `ai-credentials-budget` (the plan AI ceiling `ia_usd_cents`).

## Requirements

### Requirement: Billing switch is off and cannot be turned on yet
`fn_cobranca_ligada()` SHALL return true only when `platform_config` has the row `chave = 'MODULO_COBRANCA'` with `valor = 'ligado'`, and `updateModuloDaInstalacao()` SHALL refuse turning the module `cobranca` on with `{ ok: false, error: "modulo_ainda_nao_disponivel" }` because it is listed in `MODULOS_AINDA_NAO_LIGAVEIS`.

#### Scenario: Fresh installation
- **WHEN** `platform_config` has no `MODULO_COBRANCA` row
- **THEN** `fn_cobranca_ligada()` returns false and `fn_limite_do_plano` returns null for every organization

#### Scenario: Owner tries to switch billing on
- **WHEN** a platform admin calls `updateModuloDaInstalacao({ modulo: "cobranca", ligado: true })`
- **THEN** the action returns `modulo_ainda_nao_disponivel` and `platform_config` is unchanged

### Requirement: Plans belong to the installation and subscriptions to the organization
`public.cobranca_planos` SHALL have no `organization_id`, RLS enabled with no policy and all privileges revoked from `anon` and `authenticated`, CHECKs `preco_cents >= 500`, `moeda = 'BRL'`, `intervalo in ('mes','ano')`, `trial_dias between 0 and 90`, and the unique partial index `cobranca_planos_um_padrao` allowing one non-archived `padrao_no_cadastro` plan; `public.cobranca_assinaturas` SHALL hold at most one row per `organization_id` with `estado in ('trial','ativa','em_atraso','cancelada')`, readable through policy `tenant_isolation_cobranca_assinaturas_select` only by `fn_role_at_least(organization_id, 'admin')` and writable only by `service_role`; an organization without a row is exempt.

#### Scenario: Second signup-default plan
- **WHEN** a second non-archived plan with `padrao_no_cadastro = true` is inserted
- **THEN** Postgres rejects it with a unique violation on `cobranca_planos_um_padrao`

#### Scenario: Manager reads the subscription
- **WHEN** a `manager` selects `cobranca_assinaturas` through the session client
- **THEN** zero rows are returned

### Requirement: Plan limit lookup
`fn_limite_do_plano(p_org, p_recurso)` SHALL raise SQLSTATE `22023` for a `p_recurso` outside `assentos`, `canais`, `ia_usd_cents`, and SHALL return null when billing is off, when the organization has no `cobranca_assinaturas` row, or when the plan's `max_assentos` / `max_canais` / `teto_ia_usd_cents` is null, otherwise the ceiling of the plan in `plano_id`.

#### Scenario: Exempt organization
- **WHEN** billing is on and an organization has no subscription row
- **THEN** `fn_limite_do_plano(org, 'assentos')` returns null

#### Scenario: Unknown resource
- **WHEN** `fn_limite_do_plano(org, 'tokens')` is called
- **THEN** it raises `recurso_do_plano_invalido` with SQLSTATE `22023`

### Requirement: Seat ceiling enforced by trigger
Trigger `trg_trava_assentos_do_plano` on `public.user_organizations` (before insert or update of `revoked_at`, `provisional_until_handover`, `organization_id`) SHALL raise SQLSTATE `PT402` with message `limite_do_plano:assentos:<limit>` when the other non-revoked, non-provisional members of the organization already reach `fn_limite_do_plano(org, 'assentos')`.

#### Scenario: Organization at the seat ceiling
- **WHEN** a plan has `max_assentos = 2`, the organization has 2 active members, and a third membership row is inserted
- **THEN** the insert fails with SQLSTATE `PT402` and message `limite_do_plano:assentos:2`

#### Scenario: Billing off
- **WHEN** billing is off and the same insert runs
- **THEN** the membership is created

### Requirement: Messaging-number ceiling enforced by trigger
Trigger `trg_trava_canais_do_plano` on `public.channel_sessions` SHALL raise SQLSTATE `PT402` with message `limite_do_plano:canais:<limit>` when the organization's other non-archived channels whose `provider <> 'wacalls'` reach `fn_limite_do_plano(org, 'canais')`, and the connect routes `POST /api/v1/channel-sessions`, `POST /api/v1/channels/partner`, `POST /api/v1/channels/graph-partner`, `POST /api/v1/channels/official`, `POST /api/v1/channels/social` and `POST /api/v1/onboarding/whatsapp/session` SHALL translate that refusal through `traduzirLimiteDoPlano` into HTTP 409 `plan_limit_reached` with `details = { recurso: "canais", limite }`.

#### Scenario: Connecting one number too many
- **WHEN** an admin of an organization whose plan has `max_canais = 1` and one active WhatsApp channel connects another number
- **THEN** the response is 409 `plan_limit_reached` with `details.recurso = "canais"` and `details.limite = 1`

#### Scenario: Voice line does not count
- **WHEN** the organization at the same ceiling creates a channel with `provider = 'wacalls'`
- **THEN** the trigger does not refuse it

### Requirement: Owner manages plans
`POST /api/v1/admin/cobranca/planos` and `PATCH /api/v1/admin/cobranca/planos/[id]` SHALL call `requireSupportWrite()` and `requirePlatformAdminEscrita()`, answer 404 `not_found` while billing is off, 400 `validation_failed` for a body outside `novoPlanoSchema` / `edicaoDoPlanoSchema`, 409 `state_conflict` on a second signup-default plan, and on PATCH 409 `plano_com_assinantes` when `preco_cents` or `intervalo` changes while any `cobranca_assinaturas` row points to the plan in `plano_id` or `plano_agendado_id`; they SHALL audit `cobranca.plano_salvo` or `cobranca.plano_arquivado`, with POST answering 201.

#### Scenario: Billing off
- **WHEN** a full-scope platform admin posts a valid plan while `MODULO_COBRANCA` is not `ligado`
- **THEN** the response is 404 `not_found` and no `cobranca_planos` row is inserted

#### Scenario: Repricing a plan with subscribers
- **WHEN** a PATCH changes `preco_cents` of a plan referenced by one subscription
- **THEN** the response is 409 `plano_com_assinantes` with `details.assinantes = 1` and the price is unchanged

### Requirement: Owner assigns, changes and removes an organization's plan
`/api/v1/admin/tenants/[id]/assinatura` SHALL, behind `requireSupportWrite(id)`, `requirePlatformAdminEscrita()` and 404 while billing is off, create a `trial` row with `trial_ate = now + plano.trial_dias` on POST (409 `state_conflict` when a row exists, 422 `plano_invalido` for a missing or archived plan), change `plano_id` on PATCH only during an unexpired trial without provider (409 `pagamento_pendente` for `em_atraso`/`cancelada` or an expired trial, 422 `plano_invalido` for another `intervalo`, 409 `plan_limit_reached` with `details.excedente` when current usage exceeds the new plan), and on DELETE remove a row without provider and reactivate a billing suspension, auditing `cobranca.plano_trocado` and `cobranca.isencao_definida`.

#### Scenario: Assigning a plan to an exempt organization
- **WHEN** the owner posts `{ "plano_id": <active plan with trial_dias 14> }` for an organization without subscription
- **THEN** the response is 201 with `estado = "trial"` and `trial_ate` about 14 days ahead, and an audit `cobranca.plano_trocado` with `quando = "atribuido"` exists

#### Scenario: Downgrade below current usage
- **WHEN** a PATCH during the trial picks a plan whose `max_assentos` is lower than the organization's active members
- **THEN** the response is 409 `plan_limit_reached` with `details.excedente` and `plano_id` is unchanged

### Requirement: Grace period for a subscription
`POST /api/v1/admin/tenants/[id]/assinatura/prazo` SHALL accept `{ ate }` as an ISO datetime between now and 60 days ahead (422 `validation_failed` otherwise), answer 404 `not_found` when billing is off or the organization has no subscription, write `cobranca_assinaturas.prazo_extra_ate`, reactivate a billing suspension, and audit `cobranca.prazo_concedido`.

#### Scenario: Grace beyond 60 days
- **WHEN** the owner posts `ate` 90 days ahead
- **THEN** the response is 422 `validation_failed` and `prazo_extra_ate` is unchanged

### Requirement: Plan chosen when creating a tenant and trial at signup
`POST /api/v1/admin/tenants` SHALL accept an optional `plano_id` and answer 422 `plano_invalido` when billing is off or the plan is missing or archived, and trigger `trg_trial_na_criacao_da_org` SHALL insert a `trial` subscription with the `trial_dias` of the non-archived `padrao_no_cadastro` plan for an organization whose `created_by` is set and is not an active platform admin, only while billing is on and such a plan exists.

#### Scenario: Plan sent while billing is off
- **WHEN** a platform admin creates a tenant with a `plano_id` and billing is off
- **THEN** the response is 422 `plano_invalido` and no organization is created

#### Scenario: Self-service signup with billing off
- **WHEN** a user creates an organization through signup while billing is off
- **THEN** no `cobranca_assinaturas` row is created

### Requirement: Billing suspensions spare exempt organizations and end when billing is switched off
`fn_suspender_organizacao(org, 'cobranca', ...)` SHALL return `{ changed: false, motivo: "org_isenta" }` for an active organization without subscription, and switching the module `cobranca` off in `updateModuloDaInstalacao()` SHALL first run `fn_cobranca_liberar_suspensoes`, which reactivates every organization with `suspended_kind = 'cobranca'`, and audit `cobranca.modulo_desligado` with `metadata.liberadas`.

#### Scenario: Billing suspension of an exempt organization
- **WHEN** `fn_suspender_organizacao` is called with `p_kind = 'cobranca'` for an active organization without subscription
- **THEN** it returns `motivo = "org_isenta"` and `organizations.status` stays `active`

#### Scenario: Release fails
- **WHEN** `fn_cobranca_liberar_suspensoes` errors while the owner switches billing off
- **THEN** the action returns `liberacao_falhou` and `MODULO_COBRANCA` keeps its value

### Requirement: Billing surfaces exist only while billing is on
`/admin/cobranca` SHALL respond `notFound()` and the admin sidebar entry `/admin/cobranca` SHALL be hidden unless the installation module `cobranca` is on, `/app/settings/billing` SHALL render `PainelDaAssinatura` from `lerPainelDaAssinatura()` only while billing is on (otherwise the "Em breve — Fase 2" card), and `app/app/layout.tsx` SHALL show `FaixaDoTesteGratis` to organization admins only when the subscription is `trial` with `trial_ate` at most 7 days ahead.

#### Scenario: Billing off
- **WHEN** a platform admin requests `/admin/cobranca` while the module is off
- **THEN** the response is 404 and the sidebar has no "Cobrança" entry

#### Scenario: Trial ending
- **WHEN** an admin opens `/app` while billing is on and the organization's trial ends in 3 days
- **THEN** the trial banner shows 3 days

#### Scenario: Trial far from the end
- **WHEN** the trial ends in 10 days
- **THEN** no trial banner is shown
