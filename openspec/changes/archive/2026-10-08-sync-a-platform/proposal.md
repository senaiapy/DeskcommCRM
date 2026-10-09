## Why

The platform specs were written against upstream commit `477b77678`. HEAD is 491 upstream commits later and the platform code drifted: the reseller billing core (plans, subscriptions, seat and number ceilings, migration 0583) landed, the team invite/accept/reactivate paths learned the plan limit, the audit CSV export reads in pages, a route for the RGPD art. 15 paragraphs was added to settings, and the public pages (`/`, `/legal/*`, `/signup`) now resolve the brand from the database. This change brings the catalog back in line with the code.

## What Changes

- Spec-only: no product code changes.
- New capability `billing-plans` documents migration 0583, `lib/cobranca/*`, `/api/v1/admin/cobranca/planos*`, `/api/v1/admin/tenants/[id]/assinatura*` and the billing surfaces.
- `team-invites`: the bulk invite, accept and reactivate requirements gain the plan-limit refusal.
- `audit-log`: new requirement for `GET /api/v1/audit/export`.
- `settings`: the optional-features page hides installation-only modules; new requirement for `PATCH /api/v1/settings/art15`.
- `white-label-branding` and `navigation-shell`: the public landing page at `/` and the brand on public pages.
- BREAKING: none.

## Capabilities

### New Capabilities
- `billing-plans`: installation-level billing of the organizations (plans, subscriptions, trial, plan ceilings), off by default.

### Modified Capabilities
- `team-invites`
- `audit-log`
- `settings`
- `white-label-branding`
- `navigation-shell`

## Impact

Only `openspec/specs/*` after archive. No API, schema or runtime change.
