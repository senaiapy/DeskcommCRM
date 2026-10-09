# team-invites Specification

## Purpose
Team membership management for an organization: inviting people by email, listing/resending/revoking pending invites, accepting an invite, and revoking or reactivating members. Pages: `app/app/team/page.tsx` (members, invites, attendants), `app/app/team/invite/page.tsx` (admin-only invite form) and the public accept page `app/team/accept-invite/[token]/page.tsx`. API: `GET /api/v1/team`, `GET /api/v1/team/assignable`, `POST /api/v1/team/invite`, `GET /api/v1/team/invites`, `POST /api/v1/team/invites/[id]/resend`, `POST /api/v1/team/invites/[id]/revoke`, `POST /api/v1/team/[user_id]/revoke`, `POST /api/v1/team/[user_id]/reactivate`, `PATCH /api/v1/team/[user_id]/interface` (role change is specified in `rbac-roles`). Libraries: `lib/auth/invite-token.ts` (stateless HMAC token, 24h TTL), `lib/auth/issue-invite.ts` (sign + email + audit), `lib/team/convites.ts` (`emitirConvite`, `reenviarConvite`, status derivation), `lib/auth/aplicar-convite.ts` and the Server Action `app/actions/team/acceptInvite.ts`. Data: `public.team_invites` and `public.user_organizations`; acceptance runs through `fn_accept_team_invite` (service role only). Emails go through `sendEmail` in `lib/email/roteador.ts` with `buildInviteEmail`.

## Requirements

### Requirement: Pending invite table with one open invite per email
`public.team_invites` SHALL store `organization_id` (FK `organizations` on delete cascade), `email`, `role` (CHECK `viewer|agent|manager|admin`), `interface_settings`, `invited_by`, `email_dispatched`, `last_sent_at`, `resend_count`, `expires_at`, `accepted_at`, `revoked_at`, with the unique partial index `team_invites_um_pendente_por_email_idx` on `(organization_id, lower(email)) where accepted_at is null and revoked_at is null`.

#### Scenario: Second pending invite to the same email
- **WHEN** an insert creates a second `team_invites` row for the same organization and the same email in different case while the first is neither accepted nor revoked
- **THEN** Postgres rejects it with a unique violation

### Requirement: Invite RLS by role
`team_invites` SHALL have RLS where SELECT (`team_invites_select`) requires membership via `fn_user_org_ids()` and `fn_role_at_least(organization_id, 'manager')` (or platform admin), and writes (`team_invites_write`) require `fn_role_at_least(organization_id, 'admin')` (or full-scope platform admin), with all privileges revoked from `anon`.

#### Scenario: Agent reads invites
- **WHEN** an `agent` of organization A selects `team_invites` through the session client
- **THEN** zero rows are returned

#### Scenario: Manager tries to revoke
- **WHEN** a `manager` updates `team_invites.revoked_at` through the session client
- **THEN** zero rows are affected

### Requirement: Bulk invite endpoint
`POST /api/v1/team/invite` SHALL call `requireSupportWrite()` and `requireRole("admin")`, validate `invitations` as 1 to 20 items of `{ email, role, interface_settings? }`, refuse with HTTP 409 `plan_limit_reached` (`details = { recurso: "assentos", limite }`) before any e-mail when the non-revoked, non-provisional members already reach `fn_limite_do_plano(org, 'assentos')`, skip emails that already have an active membership with `reason = "already_member"`, and respond HTTP 201 with `{ sent: [{ email, invite_id, expires_at, email_dispatched, email_error?, accept_url }], failed: [{ email, reason }] }`.

#### Scenario: Inviting an existing member
- **WHEN** an admin invites an email that belongs to a non-revoked member of the active organization
- **THEN** the response is HTTP 201 and that email appears in `failed` with `reason = "already_member"`

#### Scenario: Non-admin invites
- **WHEN** a `manager` calls `POST /api/v1/team/invite`
- **THEN** the response is HTTP 403 `forbidden_role`

#### Scenario: More than 20 emails
- **WHEN** the body has 21 invitations
- **THEN** the request is rejected with a validation error and no invite is created

#### Scenario: Plan seat ceiling reached
- **WHEN** billing is on, the organization's plan has `max_assentos = 3` and it already has 3 active non-provisional members
- **THEN** the response is HTTP 409 `plan_limit_reached` with `details.limite = 3` and no `team_invites` row is created

#### Scenario: Plan limit unreadable
- **WHEN** reading `fn_limite_do_plano` fails
- **THEN** the invite proceeds and the seat trigger remains the enforcement at acceptance

### Requirement: Invite token and email delivery
Each invite SHALL carry an HMAC-SHA256 token signed with `INVITE_TTL_SECONDS = 86400` expiry over `{ invite_id, email, organization_id, role, exp, iat, invited_by, interface_settings }` using `INVITE_TOKEN_SECRET` (fallback `INTERNAL_SECRET`), and `issueInvite` SHALL send the email with `accept_url = <NEXT_PUBLIC_APP_URL>/team/accept-invite/<token>` and audit `member.invited` with `email_dispatched` and `email_error`, without failing the invite when delivery fails.

#### Scenario: Email transport not configured
- **WHEN** an admin invites someone and the email transport reports an error
- **THEN** the `sent` item has `email_dispatched = false`, a non-empty `email_error` and a usable `accept_url`
- **AND** an audit row `member.invited` records `email_dispatched = false`

#### Scenario: Tampered or expired token
- **WHEN** `verifyInviteToken` receives a token whose signature does not match or whose `exp` is in the past
- **THEN** it returns `null`

