## Context

Retro-spec of the platform layer of DeskcommCRM as of 2026-10-07 (branch `main`). The code is the authority; `CLAUDE.md`, `docs/prd/01-prd-platform-base.md`, `docs/specs/01-spec-platform-base.md` and `docs/specs/09-spec-frontend-backend-integration.md` were used only as seed material.

## How the evidence was gathered

- Each capability was traced from its entry points (`proxy.ts`, `app/app/**`, `app/admin/**`, `app/onboarding/**`, `app/actions/**`, `app/api/v1/**`) into `lib/**` and into `supabase/baseline.sql` / `supabase/migrations/*` for tables, constraints, policies and functions.
- Every requirement names a concrete route, table, column, env var or error code that was opened and read; requirements that could not be backed by code were dropped (e.g. a seat limit for team invites, an HMAC on cursors).
- Line references are recorded per capability in `tasks.md`. Five capabilities (auth-session, mfa-policy, multi-tenancy-rls, rbac-roles, team-invites) were traced by a helper agent and spot-checked.
- No tests were run; this change states behaviour, it does not re-prove it. Where the repo already has a gate for an invariant, the spec names the gate file.

## Inconsistencies found (docs vs code)

1. **Cursor pagination is not HMAC-signed.** `CLAUDE.md` says "cursor opaco base64+HMAC" and `lib/api/README.md` points to `lib/api/pagination.ts`, which does not exist. Each list handler (`contacts/_handler.ts:45`, `conversations/_handler.ts:131`, `admin/*`) encodes plain base64url JSON with no signature.
2. **Idempotency-Key is not on every create POST.** Only 10 route files read the header (e.g. `messages`, `message-templates`, `agenda/agendamentos`, `channel-sessions`, `admin/tenants`).
3. **Two validation error codes.** `validation_failed` is canonical in `lib/api/errors.ts`; `validation_error` is not in `ApiErrorCodes` but is returned by about 24 routes. `auth_in_query_forbidden` is defined and never emitted.
4. **X-Request-Id is not one id end to end.** `proxy.ts` sets `x-request-id` on its own response, but most handlers mint a fresh `randomUUID()` instead of reading the incoming id, so the proxy id and the handler/audit id differ.
5. **Audit listing gate vs RLS.** `GET /api/v1/audit` and `/app/audit` admit `manager`, but policy `audit_log_select` returns rows only to org `admin` and platform admins, so managers see an empty list.
6. **Rate-limit headers.** `CLAUDE.md` promises `X-RateLimit-*` with every 429; only three routes send them, the rest send `Retry-After` only. Counters are fixed-window, while Spec 11 says sliding window (acknowledged in `lib/mcp/rate-limit.ts`).
7. **Role ranks.** `CLAUDE.md` and `fn_role_at_least` use 1–4; `ROLE_RANK` in `lib/auth/types.ts` is 1–5 with the non-human `ai_operator` at 3. The order agrees.
8. **Auth entry points.** Docs mention `app/api/v1/auth/*` for login/reset; these are server actions in `app/actions/auth/*` with pages under `app/(public)/login/*`.
9. **Membership table and policy names.** The membership table is `user_organizations`; the `tenant_isolation_<table>_all` naming holds for most tables but not for `organizations`, `user_organizations` and `team_invites`.
10. **Agency console.** `docs/specs/19-spec-console-de-agencia.md` has no implementation.
11. **`platform_admins.mfa_required` default.** The column defaults to `true`; `scripts/bootstrap-owner.ts` writes `false` on purpose (documented).

## Declared debts

- `updateNotificationPrefs()` validates and returns `feature_not_yet_available`; notification preferences are not persisted.
- The MFA verification lockout counts attempts in a client cookie (`mfa_attempts`), which a client can clear.
- `INVITE_TOKEN_SECRET` falls back to `INTERNAL_SECRET` and then to a hard-coded development string when neither is set.
- Without a service-role key, invites are token-only (no `team_invites` row) and resend answers 503.
- The in-memory rate-limit fallback is per process; with more than one app instance the effective limit multiplies.
- Platform admins can be granted or revoked only by a DBA; the API answers 405 by design.

## Goals / Non-Goals

- Goal: an accurate, diffable baseline per canonical capability id.
- Non-goal: fixing any drift above. Each item is a candidate for a later, separate change.
