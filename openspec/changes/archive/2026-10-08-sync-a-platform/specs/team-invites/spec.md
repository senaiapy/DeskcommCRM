## MODIFIED Requirements

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