### Requirement: Reinviting renews the pending row
When the service-role client is configured, `emitirConvite` SHALL renew an existing pending `team_invites` row for the same organization and email (new `expires_at`, `resend_count + 1`) instead of inserting a duplicate, and otherwise insert a new row whose `id` is the token's `invite_id`.

#### Scenario: Reinvite a pending email
- **WHEN** an admin invites an email that already has a pending invite in the same organization
- **THEN** the same `team_invites.id` is returned with `resend_count` incremented

### Requirement: Listing invites with derived status
`GET /api/v1/team/invites` SHALL require `requireRole("manager")`, return the active organization's `team_invites` ordered by `created_at desc`, each with a derived `status` and an `accept_url` only while the invite is open.

#### Scenario: Accepted invite in the list
- **WHEN** a manager lists invites and one row has `accepted_at` set
- **THEN** that item has `accept_url = null`

### Requirement: Resend and revoke an invite
`POST /api/v1/team/invites/[id]/resend` and `POST /api/v1/team/invites/[id]/revoke` SHALL call `requireSupportWrite()` and `requireRole("admin")`, respond HTTP 400 `invalid_request` for a non-UUID id, HTTP 404 `not_found` when the invite is not in the active organization, and HTTP 409 `state_conflict` when it was already accepted; resend SHALL respond HTTP 503 `unavailable` when the service role is not configured and HTTP 409 when revoked, and revoke SHALL set `revoked_at`/`revoked_by`, audit `member.invite_revoked`, and return `already_revoked: true` idempotently.

#### Scenario: Revoke an accepted invite
- **WHEN** an admin revokes an invite whose `accepted_at` is set
- **THEN** the response is HTTP 409 `state_conflict`

#### Scenario: Revoke twice
- **WHEN** an admin revokes an invite that is already revoked
- **THEN** the response is HTTP 200 with `data.already_revoked = true`

### Requirement: Accepting an invite
`acceptInviteAction(token)` SHALL return `invalid_or_expired` for an invalid token, `not_authenticated` without a session, `email_mismatch` (with `expectedEmail`) when the logged-in email differs case-insensitively from the token email, refuse during a support session, and otherwise `aplicarConvite` SHALL refuse a revoked `team_invites` row, call `fn_accept_team_invite` through the service role (mapping SQLSTATE `42501` to `invalid_or_expired` and SQLSTATE `PT402` from the plan seat trigger to `limite_do_plano`), audit `member.accepted` when the membership changed, set `team_invites.accepted_at`/`accepted_by`, set cookie `active_org` to the invite's organization, and redirect to `/app`.

#### Scenario: Wrong account accepts
- **WHEN** a user logged in as `a@x.com` accepts a token issued to `b@x.com`
- **THEN** the action returns `error = "email_mismatch"` and no `user_organizations` row is created

#### Scenario: Revoked invite
- **WHEN** a valid, unexpired token is accepted but its `team_invites` row has `revoked_at` set
- **THEN** the action returns `error = "invalid_or_expired"`

#### Scenario: Successful acceptance
- **WHEN** the invited user accepts a valid open invite
- **THEN** a non-revoked `user_organizations` row exists with the invite's role, `team_invites.accepted_at` is set, cookie `active_org` holds the organization id, and the response redirects to `/app`

#### Scenario: Organization filled its seats after the invite
- **WHEN** a valid open invite is accepted while the organization's plan seat ceiling is already reached
- **THEN** the action returns `error = "limite_do_plano"` and no `user_organizations` row is created

### Requirement: Revoking and reactivating members
`POST /api/v1/team/[user_id]/revoke` SHALL require `requireSupportWrite()` and `requireRole("admin")`, respond HTTP 409 `state_conflict` when the target is the caller or the last non-revoked `admin`, HTTP 404 `not_found` when not a member, return `already_revoked: true` for an already revoked member, and otherwise set `user_organizations.revoked_at`, audit `conversation.released` per open conversation assigned to the target and `member.revoked`; `POST /api/v1/team/[user_id]/reactivate` SHALL clear `revoked_at`, audit `member.reactivated`, return `already_active: true` for an active member, and answer HTTP 409 `plan_limit_reached` when the plan seat trigger refuses the reactivation.

#### Scenario: Admin revokes themself
- **WHEN** an admin calls `POST /api/v1/team/<own id>/revoke`
- **THEN** the response is HTTP 409 `state_conflict`

#### Scenario: Revoke the last admin
- **WHEN** a caller with effective role `admin` (e.g. a full-access support session) revokes the only non-revoked `admin` member of the organization
- **THEN** the response is HTTP 409 `state_conflict` and `revoked_at` stays null

#### Scenario: Reactivate a revoked member
- **WHEN** an admin reactivates a member whose `revoked_at` is set
- **THEN** `revoked_at` becomes null, the original role is kept, and the response contains `reactivated_at`

#### Scenario: Reactivation beyond the seat ceiling
- **WHEN** an admin reactivates a revoked member while the plan seat ceiling is already reached
- **THEN** the response is HTTP 409 `plan_limit_reached` and `revoked_at` stays set

### Requirement: Team listing permissions
`GET /api/v1/team` SHALL require `requireRole("manager")` and `GET /api/v1/team/assignable` SHALL require `requireRole("agent")`, both scoped to the active organization, and the page `app/app/team/invite/page.tsx` SHALL redirect to `/403` when the active role ranks below `admin`.

#### Scenario: Viewer lists assignable members
- **WHEN** a `viewer` calls `GET /api/v1/team/assignable`
- **THEN** the response is HTTP 403 `forbidden_role`

#### Scenario: Manager opens the invite page
- **WHEN** a `manager` requests `/app/team/invite`
- **THEN** the response redirects to `/403`
