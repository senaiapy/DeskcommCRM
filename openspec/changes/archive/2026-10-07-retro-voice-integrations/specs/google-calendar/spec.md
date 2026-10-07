## Purpose
Per-user Google Calendar connection for the agenda: each member authorizes their own Google account by OAuth, chooses
which calendars count as busy time and which receives CRM appointments, and three crons keep it alive — token refresh
(`agenda-google-refresh`), inbound incremental sync of calendars (`agenda-google-sync`) and outbound reconciliation of
appointments with Google events and Meet links (`agenda-google-push`). Generic scheduling (appointment types,
availability, reminders, journeys) is covered by `agenda-scheduling`; the Google Ads OAuth connection by `ads-attribution`.

## ADDED Requirements

### Requirement: OAuth app configuration source
`configuracaoDoGoogle` (lib/agenda/google/config.ts) SHALL use the client id and encrypted secret stored in `platform_google_oauth` when both exist and decrypt, and otherwise fall back entirely to `GOOGLE_CALENDAR_CLIENT_ID` and `GOOGLE_CALENDAR_CLIENT_SECRET`, with redirect URI `/api/v1/agenda/google/callback`; with neither, connect SHALL redirect to `/app/agenda?erro=google_nao_configurado`.

#### Scenario: Stored secret fails to decrypt
- **WHEN** the `platform_google_oauth` secret cannot be decrypted
- **THEN** the environment pair is used whole, never mixing the stored client id with the environment secret

### Requirement: Connect binds state to the browser
`GET /api/v1/agenda/google/connect` SHALL require role `agent`, issue a `state` signed with `INTERNAL_SECRET` carrying organization, user, auth session and a random nonce, set a `sameSite: lax` binding cookie derived from the same nonce, and redirect to Google consent with the user's e-mail as suggested account.

#### Scenario: Weak state secret
- **WHEN** `INTERNAL_SECRET` is too short for `emitirEstado`
- **THEN** the user is redirected to `/app/agenda?erro=segredo_indisponivel` and `agenda.google.conexao_falhou` is audited

### Requirement: Callback verification and single-use state
`GET /api/v1/agenda/google/callback` SHALL verify the signed `state` and the binding cookie, record the nonce in `calendar_oauth_nonces` (unique violation 23505 means `state_reutilizado`), exchange the code only after these checks, and require all mandatory scopes, answering on failure with a bridge page back to `/app/agenda?erro=<code>` (`conexao_cancelada`, `retorno_nao_verificavel`, `troca_de_codigo_falhou`, `permissao_incompleta`, `sem_token_de_renovacao`, `cifra_indisponivel`, `nao_consegui_guardar`).

#### Scenario: Replayed callback
- **WHEN** the same `state` reaches the callback a second time
- **THEN** the nonce insert fails with 23505 and the user returns with `erro=retorno_nao_verificavel` without a code exchange

#### Scenario: User unchecks a calendar scope
- **WHEN** Google returns a token missing a required scope
- **THEN** the user returns with `erro=permissao_incompleta` and no connection is stored

### Requirement: Encrypted per-user connection row
A verified callback SHALL upsert `calendar_connections` on `(organization_id, user_id, provider, account_email)` with `provider='google_calendar'`, `status='healthy'`, encrypted `oauth_access_token_encrypted`/`oauth_refresh_token_encrypted`, and SHALL refuse with `sem_token_de_renovacao` when Google returns no refresh token and none is already stored for that account.

#### Scenario: Re-consent without refresh token for a known account
- **WHEN** the account already has a stored refresh token and Google omits `refresh_token`
- **THEN** the connection is updated keeping the stored refresh token

### Requirement: Calendar selection
`GET /api/v1/agenda/google/calendarios` SHALL require role `agent` and list the caller's connections and `calendar_connection_calendars`, and `PATCH` SHALL save sources and destination through `fn_google_selection` with the active `organization_id`, answering 409 `conflict` when the selection revision is stale (SQLSTATE 40001) and 403 `forbidden` for calendars outside the caller's connection.

#### Scenario: Stale selection
- **WHEN** the calendar catalog changed after the screen loaded and the user saves
- **THEN** the response is 409 `conflict`

### Requirement: Disconnect wipes local Google data
`DELETE /api/v1/agenda/google/desconectar` SHALL require role `agent` for the caller's own connection and `manager` when `user_id` names another member, answer 404 `not_found` when no Google connection exists, delete that user's `calendar_external_events` and `calendar_connection_calendars`, and set `calendar_connections.status='disconnected'` with tokens, `token_expires_at` and `sync_token` nulled.

#### Scenario: Agent disconnects a colleague
- **WHEN** an agent sends `user_id` of another member
- **THEN** the response is 403 and the colleague's connection is untouched

### Requirement: Token refresh cron
`/api/v1/cron/agenda-google-refresh` (GET or POST) SHALL require the cron bearer (`INTERNAL_CRON_SECRET` or `INTERNAL_SECRET`, else 401 `unauthenticated`), renew at most 50 `healthy`/`rate_limited` connections whose `token_expires_at` falls within 15 minutes, store the new encrypted access token with `status='healthy'`, and mark `token_expired` when no refresh token is stored or Google classifies the failure as requiring re-authentication.

#### Scenario: Revoked grant
- **WHEN** Google refuses the refresh token as revoked
- **THEN** the connection status becomes `token_expired` with a `last_sync_error` message

### Requirement: Inbound sync cron
`/api/v1/cron/agenda-google-sync` SHALL require the cron bearer, process at most 25 `healthy` Google connections and 25 calendars per run, refresh a connection's calendar catalog when older than 24 hours, sync incrementally with the stored sync token (resetting on HTTP 410), defer by 24 hours calendars that are neither busy-source nor destination and have no linked appointment, and write `last_sync_error` on failure.

#### Scenario: Sync token expired at Google
- **WHEN** Google answers 410 for an incremental sync
- **THEN** the calendar sync is reset and restarted from a full listing

### Requirement: Outbound appointment reconciliation cron
`/api/v1/cron/agenda-google-push` SHALL require the cron bearer, read pending appointments from `googlePushCandidates`, and call `reconcileAppointment`, which creates, patches or deletes the Google event at `https://www.googleapis.com/calendar/v3` with `sendUpdates=all` (and `conferenceDataVersion=1` for Meet), auditing `agenda.google.sync_executado` per organization with processed and failed counts.

#### Scenario: Reconciliation throws
- **WHEN** `reconcileAppointment` throws for one appointment
- **THEN** it is counted as a failure and the remaining candidates are still processed

### Requirement: Revoked members are never synced
All three crons SHALL pass connections through `apenasDeMembrosAtivos`, skipping any `user_id` whose `user_organizations` link to that organization is revoked.

#### Scenario: Employee removed from the organization
- **WHEN** a member with a healthy Google connection has `revoked_at` set
- **THEN** the refresh, sync and push crons no longer read or write that member's calendar
