# rbac-roles Specification

## Purpose
Role-based access control inside a tenant plus the cross-tenant platform-admin role. Human roles are stored in `user_organizations.role` (CHECK `viewer|agent|manager|admin`); TypeScript ranks live in `ROLE_RANK` of `lib/auth/types.ts` (`viewer 1 < agent 2 < ai_operator 3 < manager 4 < admin 5`, where `ai_operator` is a non-human role used by agents/tokens). The single route gate is `requireRole(min, opts)` in `lib/auth/require-role.ts`; database-side checks are `fn_user_role_in_org(uuid)` and `fn_role_at_least(uuid, text)` in `supabase/baseline.sql`. Platform admins are rows of `public.platform_admins` (`scope full|support_readonly`, `revoked_at`), checked by `fn_is_platform_admin()` / `fn_is_platform_admin_full()`, by `requirePlatformAdmin()` / `requirePlatformAdminEscrita()` in `lib/auth/requirePlatformAdmin.ts`, and early by `proxy.ts` for `/admin/*`. Error codes come from `lib/api/errors.ts`. Role changes of team members go through `PATCH /api/v1/team/[user_id]` (alias `PATCH /api/v1/team/[user_id]/role`).

## Requirements

### Requirement: requireRole is the single route gate
`requireRole(min)` SHALL respond HTTP 401 `unauthenticated` when `loadAuthUser()` returns no user, HTTP 403 `forbidden_tenant` when there is no active organization, and HTTP 403 `forbidden_role` (message `Permissão insuficiente. Requer role >= <min>.`) when the rank of the effective role is below `ROLE_RANK[min]`, otherwise returning `{ ok: true, user, org }` with `org.role` set to the effective role.

#### Scenario: Agent calls a manager route
- **WHEN** a user whose effective role is `agent` calls `GET /api/v1/team` (gated by `requireRole("manager")`)
- **THEN** the response is HTTP 403 with `error.code = "forbidden_role"`
- **AND** an audit row `authz.denied` records `metadata.required_role = "manager"` and `metadata.effective_role = "agent"`

#### Scenario: No session
- **WHEN** an unauthenticated request reaches a handler gated by `requireRole`
- **THEN** the response is HTTP 401 with `error.code = "unauthenticated"`

### Requirement: Effective role comes from the database
`requireRole` SHALL obtain the effective role by calling `rpc("fn_user_role_in_org", { p_org })` at request time (not from the cookie or the in-memory membership snapshot) and SHALL respond HTTP 500 `internal_error` if that RPC fails.

#### Scenario: Role demoted after login
- **WHEN** a user's `user_organizations.role` changes from `admin` to `viewer` while their session is open
- **THEN** their next call to a `requireRole("admin")` route responds HTTP 403 `forbidden_role`

### Requirement: Database role ranking
`public.fn_role_at_least(p_org, p_min)` SHALL compare the role returned by `fn_user_role_in_org(p_org)` using the levels `viewer 1, agent 2, manager 3, admin 4` and return `false` when the user has no role in that organization.

#### Scenario: Non-member
- **WHEN** `fn_role_at_least(<org B>, 'viewer')` is evaluated for a user with no membership in org B
- **THEN** it returns `false`

### Requirement: Platform admin shortcut in requireRole
`requireRole` SHALL let a platform admin without an active support session bypass the tenant rank only when the route passes `allowPlatformAdmin`: with `"leitura"` for any scope, and with `true` only for `platform_admin_scope = "full"` whose session has no MFA debt (otherwise HTTP 403 `mfa_required`), while `support_readonly` admins fall through to the normal tenant rank.

#### Scenario: support_readonly admin on a write route
- **WHEN** a platform admin with `scope = 'support_readonly'` who is a `viewer` member of the active organization calls a route gated by `requireRole("admin", { allowPlatformAdmin: true })`
- **THEN** the response is HTTP 403 `forbidden_role`

#### Scenario: Full-scope admin with read shortcut
- **WHEN** a full-scope platform admin who is a `viewer` member calls a `GET` route gated by `requireRole("admin", { allowPlatformAdmin: "leitura" })`
- **THEN** the gate returns `ok: true`

### Requirement: Platform admin status
`fn_is_platform_admin()` SHALL return `true` iff `platform_admins` has a row for `auth.uid()` with `revoked_at is null`, and `fn_is_platform_admin_full()` SHALL additionally require `scope = 'full'`.

#### Scenario: Revoked platform admin
- **WHEN** a user's `platform_admins.revoked_at` is set
- **THEN** `fn_is_platform_admin()` returns `false` for that user

### Requirement: Admin surface gating
`proxy.ts` SHALL redirect authenticated requests to `/admin/*` (except `/admin/forbidden`) to `/admin/forbidden` when `rpc fn_is_platform_admin` errors or returns false, and `requirePlatformAdmin()` SHALL redirect to `/admin/forbidden` when the caller has no non-revoked `platform_admins` row.

#### Scenario: Tenant admin opens the platform admin panel
- **WHEN** a tenant `admin` without a `platform_admins` row requests `/admin`
- **THEN** the response redirects to `/admin/forbidden`

### Requirement: Platform admin writes require full scope
`requirePlatformAdminEscrita()` SHALL throw `EscritaDePlatformAdminNegada("forbidden_scope")` when the caller's `platform_admins.scope` is not `full`, and `EscritaDePlatformAdminNegada("mfa_required")` when `mfaEmDivida()` is true.

#### Scenario: Read-only support admin writes
- **WHEN** a `support_readonly` platform admin invokes a server action guarded by `requirePlatformAdminEscrita()`
- **THEN** the action is refused with reason `forbidden_scope`

### Requirement: Active support session blocks requireRole once ended
`requireRole` SHALL respond HTTP 403 `forbidden` with message `O acompanhamento terminou. Saia para continuar.` when the user carries a support context whose status is not `active`.

#### Scenario: Expired support session
- **WHEN** a platform admin whose support session expired calls any `requireRole` route
- **THEN** the response is HTTP 403 with `error.code = "forbidden"`

### Requirement: Changing a member's role
`PATCH /api/v1/team/[user_id]` (and its alias `PATCH /api/v1/team/[user_id]/role`) SHALL call `requireSupportWrite()` and `requireRole("admin")`, validate `{ role }` against `viewer|agent|manager|admin`, respond HTTP 404 `not_found` when the target is not a member of the active organization, HTTP 409 `state_conflict` when the target is revoked or is the last non-revoked `admin` being demoted, and on success update `user_organizations.role`, audit `team.role_changed` with `old_role`/`new_role`, and return `{ user_id, role }`.

#### Scenario: Demoting the last admin
- **WHEN** the only non-revoked `admin` of an organization is PATCHed to `role = "manager"`
- **THEN** the response is HTTP 409 `state_conflict` and the role is unchanged

#### Scenario: Promote an agent to manager
- **WHEN** an admin PATCHes a non-revoked `agent` member with `{ "role": "manager" }`
- **THEN** the response is HTTP 200 with `data.role = "manager"` and an audit row `team.role_changed` is written
