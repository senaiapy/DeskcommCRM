# agenda-scheduling Specification

## Purpose
Internal scheduling: appointment types, per-person working hours and exceptions, free-slot computation, booking/rescheduling/cancelling appointments for contacts (by the team, by the AI agent through MCP tools sharing the same handlers, or by server integrations with a `dsk_` bearer token), reminders sent over the conversation channel, expiry of unconfirmed requests, and Google Calendar/Meet sync as used by the agenda. Page `app/app/agenda`. API `app/api/v1/agenda/{agendamentos,agendamentos/[id],agendamentos/[id]/google/*,tipos,tipos/reativar,horarios-livres,configuracao,excecoes,enderecos,pessoas,vinculos,google/connect,google/callback,google/desconectar,google/calendarios,google/calendarios/atualizar}`; decision logic in `app/api/v1/agenda/agendamentos/_handler.ts` and `lib/agenda/*` (`consulta.ts`, `horarios-livres.ts`, `ocupados.ts`, `lembretes.ts`, `google/*`). Tables `calendar_event_types`, `calendar_appointments`, `calendar_availability_exceptions`, `calendar_locations`, `calendar_connections`, `calendar_connection_calendars`, `calendar_external_events`, `calendar_oauth_nonces`. Crons `app/api/v1/cron/{agenda-reminder,agenda-expira-pendentes,agenda-google-refresh,agenda-google-sync,agenda-google-push}`. There is no public self-booking page: `calendar_event_types.slug` is explicitly not a public URL, although `calendar_appointments.source` allows the value `public_page`.

## Requirements

### Requirement: Dual authentication for appointment mutations
`POST`, `PATCH` and `DELETE /api/v1/agenda/agendamentos` SHALL call `requireSupportWrite` and accept either a session with role `agent` or a `dsk_` bearer token with scope `mcp:write` (evaluated as role `ai_operator`), taking `organization_id` from the validated cookie or token row and never from the body, while `GET` SHALL accept only a session with role `viewer`.

#### Scenario: Viewer tries to book
- **WHEN** a session with role `viewer` posts to `/api/v1/agenda/agendamentos`
- **THEN** the response is 403 and no `calendar_appointments` row is written

#### Scenario: Server integration books with a token
- **WHEN** a request carries `Authorization: Bearer dsk_...` with scope `mcp:write` and a valid body
- **THEN** the appointment is created in the token's organization with `created_by_kind` derived from the actor and `source = 'mcp'`

### Requirement: Idempotent booking
`POST /api/v1/agenda/agendamentos` SHALL accept an optional `Idempotency-Key` header that must be a UUID, and replay the stored response for a repeated key with the same body via `comIdempotencia`.

#### Scenario: Malformed key
- **WHEN** `Idempotency-Key` is not a UUID
- **THEN** the response is 400 `validation_failed`

#### Scenario: Same key, different body
- **WHEN** a key already used with another body is sent
- **THEN** the response is 409 `idempotency_conflict`

#### Scenario: Same request still running
- **WHEN** a request with the same key is still in progress
- **THEN** the response is 409 `idempotency_in_progress`

### Requirement: Booking respects the type and the availability grid
The booking handler SHALL refuse an inactive type with 422 `agenda_tipo_desativado`, an owner without published hours with 422 `agenda_fora_da_jornada`, and, for non-user actors (AI agent, API token), a start time that is not one of the computed free slots with 422 `agenda_horario_indisponivel`; user actors MAY book outside the grid but SHALL be refused with 422 `agenda_horario_indisponivel` when the interval overlaps another occupying appointment or a busy Google Calendar event of the owner.

#### Scenario: Agent books an unoffered slot
- **WHEN** the AI agent books 10:07 while the free slots are 10:00 and 10:30
- **THEN** the response is 422 `agenda_horario_indisponivel`

#### Scenario: Team member squeezes in over a busy Google event
- **WHEN** a user books an interval that overlaps a busy event synced from the owner's Google Calendar
- **THEN** the response is 422 `agenda_horario_indisponivel`

#### Scenario: Type requires confirmation
- **WHEN** a booking succeeds for a type with `requires_confirmation = true`
- **THEN** the response is 201 and the row has `status = 'pending'`; otherwise `status = 'confirmed'`

### Requirement: Appointment status and optimistic concurrency
`calendar_appointments.status` SHALL be constrained to `pending`, `confirmed`, `cancelled`, `completed`, `no_show`, and PATCH/DELETE SHALL return 409 `conflict` when the body `revision` differs from the stored one.

#### Scenario: Stale revision
- **WHEN** PATCH is sent with an outdated `revision`
- **THEN** the response is 409 `conflict` and the row is unchanged

