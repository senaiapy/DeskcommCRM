# settings Specification

## Purpose
The general settings surface under `/app/settings` (hub `app/app/settings/page.tsx`) not owned by a more specific capability: personal profile (`/app/settings/profile`, `app/actions/settings/updateProfile.ts`), organization profile and danger zone (`/app/settings/tenant`, `updateTenant.ts`, `apagarDadosOperacionaisDaOrganizacao.ts`), notification sounds and preferences (`/app/settings/notifications`, `GET|POST|DELETE /api/v1/settings/sons`, `updateNotificationPrefs.ts`), outgoing message signature (`GET|PATCH /api/v1/settings/assinatura`), optional features (`/app/settings/recursos`, catalog `lib/recursos-opcionais/catalogo.ts`) and the self-host update panel (`/app/settings/atualizacao`). Other pages under `/app/settings` belong to their own capabilities: security (mfa-policy), api-tokens, marca (white-label-branding), tags, templates, atendimento/routing, automacoes, canal-oficial, conversoes, meta-ads, voip-trunk, billing. Most organization preferences live in `public.organizations` columns or keys of `organizations.settings` (jsonb).

## Requirements

### Requirement: Personal profile in auth metadata
`updateProfile()` SHALL validate `profileSchema` (`full_name` ≤120, `locale` among visible languages or `auto`, `timezone`, optional `avatar_url`), write it to the Supabase Auth user metadata (`auth.updateUser`), and audit `profile.updated`.

#### Scenario: User changes timezone
- **WHEN** a signed-in user saves the profile with `timezone = "America/Bogota"`
- **THEN** `user_metadata.timezone` is updated and an audit row `profile.updated` exists

### Requirement: Organization profile requires company admin
`updateTenant()` SHALL refuse support sessions that cannot write (`forbidden`) and callers failing `podeAdministrarEmpresa()` (role below `admin` and not a full platform admin) with `forbidden_role`, then update `organizations` columns `display_name`, `legal_name`, `cnpj`, `country`, `timezone`, `locale`, `currency`, `media_retention_days` (30–3650), `media_retention_enforced`, `dpo_email` and `privacy_policy_url`, auditing `org.updated`.

#### Scenario: Manager edits company data
- **WHEN** a `manager` submits the organization form
- **THEN** the action returns `forbidden_role` and the `organizations` row is unchanged

#### Scenario: Retention below the floor
- **WHEN** an admin submits `media_retention_days = 10`
- **THEN** validation fails and nothing is written

### Requirement: Erasing operational data needs the company name
`apagarDadosOperacionaisDaOrganizacao()` SHALL require company-admin rights and satisfied MFA (`mfa_required` otherwise), compare `confirmNome` server-side with the stored organization name (`confirmacao_nao_confere` on mismatch), and audit `org.dados_operacionais_apagados`.

#### Scenario: Wrong confirmation name
- **WHEN** an admin types a name different from the organization's
- **THEN** the action returns `confirmacao_nao_confere` and no data is deleted

### Requirement: Notification sounds per organization
`GET /api/v1/settings/sons` SHALL be readable by any member (`viewer`), while `POST` and `DELETE` SHALL require `manager`, storing the configuration under `organizations.settings.sons_de_aviso` with a non-destructive merge of the other `settings` keys.

#### Scenario: Agent tries to change sounds
- **WHEN** an `agent` sends `POST /api/v1/settings/sons`
- **THEN** the response is 403 and `settings.sons_de_aviso` is unchanged

### Requirement: Message signature setting
`GET` and `PATCH /api/v1/settings/assinatura` SHALL require role `manager`, and `PATCH` SHALL write only `organizations.settings.assinatura_mensagens`, preserving every other `settings` key, answering 422 `validation_failed` for an invalid body.

#### Scenario: Turning the signature on
- **WHEN** a manager PATCHes a valid signature configuration
- **THEN** `settings.assinatura_mensagens` holds it and other keys such as `sons_de_aviso` are untouched

### Requirement: Optional features page gated by role
`/app/settings/recursos` SHALL redirect to `/403` for users below `manager` who are not the server owner, list the optional features of `RECURSOS_OPCIONAIS`, and allow toggling a feature only when the user's role reaches the role mapped to that feature's `quemDecide` (installation-level features only for the server owner).

#### Scenario: Agent opens optional features
- **WHEN** an `agent` requests `/app/settings/recursos`
- **THEN** the response redirects to `/403`

### Requirement: Update panel only for the server owner
`/app/settings/atualizacao` SHALL respond with Next.js `notFound()` (404) to any user who is not a platform admin.

#### Scenario: Organization admin probes the update panel
- **WHEN** an org admin without `is_platform_admin` requests `/app/settings/atualizacao`
- **THEN** the response is 404

### Requirement: Notification preferences are not yet persisted
`updateNotificationPrefs()` SHALL validate its input and then return `{ ok: false, error: "feature_not_yet_available" }` without writing anything.

#### Scenario: Saving notification preferences
- **WHEN** a user submits valid notification preferences
- **THEN** the action returns `feature_not_yet_available` and no row changes
