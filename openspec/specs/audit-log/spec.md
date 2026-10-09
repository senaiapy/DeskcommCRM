# audit-log Specification

## Purpose
Append-only trail of mutations and security events. Writers call `audit()` / `auditForOrganizations()` from `lib/audit/index.ts` with an action from the single vocabulary `AUDIT_ACTIONS` in `lib/audit/actions.ts`; rows land in `public.api_audit_log` (actor user or API token, platform-admin/support flag, IP, user agent, resource, `request_id`, `metadata`). Tenants read it at `/app/audit` (`app/app/audit/page.tsx`) via `GET /api/v1/audit`; platform admins read across tenants via `GET /api/v1/admin/audit` and `GET /api/v1/admin/audit/{entryId}`. Migration 0258 makes the table insert-only for PostgREST roles; expiry is `public.fn_expurgar_auditoria_vencida` driven by `AUDIT_LOG_RETENTION_DAYS` (policy constants in `lib/retencao/politica.ts`; the retention cron itself is specified with data retention).

## Requirements

### Requirement: Fire-and-forget audit writes
`audit()` SHALL insert one row into `api_audit_log` using the service-role client when configured (else the user client under policy `audit_log_insert_tenant_member`), and SHALL never throw to the caller: an insert error SHALL be reported through `console.error` and `Sentry.captureException` while the primary mutation proceeds.

#### Scenario: Audit insert fails
- **WHEN** the insert into `api_audit_log` returns an error during `POST /api/v1/settings/api-tokens`
- **THEN** the token is still created and returned with 201, and a Sentry exception `[audit] write failed` is captured

### Requirement: Single action vocabulary
Every value written to `api_audit_log.action` SHALL be a member of the `AUDIT_ACTIONS` array in `lib/audit/actions.ts`, from which the `AuditAction` type and the admin filter list are both derived.

#### Scenario: New action added
- **WHEN** a code is appended to `AUDIT_ACTIONS`
- **THEN** it type-checks as `AuditAction` and appears in the audit filter without a second list being edited

### Requirement: Support-session attribution
When the acting user is inside an active platform support session for the audited organization, `audit()` SHALL merge `support_session_id`, `support_access_mode` and `support_auth_session_id` into `metadata` and set `acting_as_platform_admin = true`.

#### Scenario: Admin edits a tenant while supporting it
- **WHEN** a platform admin in a `full` support session updates a record of that organization
- **THEN** the audit row has `acting_as_platform_admin = true` and `metadata.support_session_id` set

### Requirement: Append-only for PostgREST roles
Roles `anon`, `authenticated` and `service_role` SHALL hold no UPDATE, DELETE or TRUNCATE privilege on `public.api_audit_log` (explicit `revoke` in migration 0258 and the baseline appendix), and RLS SHALL offer only INSERT and SELECT policies.

#### Scenario: Service key tries to erase a row
- **WHEN** a REST call with the service key issues DELETE on `api_audit_log`
- **THEN** Postgres refuses it with a permission error and the row remains

### Requirement: Tenant audit listing
`GET /api/v1/audit` SHALL require at least role `manager` (platform admins allowed read-only), filter by the active `organization_id`, accept `actor_id`, `action` (substring), `resource_type`, `from`, `to`, `cursor` and `limit`, order by `(created_at, id)` descending, and read through the RLS policy `audit_log_select`, which returns rows only to org admins and platform admins.

#### Scenario: Viewer is refused
- **WHEN** a `viewer` calls `GET /api/v1/audit`
- **THEN** the response is 403

#### Scenario: Manager passes the gate but RLS hides rows
- **WHEN** a `manager` (not admin) calls `GET /api/v1/audit`
- **THEN** the response is 200 with an empty `data` array, because `audit_log_select` requires `fn_role_at_least(organization_id, 'admin')`

### Requirement: Cross-tenant audit for platform admins
`GET /api/v1/admin/audit` and `GET /api/v1/admin/audit/{entryId}` SHALL require `requirePlatformAdmin()` and SHALL not be scoped to a single organization.

#### Scenario: Org admin calls the platform endpoint
- **WHEN** an organization admin without `is_platform_admin` calls `GET /api/v1/admin/audit`
- **THEN** the response is 403 `forbidden` and no rows are returned

### Requirement: Retention with a floor
Audit rows SHALL be expired only by `public.fn_expurgar_auditoria_vencida` (security definer, no row selector, revoked from anon/authenticated), with a default of 1825 days configurable by `AUDIT_LOG_RETENTION_DAYS` and a floor of 90 days enforced inside the function body.

#### Scenario: Operator sets 30 days
- **WHEN** `AUDIT_LOG_RETENTION_DAYS=30`
- **THEN** rows younger than 90 days are still kept

### Requirement: Cron runs audit only when they had an effect
A route under `app/api/v1/cron/` SHALL write an audit row only when the run changed something, which `tests/unit/cron-audita-so-quando-ha-efeito.test.ts` checks across every cron route.

#### Scenario: Empty routing tick
- **WHEN** the routing cron runs and assigns no conversation
- **THEN** no `api_audit_log` row is written for that run

### Requirement: Paged CSV export of the tenant audit
`GET /api/v1/audit/export` SHALL require `requireRole("manager")` (platform admins read-only), accept the filters of `GET /api/v1/audit` while ignoring `limit`, answer 422 `validation_failed` for an invalid query, read `api_audit_log` of the active organization in pages of 1000 rows ordered by `created_at` then `id` descending until 10,000 rows, an empty page or the exact `count` is reached, and respond 200 `text/csv` with `Content-Disposition: attachment; filename="audit-<date>.csv"` and `X-Request-Id`.

#### Scenario: More than one thousand entries
- **WHEN** an org admin exports a window holding 2,500 audit rows
- **THEN** the CSV has 2,500 data lines after the header instead of stopping at 1,000

#### Scenario: Window larger than the ceiling
- **WHEN** the window holds 12,000 audit rows
- **THEN** the CSV has exactly 10,000 data lines, newest first

#### Scenario: Viewer exports
- **WHEN** a `viewer` calls `GET /api/v1/audit/export`
- **THEN** the response is 403 and no CSV is produced
