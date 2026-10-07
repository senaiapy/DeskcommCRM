# platform-admin-console Specification

## Purpose
The cross-tenant console for installation operators ("platform admins"). Pages live under `app/admin/(protected)/` (dashboard, tenants, tenants/[id] with agent and health, users, platform-admins, audit, incidents, inbox, lgpd, usage, marca, email, google, meta, modulos, extensoes, sistema, configuracao, cadastro, destinos-internos) with `/admin/forbidden` for refusals; APIs under `app/api/v1/admin/*` (tenants list/create/suspend/reactivate/impersonate, impersonate/end, users, platform-admins, audit, incidents, inbox/conversations, lgpd/requests, usage, dashboard/kpis). Identity comes from `public.platform_admins` (`scope`, `mfa_required`, `revoked_at`) checked by `lib/auth/requirePlatformAdmin.ts`; temporary support access into a tenant uses `public.platform_support_sessions`, the RPCs `fn_start_support` / `fn_end_support` / `fn_support_context`, and the signed cookie `deskcomm-impersonate` from `lib/impersonate/cookie.ts` (secret `IMPERSONATE_COOKIE_SECRET`). The agency console of `docs/specs/19-spec-console-de-agencia.md` has no implementation.

## Requirements

### Requirement: Admin surface gated in proxy and server
`proxy.ts` SHALL redirect any `/admin/*` request other than `/admin/forbidden` to `/admin/forbidden` when RPC `fn_is_platform_admin` is false, and `requirePlatformAdmin()` SHALL authoritatively require a `platform_admins` row with `revoked_at IS NULL`, redirecting to `/login/mfa?next=/admin` when that row has `mfa_required = true` and the session is not `aal2`.

#### Scenario: Tenant admin opens /admin
- **WHEN** an organization admin without a `platform_admins` row requests `/admin/tenants`
- **THEN** the response redirects to `/admin/forbidden`

#### Scenario: Admin API without platform role
- **WHEN** a non platform admin calls `GET /api/v1/admin/tenants`
- **THEN** the response is 403 `forbidden`

### Requirement: Writes need full scope and satisfied MFA
Mutating admin operations SHALL use `requirePlatformAdminEscrita()`, which refuses with 403 `forbidden_scope` when `platform_admins.scope` is not `full` and 403 `mfa_required` when `mfaEmDivida()` is true.

#### Scenario: Read-only platform admin suspends a tenant
- **WHEN** a platform admin whose scope is not `full` calls `POST /api/v1/admin/tenants/{id}/suspend`
- **THEN** the response is 403 `forbidden_scope` and the organization status is unchanged

### Requirement: Tenant suspension and reactivation
`POST /api/v1/admin/tenants/{id}/suspend` SHALL call `requireSupportWrite(id)` and `requirePlatformAdminEscrita()`, answer 404 `not_found` for an unknown organization, perform the change in RPC `fn_suspender_organizacao` (status, pending jobs to failed, queued messages), answer 409 `retry_later` on a lock conflict, and audit `tenant.suspended`; `POST .../reactivate` SHALL audit `tenant.reactivated`.

#### Scenario: Suspend an existing tenant
- **WHEN** a full-scope platform admin with MFA suspends an active organization
- **THEN** the organization status becomes suspended and an audit row `tenant.suspended` is written

### Requirement: Tenant listing
`GET /api/v1/admin/tenants` SHALL accept `q`, `status` (`active`, `suspended`, `onboarding`, `redacted`), `cursor` and `limit`, treat `onboarding` as `status = 'active' AND onboarded_at IS NULL`, and audit `platform_admin.tenants_listed`.

#### Scenario: Filter organizations still onboarding
- **WHEN** a platform admin lists with `status=onboarding`
- **THEN** only active organizations with `onboarded_at` null are returned

### Requirement: Platform admins are managed only by the DBA
`/api/v1/admin/platform-admins` SHALL expose only `GET` (auditing `platform_admin.platform_admins_listed`) and SHALL answer every other method with 405 `method_not_allowed` and `Allow: GET`.

#### Scenario: Attempt to grant platform admin via API
- **WHEN** a platform admin sends `POST /api/v1/admin/platform-admins`
- **THEN** the response is 405 with `error.code = "method_not_allowed"`

### Requirement: Starting a support session
`POST /api/v1/admin/tenants/{id}/impersonate` SHALL require a platform admin with satisfied MFA, accept `access_mode` `full` or `support_readonly` (default `full`), answer 503 `upstream_unavailable` when `IMPERSONATE_COOKIE_SECRET` is not ready and 409 `state_conflict` when the user is already in a support session, create the row through `fn_start_support` with a TTL of `IMPERSONATE_TTL_SECONDS` (3600 s), set the HMAC-signed cookie `deskcomm-impersonate` (httpOnly, SameSite strict), and audit `platform_admin.impersonate_started` with `acting_as_platform_admin = true`.

#### Scenario: Read-only support session
- **WHEN** a platform admin starts support with `access_mode = "support_readonly"`
- **THEN** a `platform_support_sessions` row with `expires_at` about one hour ahead exists, the response carries `redirect_url = "/app/inbox"`, and later writes in that tenant answer 403 through `requireSupportWrite()`

#### Scenario: Nested support
- **WHEN** a platform admin already inside a support session starts another
- **THEN** the response is 409 `state_conflict`

### Requirement: Ending a support session
`POST /api/v1/admin/impersonate/end` SHALL end the session through `fn_end_support` for the caller's auth session, delete the `deskcomm-impersonate` cookie, and audit `platform_admin.impersonate_ended`; `platform_support_sessions` SHALL allow at most one open session (`ended_at IS NULL`) per `auth_session_id` and grant no privilege to `anon` or `authenticated`.

#### Scenario: Exit support
- **WHEN** the admin ends support
- **THEN** the session row gets `ended_at`, the cookie is gone, and an audit row `platform_admin.impersonate_ended` exists

### Requirement: Invalid presentation cookie is dropped at the edge
On `/app*` paths, `proxy.ts` SHALL verify the `deskcomm-impersonate` cookie's HMAC and expiry with `IMPERSONATE_COOKIE_SECRET` and delete it when invalid, while the database support session stays authoritative.

#### Scenario: Tampered cookie
- **WHEN** a request to `/app/inbox` carries a `deskcomm-impersonate` cookie with a bad signature
- **THEN** the response deletes that cookie
