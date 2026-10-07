## Purpose
Session authentication on Supabase Auth through `@supabase/ssr`. Login, logout, password reset and MFA challenge are Server Actions in `app/actions/auth/*` (`signInWithPassword`, `signOut`, `requestPasswordReset`, `updatePassword`, `verifyMfa`, `signInWithGoogle`) backing the public pages under `app/(public)/login/*` (`/login`, `/login/forgot`, `/login/reset`, `/login/mfa`, `/login/recovery`, `/login/continuar`). Email links land on `GET|POST /auth/confirm` and Google OAuth returns to `GET /auth/callback`. Clients are `lib/supabase/server.ts`, `lib/supabase/browser.ts` and `lib/supabase/admin.ts`; the request gate is `proxy.ts` (authentication part only) with `lib/auth/public-paths.ts`; the authenticated user loader is `loadAuthUser()` in `lib/auth/server.ts`. Session-scoped API helpers live in `app/api/v1/auth/*` (`realtime-token`, `support`, `interface`). The first owner of a self-host install is created by `scripts/bootstrap-owner.ts`.

## ADDED Requirements

### Requirement: Session cookie name and attributes
The server client (`lib/supabase/server.ts`) and the request gate (`proxy.ts`) SHALL store the Supabase session in a cookie named `sb-deskcomm-auth` with `httpOnly: true`, `path: "/"`, `secure` from `cookieSecure()` and `sameSite: "strict"`, except the Google sign-in starter client `createClientDeEntradaComGoogle()` which uses `sameSite: "lax"` only for the PKCE verifier.

#### Scenario: Password login sets the strict session cookie
- **WHEN** a user signs in through `signInWithPassword` with valid credentials and no verified TOTP factor
- **THEN** the response sets cookie `sb-deskcomm-auth` with `HttpOnly` and `SameSite=Strict`
- **AND** the action redirects to the sanitized `next` path or `/app`

### Requirement: JWT is always validated with getUser
Server-side identity checks in `proxy.ts`, `loadAuthUser()`, `requirePlatformAdmin()` and `GET /api/v1/auth/realtime-token` SHALL resolve the user with `supabase.auth.getUser()` (server-validated JWT) and never authorize with `getSession()`.

#### Scenario: Realtime token requires a validated user
- **WHEN** `GET /api/v1/auth/realtime-token` is called without a valid session
- **THEN** the response is HTTP 401 with `error.code = "unauthenticated"`

### Requirement: Request gate rejects unauthenticated requests
`proxy.ts` SHALL, for any path not matched by `PUBLIC_PATHS` in `lib/auth/public-paths.ts`, answer HTTP 401 JSON `{ error: { code: "unauthenticated" } }` for `/api/*` paths and a redirect to `/login?next=<path>` for UI paths when `getUser()` returns no user.

#### Scenario: Anonymous API call
- **WHEN** an unauthenticated request hits a non-public `/api/v1/*` path
- **THEN** the response is HTTP 401 with body `error.code = "unauthenticated"` and an `x-request-id` header

#### Scenario: Anonymous page navigation
- **WHEN** an unauthenticated browser requests `/app/inbox`
- **THEN** the response redirects to `/login?next=%2Fapp%2Finbox`

#### Scenario: Public auth paths bypass the gate
- **WHEN** an unauthenticated request hits `/login`, `/auth/confirm` or `/auth/callback`
- **THEN** `proxy.ts` passes the request through without calling `getUser()`

### Requirement: Password login is rate-limited and audited
`signInWithPassword` SHALL check `authRateLimited("login", ...)` and `contaBloqueadaPorFalhas(email, ...)` before calling GoTrue, returning `{ ok: false, error: "rate_limited" }` with audit `auth.login_rate_limited` when blocked, `{ ok: false, error: "invalid_credentials" }` with audit `auth.login_failed` on wrong credentials, and audit `auth.login_success` on success.

#### Scenario: Wrong password
- **WHEN** `signInWithPassword` is called with a wrong password
- **THEN** it returns `error = "invalid_credentials"`
- **AND** an `api_audit_log` row with action `auth.login_failed` carries `metadata.email_hash` instead of the email

### Requirement: Login with an enrolled factor requires the MFA challenge
`signInWithPassword` and `GET /auth/callback` SHALL NOT complete login for a user with a verified TOTP factor: the action returns `{ ok: false, error: "mfa_required", challengeId }` and the callback redirects to `/login/mfa?factor=<id>&next=<path>`; `verifyMfa` then elevates the session to `aal2` and audits `auth.mfa_success`.

