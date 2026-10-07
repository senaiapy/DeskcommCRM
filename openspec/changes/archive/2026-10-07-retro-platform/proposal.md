## Why

DeskcommCRM has no OpenSpec baseline: its platform behaviour (auth, tenancy, RBAC, the `/api/v1` contract, tokens, audit, admin console, branding, i18n, e-mail, settings, navigation, rate limiting) is described only in `CLAUDE.md`, `docs/prd/*` and `docs/specs/*`, several of which have drifted from the code. This change writes a retro-spec — a catalog of what the code does today — using the shared capability ids of `openspec-capability-map.md`, so the platform layer can be compared side by side with TEMPLATE_CRM, TEMPLATE_CRM_V1 and TEMPLATE_ERP_v1 in the compare-and-refactor step.

## What Changes

- Documentation only: sixteen new capability specs under `openspec/specs/` once archived. No product code, migration, env var or route changes.
- Every requirement was checked against the code; where docs and code disagree the spec follows the code and `design.md` lists the drift.
- **BREAKING:** none. Nothing in the running product changes; the only thing that "breaks" is the assumption that `CLAUDE.md` is fully accurate — the drift list in `design.md` names where it is not.

## Capabilities

### New Capabilities
- `auth-session`: Supabase Auth login/logout/reset/callback via server actions, cookie `sb-deskcomm-auth`, `getUser()` in `proxy.ts`, owner bootstrap.
- `mfa-policy`: optional TOTP MFA, the two additive policies (`platform_admins.mfa_required`, `organizations.settings.security.*`), `mfaEmDivida()` 403.
- `multi-tenancy-rls`: `organization_id` + RLS via `fn_user_org_ids()`, `user_organizations`, active-org cookie, tenant provisioning.
- `rbac-roles`: `requireRole()` ranks, `fn_role_at_least`, platform admin role, last-admin guards.
- `team-invites`: `team_invites`, HMAC invite tokens, invite/resend/revoke/accept, member revoke/reactivate.
- `onboarding`: the `/onboarding` wizard, `organizations.onboarding_state` / `onboarded_at`.
- `api-rest-contract`: `ok()`/`fail()` envelopes, error codes, `X-Request-Id`, Idempotency-Key receipts, cursor pagination, `requireSupportWrite()`.
- `api-tokens`: `dsk_` bearer tokens in `api_tokens`, dual auth per route, public-path requirement, token ceilings.
- `audit-log`: `api_audit_log`, `AUDIT_ACTIONS`, append-only grants, tenant and platform audit views, retention floor.
- `platform-admin-console`: `/admin` surface, tenant suspend/reactivate, support sessions (impersonation).
- `white-label-branding`: `platform_branding` and organization branding over env seed, logo upload, never-throw resolvers.
- `i18n`: language registry, resolution chain, `traduzir()` dictionary.
- `platform-email`: SMTP-or-Resend router, `platform_smtp_settings`, GoTrue templates.
- `settings`: profile, organization profile, danger zone, sounds, signature, optional features, update panel.
- `navigation-shell`: proxy routing, shell redirects, `NAV_CATALOG`, role/module/interface filtering.
- `rate-limit`: Redis fixed-window counters with memory fallback, auth/token/route budgets.

### Modified Capabilities
- None.

## Impact

- Files added: `openspec/changes/retro-platform/**` only.
- Ids dropped from the seed list: none. `custom-fields-views` was checked and not written: pipeline custom field definitions (`crm_pipelines.settings.fields`, `custom_fields jsonb`) are documented with the CRM capabilities and no saved-views feature exists.
