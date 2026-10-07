## Purpose
Transactional e-mail sent by the platform itself (not conversations with customers): team invites (`lib/auth/issue-invite.ts`, `app/actions/onboarding/sendOnboardingInvites.ts`), LGPD export delivery and SLA alarms (`lib/lgpd/email-delivery.ts`, `lib/lgpd/sla-alarm.ts`), and the access e-mails (confirmation, recovery) that Supabase Auth (GoTrue) renders from `GET /email-templates/{confirmation,recovery}`. Transport selection is `lib/email/roteador.ts` (SMTP when configured, else Resend); SMTP lives in `lib/email/smtp.ts` with configuration in the singleton table `public.platform_smtp_settings` (password encrypted) or env `SMTP_HOST`, `SMTP_PORT`, `SMTP_SECURITY`, `SMTP_USERNAME`, `SMTP_PASSWORD`, `SMTP_FROM_EMAIL`, `SMTP_FROM_NAME`; Resend lives in `lib/email/resend.ts` reading `RESEND_API_KEY` and `RESEND_FROM_EMAIL` through `valorDaInstalacao()` (`lib/instalacao/config.ts`, database `platform_config` above env). Platform admins configure SMTP at `/admin/email` (`app/actions/settings/smtp.ts`). Sender name and colors come from `marcaDaSaida()`.

## ADDED Requirements

### Requirement: One transport per send, no silent fallback
`sendEmail()` in `lib/email/roteador.ts` SHALL send through SMTP when `isSmtpConfigured(getSmtpConfig())` is true and through Resend otherwise, SHALL NOT retry a failed SMTP send through Resend, and SHALL report the transport used in the result's `via` field.

#### Scenario: SMTP misconfigured
- **WHEN** SMTP settings exist but the server rejects the login
- **THEN** the result is `ok: false` with `via: "smtp"` and no Resend call is made

### Requirement: SMTP configuration database above environment
`getSmtpConfig()` SHALL read row `id = 1` of `platform_smtp_settings` (decrypting `smtp_password_encrypted`) and SHALL fall back to the `SMTP_*` environment variables when the row is absent or unreadable; the table SHALL hold a single row (`platform_smtp_settings_singleton`).

#### Scenario: Only env configured
- **WHEN** `platform_smtp_settings` has no row and `SMTP_HOST` is set
- **THEN** the configuration source is the environment

### Requirement: SMTP settings are a platform-admin write
`updateSmtp()` SHALL require `requirePlatformAdminEscrita()` (via `escritaDeAdminOuRecusa`), validate port 1–65535 and a bare sender address, refuse to save a password when encryption is unavailable, and audit `platform_smtp_settings.updated`.

#### Scenario: Organization admin tries to change SMTP
- **WHEN** a user who is not a full-scope platform admin calls `updateSmtp`
- **THEN** the action returns a refusal code and `platform_smtp_settings` is unchanged

### Requirement: Resend needs key and sender
The Resend path SHALL read `RESEND_API_KEY` and `RESEND_FROM_EMAIL` at send time (database value above env) and SHALL return `{ ok: false, error: "not_configured" }` without throwing when either is missing.

#### Scenario: No e-mail provider at all
- **WHEN** neither SMTP nor `RESEND_API_KEY`/`RESEND_FROM_EMAIL` is configured and an invite is issued
- **THEN** the send returns `not_configured` and the invite itself is still created

### Requirement: Branded access templates for GoTrue
`GET /email-templates/{modelo}` SHALL accept only `confirmation` and `recovery` (404 otherwise), render with `marcaDaSaida(null)` resolved on every request, and be reachable without a session (listed in `PUBLIC_PATHS`).

#### Scenario: GoTrue fetches the recovery template
- **WHEN** GoTrue requests `/email-templates/recovery`
- **THEN** the response is 200 HTML carrying the current installation brand

#### Scenario: Unknown template
- **WHEN** `/email-templates/welcome` is requested
- **THEN** the response is 404

### Requirement: Invite e-mails carry the organization brand
Invite e-mails SHALL use `marcaDaSaida(organizationId)` for sender name and colors, and a delivery failure SHALL NOT fail the invite.

#### Scenario: Invite in a branded organization
- **WHEN** an admin of an organization branded "Clínica Sol" invites a colleague
- **THEN** the e-mail sender display name is the resolved organization brand
