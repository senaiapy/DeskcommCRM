# multi-tenancy-rls Specification

## Purpose
Tenant isolation for a shared Postgres database. Tenants are rows of `public.organizations` (status `active|suspended|redacted|archived`); membership is `public.user_organizations` (`user_id`, `organization_id`, `role`, `accepted_at`, `revoked_at`). Every tenant-aware table carries `organization_id` and Row Level Security policies built on the `SECURITY DEFINER` helpers `fn_user_org_ids()`, `fn_user_role_in_org(uuid)`, `fn_role_at_least(uuid, text)` and `fn_is_platform_admin()` (schema in `supabase/baseline.sql`). The active organization of a request is resolved in `lib/auth/server.ts` (`orgAtivaSemPortao`, `resolveActiveOrg`) from the `active_org` cookie, switched by the Server Action `app/actions/shell/setActiveOrg.ts`; API routes use `orgAtivaDaApi`/`requireRole` in `lib/auth/require-role.ts`. Service-role access goes through `lib/supabase/admin.ts`. External tenant provisioning is `POST /api/v1/tenants/provision`. Isolation is proven by `tests/invariants/rls-isolation.test.ts` and `tests/invariants/rls-completude-varredura.test.ts` (run by `pnpm test:db`).

## Requirements

### Requirement: Membership function excludes revoked memberships
`public.fn_user_org_ids()` SHALL return the `organization_id` of every `user_organizations` row with `user_id = auth.uid()` and `revoked_at is null`, plus the organization of an `active` support session from `fn_support_context()`, and EXECUTE on it SHALL be revoked from `public` and `anon` and granted to `authenticated` and `service_role`.

#### Scenario: Revoked member loses tenant visibility
- **WHEN** a user's `user_organizations.revoked_at` is set for organization A
- **THEN** `fn_user_org_ids()` evaluated with that user's JWT no longer returns A

#### Scenario: Anonymous key cannot call the helper
- **WHEN** the `anon` role calls `rpc/fn_user_org_ids`
- **THEN** Postgres denies EXECUTE

### Requirement: Tenant tables are isolated by RLS
Tenant-aware tables SHALL have RLS enabled with policies that restrict rows to `organization_id in (select fn_user_org_ids())` (optionally widened by `fn_is_platform_admin()`), so an authenticated user of organization A reads zero rows of organization B.

#### Scenario: Cross-tenant read
- **WHEN** `tests/invariants/rls-isolation.test.ts` sets `request.jwt.claims` to a user of org A and selects from each table in its `TABLES` list
- **THEN** zero rows of org B are returned and the user's own org rows are returned (positive control)

### Requirement: Every tenant-aware table is covered by the isolation test
`tests/invariants/rls-completude-varredura.test.ts` SHALL derive the set of tables with an `organization_id` column from the catalog and fail when any of them is neither in the `TABLES` list of `rls-isolation.test.ts` nor in a named exception.

#### Scenario: New table without isolation proof
- **WHEN** a migration adds a table with `organization_id` but does not add it to `TABLES`
- **THEN** `pnpm test:db` fails on the completeness sweep

### Requirement: Organization and membership table policies
`public.organizations` SHALL be readable through policy `orgs_select` only for `id in fn_user_org_ids()` or platform admins and writable only through `orgs_write_platform_admin` (`fn_is_platform_admin_full()`), and `public.user_organizations` SHALL allow SELECT to the row's own user, organization admins (`fn_role_at_least(organization_id, 'admin')`) or platform admins, and INSERT/UPDATE/DELETE only to organization admins or full-scope platform admins.

#### Scenario: Tenant admin updates organizations with the session client
- **WHEN** a tenant `admin` (not platform admin) updates their `organizations` row through the session client
- **THEN** zero rows are affected

#### Scenario: Agent lists memberships
- **WHEN** an `agent` selects `user_organizations` of their organization through the session client
- **THEN** only their own membership row is returned