#### Scenario: Password login with TOTP enrolled
- **WHEN** a user with a verified TOTP factor submits correct credentials
- **THEN** `signInWithPassword` returns `error = "mfa_required"` and the factor id in `challengeId`

#### Scenario: Too many wrong codes
- **WHEN** `verifyMfa` has already counted 3 failures in the `mfa_attempts` cookie
- **THEN** it returns `error = "mfa_locked"` with `retry_in_seconds = 60`

### Requirement: Logout clears session and active organization
`signOut` SHALL call `supabase.auth.signOut()`, delete the `active_org` cookie, audit `auth.logout` for the known user and redirect to `/login`.

#### Scenario: User signs out
- **WHEN** an authenticated user invokes `signOut`
- **THEN** the `active_org` cookie is deleted, an `auth.logout` audit row is written, and the response redirects to `/login`

### Requirement: Password reset flow
`requestPasswordReset` SHALL call `resetPasswordForEmail` with `redirectTo` `<origin>/auth/confirm?type=recovery` (origin falls back to `NEXT_PUBLIC_APP_URL`), return a neutral `{ ok: true }` regardless of account existence, and be limited by `authRateLimited("reset", email, ...)`; `/auth/confirm` with `type=recovery` SHALL redirect to `/login/reset`, where `updatePassword` sets the new password, signs out and redirects to `/login?reset=success`.

#### Scenario: Reset requested
- **WHEN** `requestPasswordReset` is called with a syntactically valid email
- **THEN** it returns `{ ok: true }` and audits `auth.password_reset_requested` with `metadata.email_hash`

#### Scenario: Recovery session with MFA enrolled
- **WHEN** `updatePassword` runs in an `aal1` recovery session whose next level is `aal2` and no `mfa_code` is provided
- **THEN** it returns `error = "mfa_required"` and the password is not changed

### Requirement: Email confirmation link is consumed only by POST
`GET /auth/confirm` with `token_hash` and `type` SHALL redirect to `/login/continuar` without verifying the OTP, and only `POST /auth/confirm` (form fields `token_hash`, `type`) SHALL call `verifyOtp`, answering with 303 redirects.

#### Scenario: Link prefetch does not burn the token
- **WHEN** `GET /auth/confirm?token_hash=X&type=recovery` is requested
- **THEN** the response redirects to `/login/continuar?type=recovery&token_hash=X` and no OTP is verified

#### Scenario: Missing token
- **WHEN** `/auth/confirm` is reached with neither `token_hash`+`type` nor `code`
- **THEN** the response redirects to `/login?error=link_invalido`

### Requirement: Google OAuth callback
`GET /auth/callback` SHALL exchange `code` for a session with `exchangeCodeForSession`, redirect to `/login?error=entrada_com_google_cancelada` when the provider returned `error`, to `/login?error=entrada_com_google` when `code` is missing or the exchange fails (auditing `auth.google_signin_failed`), and to `/login?error=acesso_revogado` when the user has no active membership and their access was revoked.

#### Scenario: Exchange fails
- **WHEN** `/auth/callback?code=bad` is requested and GoTrue rejects the code
- **THEN** an `auth.google_signin_failed` audit row is written and the response redirects to `/login?error=entrada_com_google`

### Requirement: Admin client is service-role and never persists sessions
`createAdminClient()` in `lib/supabase/admin.ts` SHALL build a singleton client with `SUPABASE_SERVICE_ROLE_KEY`, `persistSession: false`, `autoRefreshToken: false` and header `X-Client-Info: deskcomm-crm/admin`.

#### Scenario: Repeated admin client creation
- **WHEN** `createAdminClient()` is called twice in the same process
- **THEN** the same client instance is returned

### Requirement: Bootstrap of the first owner
`scripts/bootstrap-owner.ts` SHALL require `OWNER_EMAIL` and `OWNER_PASSWORD` (organization name from `OWNER_ORG_NAME`, default `Minha Empresa`) and idempotently create the confirmed auth user, the organization, a `user_organizations` row with `role = 'admin'`, and a `platform_admins` row with `scope = 'full'` and `mfa_required = false`.

#### Scenario: Missing owner credentials
- **WHEN** the script runs without `OWNER_EMAIL` or `OWNER_PASSWORD`
- **THEN** it throws `Faltam OWNER_EMAIL / OWNER_PASSWORD.` and writes nothing

#### Scenario: Owner promoted without forced MFA
- **WHEN** the script completes on a fresh database
- **THEN** `platform_admins` has a row for the owner with `scope = 'full'` and `mfa_required = false`