#### Scenario: Editing a cancelled appointment
- **WHEN** PATCH targets an appointment with `status = 'cancelled'`
- **THEN** the response is 422 `agenda_ja_cancelado`

#### Scenario: Cancelling twice
- **WHEN** DELETE targets an already cancelled appointment
- **THEN** the response is 200 with `status: "cancelled"` and `ja_estava: true`

### Requirement: Listing requires a scope
`GET /api/v1/agenda/agendamentos` SHALL require at least one scoping filter (`contact_id`, `lead_id`, `owner_user_id`, `dia` or a `de`/`ate` window) and return 422 `agenda_listagem_sem_recorte` otherwise, and 422 `agenda_listagem_alvo_nao_e_lead` when `lead_id` is not a deal of the organization.

#### Scenario: Unscoped listing
- **WHEN** a viewer calls GET without any filter
- **THEN** the response is 422 `agenda_listagem_sem_recorte`

### Requirement: Free-slot query
`GET /api/v1/agenda/horarios-livres` SHALL require role `viewer`, the query parameters `event_type_id`, `de`, `ate` (ISO-8601 with offset) and optional `owner_user_id`, and return `slots[{inicio,fim}]`, `fuso_da_regra`, `publicou_horarios`, `fuso_suposto` and `fontes_defasadas`.

#### Scenario: Inverted or too long window
- **WHEN** `ate` is not after `de` or the window exceeds `MAXIMO_DE_DIAS`
- **THEN** the response is 422 `validation_failed`

### Requirement: Appointment types managed by managers
`/api/v1/agenda/tipos` SHALL allow `viewer` to GET and require `manager` (session, or token with scope `mcp:write`) for POST/PATCH/DELETE, deriving a per-organization unique `slug` from the name, and DELETE SHALL set `is_active = false` instead of removing the row.

#### Scenario: Duplicate type name
- **WHEN** POST creates a type whose slug already exists in the organization
- **THEN** the response is 409 `conflict`

#### Scenario: Deactivating a type
- **WHEN** a manager DELETEs a type
- **THEN** the row stays with `is_active = false` and can be reactivated by `POST /api/v1/agenda/tipos/reativar`

### Requirement: Reminders over the conversation channel
`/api/v1/cron/agenda-reminder` SHALL, when authorized by `INTERNAL_CRON_SECRET` or `INTERNAL_SECRET`, select `confirmed` future appointments with a contact whose type has `reminder_enabled = true` in operating organizations, stamp `reminder_sent_at` and `reminder_sent_offsets_minutes` before sending, and skip the send when the stamp fails.

#### Scenario: Stamp fails
- **WHEN** the update of `reminder_sent_at` returns an error
- **THEN** no reminder message is sent for that appointment in this run

#### Scenario: Already sent offset
- **WHEN** an offset is already listed in `reminder_sent_offsets_minutes`
- **THEN** that offset is not sent again

### Requirement: Unconfirmed requests expire
`/api/v1/cron/agenda-expira-pendentes` SHALL set `status = 'cancelled'` with a `cancellation_reason` on `pending` appointments older than the organization's `settings.agenda.pending_expires_after_minutes` (default 1440, range 15–10080), updating only rows still `pending`.

#### Scenario: Confirmed in the meantime
- **WHEN** an appointment is confirmed between the cron's read and its update
- **THEN** the update filtered by `status = 'pending'` does not touch it

#### Scenario: Unauthorized call
- **WHEN** the cron is called without a valid cron secret
- **THEN** the response is 403 `forbidden`

### Requirement: Google Calendar connection for the agenda
`GET /api/v1/agenda/google/connect` SHALL require role `agent` and redirect to Google consent with a signed state, or redirect to `/app/agenda?erro=google_nao_configurado` when `GOOGLE_CALENDAR_CLIENT_ID` or `GOOGLE_CALENDAR_CLIENT_SECRET` is empty, and `calendar_connections` SHALL store OAuth tokens only in the `bytea` columns `oauth_access_token_encrypted` and `oauth_refresh_token_encrypted` with `status` in `connecting`, `healthy`, `token_expired`, `scope_missing`, `disconnected`, `rate_limited`, `error`.

#### Scenario: Google not configured
- **WHEN** an agent opens the connect route on an installation without Google credentials
- **THEN** the response is a redirect to `/app/agenda?erro=google_nao_configurado`

#### Scenario: Sync of healthy connections
- **WHEN** `/api/v1/cron/agenda-google-sync` runs with a valid cron secret
- **THEN** it reads at most 25 `calendar_connections` with `status = 'healthy'` and `provider = 'google_calendar'` belonging to active members and syncs their selected calendars into `calendar_external_events`
