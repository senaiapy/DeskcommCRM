## Purpose
The first-run wizard that turns a new organization into a working installation. Pages live under `app/onboarding/` (`welcome`, `connect-whatsapp`, `connect-nuvemshop`, `setup-ai`, `funil`, `testar`, `invite-team`, `done`) behind `app/onboarding/layout.tsx`; the ordered step catalog is `PASSOS` in `lib/onboarding/passos.ts`; progress is persisted in `organizations.onboarding_state` (jsonb, schema `onboardingStateSchema` in `lib/schemas/onboarding.ts`) and completion in `organizations.onboarded_at`, written by the server actions in `app/actions/onboarding/*` (`acceptWelcome`, `skipWhatsapp`, `createDefaultAgent`, `montarQuadro`, `marcarTeste`, `sendOnboardingInvites`, `finishOnboarding`). The WhatsApp step uses `GET|POST /api/v1/onboarding/whatsapp/session` and `/api/v1/onboarding/whatsapp/qr`; the funnel step proposes a pipeline (`lib/onboarding/sugerir-funil.ts`, `pacotes-de-funil.ts`) applied by RPC `fn_aplicar_quadro_do_onboarding`.

## ADDED Requirements

### Requirement: The app shell sends unfinished organizations to the wizard
`app/app/layout.tsx` SHALL redirect to `/onboarding` when the active organization's `onboarded_at` is null and the user is not in a support session, and `app/onboarding/layout.tsx` SHALL redirect to `/app/inbox` once `onboarded_at` is set or when the user is in a support session, and to `/get-started` when the user has no active organization.

#### Scenario: New organization opens the app
- **WHEN** a member of an organization with `onboarded_at IS NULL` requests `/app/inbox`
- **THEN** the response redirects to `/onboarding`

#### Scenario: Finished organization opens the wizard
- **WHEN** a member of an organization with `onboarded_at` set requests `/onboarding/welcome`
- **THEN** the response redirects to `/app/inbox`

### Requirement: Ordered steps filtered by installation
The wizard SHALL present the steps of `PASSOS` in the order welcome → connect-whatsapp → connect-nuvemshop → setup-ai → funil → testar → invite-team, omitting `connect-nuvemshop` when `NUVEMSHOP_ENABLED` is false, and `proximoPasso()` SHALL return the first visible step not yet marked in `onboarding_state`.

#### Scenario: Store integration off
- **WHEN** `NUVEMSHOP_ENABLED` is false
- **THEN** the step list has no `connect-nuvemshop` entry and the WhatsApp step is followed by `setup-ai`

#### Scenario: Skipped step counts as resolved
- **WHEN** `onboarding_state.whatsapp` is `{ "skipped": true }`
- **THEN** the step is treated as done for navigation and reported as skipped in the final summary

### Requirement: Progress persisted as a merged jsonb patch
Each step action SHALL merge its patch into `organizations.onboarding_state` through the service-role client filtered by the organization id resolved from the user's memberships, and the welcome step SHALL write only while `onboarded_at IS NULL`, failing with `org_ja_configurada` otherwise.

#### Scenario: Stale welcome tab
- **WHEN** the welcome form is submitted after the organization finished onboarding in another tab
- **THEN** `display_name` and `onboarding_state` are not rewritten and the action reports `org_ja_configurada`

#### Scenario: Tab bound to an organization the user left
- **WHEN** a step action receives an organization id that is not among the user's memberships
- **THEN** the action fails with `forbidden` and nothing is written

### Requirement: Completion is stamped once
`finishOnboarding()` SHALL set `organizations.onboarded_at` only where it is still null, and on that first transition insert an `event_log` row `tenant.onboarded` (`entity_kind = 'organization'`) and audit both `onboarding.completed` and `tenant.onboarded`, then redirect to `/app/inbox`.

#### Scenario: Double click on finish
- **WHEN** `finishOnboarding()` runs twice for the same organization
- **THEN** exactly one `tenant.onboarded` event and one pair of audit rows exist

### Requirement: Every step is audited
Step actions SHALL audit their outcome with the codes `onboarding.welcome_completed`, `onboarding.whatsapp_skipped`, `onboarding.ai_configured`, `onboarding.agente_testado` / `onboarding.agente_teste_pulado` and `onboarding.team_invited` (plus `member.invited` per invite).

#### Scenario: Skipping WhatsApp
- **WHEN** the user skips the WhatsApp step
- **THEN** an `api_audit_log` row with action `onboarding.whatsapp_skipped` is written for the organization

### Requirement: WhatsApp pairing during onboarding
`POST /api/v1/onboarding/whatsapp/session` SHALL call `requireSupportWrite()`, require role `admin` (platform admin allowed), answer 403 `mfa_required` when `mfaEmDivida()` is true and 503 `waha_not_configured` when no WAHA client is configured, and pass the `Idempotency-Key` header to the channel connection; `GET` SHALL report `WAHA_NOT_CONFIGURED`, `NOT_STARTED` or the remote session status.

#### Scenario: WAHA not configured
- **WHEN** an admin starts the WhatsApp step on an installation without WAHA settings
- **THEN** `POST` returns 503 `waha_not_configured` and `GET` returns `data.status = "WAHA_NOT_CONFIGURED"`