### Requirement: Active organization resolution from a trusted source
`orgAtivaSemPortao` SHALL resolve the active organization from the `active_org` cookie only when it matches one of the user's loaded memberships, otherwise from the first membership whose organization is operant, otherwise the first membership, and SHALL return `null` when the user has no membership; it never reads the organization from a request body.

#### Scenario: Forged cookie
- **WHEN** the `active_org` cookie holds an organization id the user is not a member of
- **THEN** the resolved active organization is the user's first operant membership

#### Scenario: No membership
- **WHEN** the user has zero non-revoked memberships
- **THEN** `requireRole` responds HTTP 403 with `error.code = "forbidden_tenant"`

### Requirement: Switching the active organization
`setActiveOrg(orgId)` SHALL accept only a UUID, refuse during a support session and when `mfaEmDivida()` is true, verify a fresh `user_organizations` row with `revoked_at is null`, `accepted_at is not null` and `organizations.status = 'active'`, then set cookie `active_org` (`httpOnly`, `sameSite: strict`, 30 days) and audit `organization.switched`.

#### Scenario: Switch to a suspended organization
- **WHEN** a member calls `setActiveOrg` for an organization whose `status` is `suspended`
- **THEN** it returns `{ ok: false, error: "forbidden" }` and the cookie is unchanged

#### Scenario: Successful switch
- **WHEN** a member calls `setActiveOrg` for an active organization they belong to
- **THEN** cookie `active_org` holds that id and an `organization.switched` audit row records `previous_organization_id`

### Requirement: Writes from a stale tab are refused
`requireRole` SHALL compare the `X-Org-Da-Aba` request header, when present, with the cookie-resolved active organization and respond HTTP 409 with `error.code = "org_divergente"` and `details.organization_id` when they differ.

#### Scenario: Two tabs on different organizations
- **WHEN** a mutating request carries `X-Org-Da-Aba: <org A>` while the `active_org` cookie resolves to org B
- **THEN** the response is HTTP 409 `org_divergente` and no write occurs

### Requirement: Non-operant organizations are blocked in the API
`requireRole` and `orgAtivaDaApi` SHALL respond HTTP 403 with `error.code = "org_suspended"` when the active organization is not operant, unless the route passes `permiteOrgSuspensa`, and pages using `resolveActiveOrg` SHALL redirect to `/account-suspended`.

#### Scenario: API call on a suspended organization
- **WHEN** a member of an organization with `status = 'suspended'` calls a route gated by `requireRole("viewer")`
- **THEN** the response is HTTP 403 `org_suspended`

### Requirement: Service-role handlers filter the organization from a trusted source
Handlers using `createAdminClient()` (which bypasses RLS) SHALL filter `organization_id` by a value resolved from the session (active organization), a validated token or a signed payload, never from the request body.

#### Scenario: MFA policy write through service role
- **WHEN** `definirExigenciaDeMfa` updates `organizations.settings`
- **THEN** the UPDATE is filtered by `id = <active organization from resolveActiveOrg>`

### Requirement: External tenant provisioning endpoint
`POST /api/v1/tenants/provision` SHALL respond HTTP 404 `not_found` unless `TENANT_PROVISIONING_SECRET` has at least 32 characters, SHALL compare the `Authorization: Bearer` value with that secret in constant time returning HTTP 401 `unauthenticated` on mismatch and HTTP 429 `rate_limited` after 10 failures per IP per minute, and on success SHALL return `{ organization_id, api_key, replay }` with HTTP 201 (200 on replay) and `Cache-Control: no-store`.

#### Scenario: Secret not configured
- **WHEN** `TENANT_PROVISIONING_SECRET` is empty or shorter than 32 characters
- **THEN** the endpoint responds HTTP 404 `not_found`

#### Scenario: Owner email already has an account
- **WHEN** a valid request names an `owner_email` that already exists in the installation
- **THEN** the response is HTTP 409 with `error.code = "owner_email_ja_tem_conta"`

#### Scenario: Replayed provisioning
- **WHEN** the same `integration` and `external_id` are provisioned again
- **THEN** the response is HTTP 200 with `replay = true` and the same `organization_id`
