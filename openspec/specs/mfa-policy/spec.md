# mfa-policy Specification

## Purpose
Optional TOTP multi-factor authentication governed by two independent, additive policies: the platform policy `platform_admins.mfa_required` and the organization policy in `organizations.settings.security` (`mfa_required`, `mfa_required_min_role`, `mfa_grace_days`, `mfa_policy_changed_at`). The pure rule lives in `lib/auth/politica-mfa.ts` (`avaliaPoliticaDeMfa`, `exigeCadastroDeMfa`, `empresaExigeMfa`, `politicaDaEmpresa`); I/O wrappers are `exigenciaDeMfa`, `requiresMfa`, `isMfaEnrolled`, `sessionAal` and `mfaEmDivida` in `lib/auth/server.ts`. The enrollment gate is `MfaEnrollGate` rendered by `app/app/layout.tsx`; the session gate is the `mfa_required` 403 in `requireRole` (`lib/auth/require-role.ts`). Settings live on the page Configurações › Segurança (`app/app/settings/security/page.tsx`) backed by the Server Actions `definirExigenciaDeMfa` and `desativarMfaDaConta` in `app/actions/auth/politicaDeMfa.ts`; enrollment and challenge use `enrollMfa`, `confirmMfaEnroll`, `verifyMfa` and `useRecoveryCode`.

## Requirements

### Requirement: Default is to not require MFA
`empresaExigeMfa(settings)` SHALL return `true` only when `organizations.settings.security.mfa_required === true`, so an organization without the key requires no enrollment, and `avaliaPoliticaDeMfa` SHALL return `{ exige: false, bloqueia: false }` when neither the platform nor the organization requires MFA.

#### Scenario: Organization never chose a policy
- **WHEN** `organizations.settings` has no `security` key and the user is not a platform admin
- **THEN** `exigeCadastroDeMfa` returns `false` and the app layout does not render `MfaEnrollGate`

### Requirement: Platform policy for platform admins
`avaliaPoliticaDeMfa` SHALL return `{ exige: true, bloqueia: true }` for a platform admin whose non-revoked `platform_admins` row has `mfa_required = true`, regardless of the organization policy and without any grace period.

#### Scenario: Platform requires, organization does not
- **WHEN** a platform admin with `platform_admins.mfa_required = true` is active in an organization with no MFA policy
- **THEN** `requiresMfa` returns `true`

### Requirement: Organization minimum role with legacy fallback
The organization policy SHALL require enrollment for members whose `ROLE_RANK` is at least `settings.security.mfa_required_min_role` (one of `none`, `admin`, `manager`, `agent`, `viewer`), and when that key is absent `mfa_required: true` SHALL be read as minimum role `admin`.

#### Scenario: Legacy boolean only
- **WHEN** `settings.security = { "mfa_required": true }` and the user is a `manager`
- **THEN** `exigeCadastroDeMfa` returns `false`
- **AND** for an `admin` of the same organization it returns `true`

#### Scenario: Minimum role agent
- **WHEN** `settings.security.mfa_required_min_role = "agent"` and the user is an `agent`
- **THEN** `exigeCadastroDeMfa` returns `true`

### Requirement: Grace period before blocking
When `settings.security.mfa_grace_days` is between 1 and 30, the policy SHALL return `{ exige: true, bloqueia: false, prazoAte }` until `max(mfa_policy_changed_at, user_organizations.accepted_at) + mfa_grace_days` and `bloqueia: true` afterwards, while values ≤ 0, invalid values or a missing anchor SHALL block immediately and values above 30 are clamped to 30.

#### Scenario: Inside the grace window
- **WHEN** the policy changed 2 days ago with `mfa_grace_days = 7` and the user's role is reached
- **THEN** `avaliaPoliticaDeMfa` returns `exige = true`, `bloqueia = false` and `prazoAte` 5 days ahead

#### Scenario: Grace expired
- **WHEN** `now` is after `prazoAte`
- **THEN** `avaliaPoliticaDeMfa` returns `bloqueia = true` and `prazoAte = null`

### Requirement: Session proof is independent of policy
`mfaEmDivida()` SHALL return `true` exactly when the user has a verified TOTP factor and the session's authenticator assurance level is not `aal2`, without consulting either policy, and `requireRole` SHALL then answer HTTP 403 `mfa_required` for callers that otherwise pass the role check (auditing `authz.denied` with `metadata.reason = "mfa_required"`).

#### Scenario: Voluntarily enrolled user in aal1 session
- **WHEN** a user with a verified TOTP factor and an `aal1` session calls a route gated by `requireRole("admin")` while being admin, in an organization that does not require MFA
- **THEN** the response is HTTP 403 with `error.code = "mfa_required"`

#### Scenario: Insufficient role hides MFA state
- **WHEN** a `viewer` with an `aal1` session and an enrolled factor calls a route gated by `requireRole("admin")`
- **THEN** the response is HTTP 403 with `error.code = "forbidden_role"`, not `mfa_required`

### Requirement: Only an organization admin changes the organization policy
`definirExigenciaDeMfa({ minRole, graceDays })` SHALL reject callers whose active-organization role is not `admin`, callers with `mfaEmDivida()`, invalid `minRole` and `graceDays` outside integer 0..30, and on change SHALL write `settings.security.{mfa_required, mfa_required_min_role, mfa_grace_days, mfa_policy_changed_at}` via the service-role client filtered by the active organization id, auditing `security.mfa_exigida` or `security.mfa_dispensada`.

#### Scenario: Manager attempts to change the policy
- **WHEN** a `manager` calls `definirExigenciaDeMfa`
- **THEN** it returns `{ ok: false }` and `organizations.settings` is unchanged

#### Scenario: Admin requires MFA from managers with 7 days grace
- **WHEN** an admin in an `aal2` (or factor-less) session calls `definirExigenciaDeMfa({ minRole: "manager", graceDays: 7 })`
- **THEN** `settings.security.mfa_required = true`, `mfa_required_min_role = "manager"`, `mfa_grace_days = 7` and `mfa_policy_changed_at` is set
- **AND** an audit row `security.mfa_exigida` records `papel_minimo_anterior` and `papel_minimo_novo`

#### Scenario: Same value resubmitted
- **WHEN** the submitted `minRole` and `graceDays` equal the effective current values
- **THEN** the action returns `{ ok: true }` without writing `organizations` or the audit log, so `mfa_policy_changed_at` is not reset

### Requirement: Disabling one's own factor requires aal2
`desativarMfaDaConta()` SHALL refuse while any policy still requires enrollment for the caller, SHALL refuse unless `sessionAal()` is `aal2`, and otherwise SHALL unenroll every TOTP factor and audit `security.mfa_desativada`.

#### Scenario: aal1 session tries to disable
- **WHEN** an enrolled user whose policy does not require MFA calls `desativarMfaDaConta` from an `aal1` session
- **THEN** it returns `{ ok: false }` and the factor remains verified

#### Scenario: Policy still requires
- **WHEN** the organization's `mfa_required_min_role` reaches the caller's role
- **THEN** `desativarMfaDaConta` returns `{ ok: false }` regardless of the session level

### Requirement: Platform admin surface enforces its own MFA flag
`requirePlatformAdmin()` (`lib/auth/requirePlatformAdmin.ts`) SHALL redirect to `/login/mfa?next=/admin` when the caller's `platform_admins.mfa_required` is `true` and the session is not `aal2`.

#### Scenario: Platform admin with required MFA in aal1
- **WHEN** a platform admin with `mfa_required = true` opens an `/admin` page in an `aal1` session
- **THEN** the response redirects to `/login/mfa?next=/admin`
