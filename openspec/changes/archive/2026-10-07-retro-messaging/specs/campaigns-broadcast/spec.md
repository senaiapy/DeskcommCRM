## Purpose
Campaigns are proactive WhatsApp broadcasts to a frozen list of contacts through one or more channel numbers, paced more restrictively than the channel itself. Key paths: pages `app/app/campaigns` (list, `new`, `[id]`, `settings`); API `app/api/v1/campaigns` (GET/POST), `campaigns/[id]` (GET/PATCH), `campaigns/[id]/[acao]` (POST preparar|iniciar|agendar|pausar|retomar|cancelar|duplicar|testar), `campaigns/[id]/recipients`, `campaigns/[id]/metrics`, `campaigns/preview`, `campaign-suppressions` (GET/POST, `[id]` DELETE), `settings/campanhas` (GET/PATCH); libs `lib/campanhas/*` (state machine `maquina-de-estados.ts`, pacing `ritmo.ts`, dispatch `rodada.ts`, eligibility `elegibilidade.ts`, suppressions `exclusoes.ts`); tables `campaigns`, `campaign_recipients`, `campaign_suppressions`, `campaign_templates`, `campaign_channel_sessions`; cron `app/api/v1/cron/campaign-worker` scheduled every minute by `docker/scheduler/entrypoint.sh`.

## ADDED Requirements

### Requirement: Manager-only campaign API
Every handler under `/api/v1/campaigns`, `/api/v1/campaign-suppressions` and `/api/v1/settings/campanhas` SHALL call `requireRole("manager")`, and every mutating handler SHALL call `requireSupportWrite()` before any effect, writing with the admin client filtered by the `organization_id` resolved from the session, never from the body.

#### Scenario: Agent cannot list campaigns
- **WHEN** a user whose role in the active organization is `agent` calls `GET /api/v1/campaigns`
- **THEN** the response is the `requireRole` denial and no `campaigns` row is read

#### Scenario: Draft creation is scoped to the caller's organization
- **WHEN** a manager calls `POST /api/v1/campaigns` with a valid body
- **THEN** the response is 201 and the inserted `campaigns` row has `organization_id` equal to the manager's active organization and `status = 'draft'`

### Requirement: Channel ownership on creation
`POST /api/v1/campaigns` SHALL reject a `channel_session_id` that does not belong to the caller's organization with HTTP 409 and error code `campanha_canal_indisponivel`, backed by the composite FK `campaigns_channel_org_fk` on `(organization_id, channel_session_id)`.

#### Scenario: Foreign channel number
- **WHEN** a manager creates a campaign with a `channel_session_id` of another organization
- **THEN** the response is 409 with code `campanha_canal_indisponivel` and no `campaigns` row is inserted

### Requirement: Campaign state machine
Campaign status transitions SHALL follow `PERMITIDO` in `lib/campanhas/maquina-de-estados.ts` over the nine statuses allowed by `campaigns_status_check` (`draft, preparing, ready, scheduled, running, paused, completed, cancelled, failed`), with `completed` and `cancelled` terminal, and a disallowed transition answered with HTTP 409 `campanha_estado_invalido`.

#### Scenario: Cancelled campaign cannot restart
- **WHEN** `POST /api/v1/campaigns/{id}/iniciar` is called for a campaign in `cancelled`
- **THEN** the response is 409 with code `campanha_estado_invalido` and `campaigns.status` stays `cancelled`

#### Scenario: Draft can be cancelled
- **WHEN** `POST /api/v1/campaigns/{id}/cancelar` is called for a campaign in `draft`
- **THEN** the response is 200, `campaigns.status = 'cancelled'` with `cancelled_at` set, and recipients still `pending`/`queued` become `cancelled`

#### Scenario: Unknown action
- **WHEN** `POST /api/v1/campaigns/{id}/explodir` is called
- **THEN** the response is 404 with code `not_found`

### Requirement: Edit permissions by field class
`PATCH /api/v1/campaigns/{id}` SHALL return 409 `campanha_nao_editavel` for any change to a campaign in a terminal status, and for content/audience changes when the status is not `draft`, while pacing fields (`channel_session_ids`, `intervalo_segundos`, `janela_inicio_hora`, `janela_fim_hora`, `teto_diario`, `teto_horario`) remain editable in any live status.

#### Scenario: Slow down a running campaign
- **WHEN** a manager PATCHes only `intervalo_segundos` on a `running` campaign
- **THEN** the change is persisted on `campaigns.intervalo_segundos`

#### Scenario: Change text of a running campaign
- **WHEN** a manager PATCHes `message_body` on a `running` campaign
- **THEN** the response is 409 with code `campanha_nao_editavel`

### Requirement: Declared legal basis
Every campaign SHALL declare `base_legal` in (`consent`, `legitimate_interest`), and `legitimate_interest` SHALL require a non-blank `lia_ref`, enforced by the `campaigns_lia_exige_ref` check and by the API returning 422 `campanha_base_legal_invalida`.

#### Scenario: Legitimate interest without LIA
- **WHEN** a PATCH sets `base_legal = 'legitimate_interest'` with an empty `lia_ref`
- **THEN** the response is 422 with code `campanha_base_legal_invalida`

### Requirement: Frozen recipient snapshot with exclusion reasons
Preparation (`POST /api/v1/campaigns/{id}/preparar`) SHALL materialize `campaign_recipients` rows with `status = 'pending'`/`eligibility_status = 'eligible'` for eligible contacts and `status = 'skipped'`/`eligibility_status = 'excluded'` with an `exclusion_reason` code (e.g. `opt_out`, `suprimido`, `duplicado`, `ja_em_campanha`, `variavel_ausente`) otherwise, under the unique constraints `campaign_recipients_contato_unico (campaign_id, contact_id)` and `campaign_recipients_endereco_unico (campaign_id, recipient_address)`.

#### Scenario: Blocked contact excluded at preparation
- **WHEN** a campaign is prepared and the audience contains a contact with `contacts.is_blocked = true`
- **THEN** that contact's `campaign_recipients` row has `eligibility_status = 'excluded'`, `exclusion_reason = 'opt_out'`, `status = 'skipped'` and `recipient_address` null

#### Scenario: Start without eligible recipients
- **WHEN** `POST /api/v1/campaigns/{id}/iniciar` is called and no recipient has `eligibility_status = 'eligible'`
- **THEN** the response is 422 with code `campanha_sem_elegiveis`

### Requirement: Send-time re-validation
Before sending, the campaign worker SHALL re-check the pending recipient's contact (opt-out, personal, anonymized, marketing refusal) and the current `campaign_suppressions` hash, marking the recipient excluded (`opted_out` for opt-out, `skipped` with `exclusion_reason = 'suprimido'` for a suppressed number) instead of sending.

#### Scenario: Number suppressed after preparation
- **WHEN** a number is added to `campaign_suppressions` after the campaign was prepared and the worker reaches that recipient
- **THEN** no message is sent and the recipient row gets `status = 'skipped'`, `exclusion_reason = 'suprimido'`

### Requirement: Operator suppression list stored as hash
`POST /api/v1/campaign-suppressions` SHALL store only `recipient_address_hash` (SHA-256 of the normalized E.164 number) and `address_tail` in `campaign_suppressions`, unique per `(organization_id, recipient_address_hash)`, answering 201 on insert and 200 with `{ ja_estava: true }` on a duplicate, and preparation SHALL exclude matching numbers with reason `suprimido`.

#### Scenario: Suppressing the same number twice
- **WHEN** a manager posts an address already present in the organization's suppression list
- **THEN** the response is 200 with `data.ja_estava = true` and no second row is inserted

#### Scenario: Invalid phone
- **WHEN** a manager posts an address that is not a valid send-format number
- **THEN** the response is 422 with code `validation_failed`

### Requirement: Scheduling
`POST /api/v1/campaigns/{id}/agendar` SHALL set `status = 'scheduled'` and `scheduled_at` only for a future instant (otherwise 422 `campanha_agenda_invalida`), and the campaign worker SHALL promote `scheduled` campaigns whose `scheduled_at <= now()` of operating organizations to `running` with `started_at` set.

#### Scenario: Past schedule date
- **WHEN** `agendar` is called with `scheduled_at` in the past
- **THEN** the response is 422 with code `campanha_agenda_invalida`

#### Scenario: Due schedule promoted
- **WHEN** the cron `campaign-worker` runs after a campaign's `scheduled_at`
- **THEN** that campaign's `status` becomes `running`

### Requirement: Paced dispatch by cron
The cron `GET|POST /api/v1/cron/campaign-worker` SHALL authenticate with `autorizaCron` (Bearer `INTERNAL_CRON_SECRET`/`INTERNAL_SECRET`, 403 `forbidden` otherwise) and send at most one message per channel number per round, only when both the channel pacing engine (`decidePacing` over `pacing_ledger`, `channel_knobs`, `channel_sessions.daily_message_limit`) and the campaign's own pace (`podeMandarAgora`: `teto_diario`, `teto_horario`, `janela_inicio_hora`/`janela_fim_hora` in the main number's timezone, `intervalo_segundos`) allow it.

#### Scenario: Cron without secret
- **WHEN** `campaign-worker` is called without a valid cron bearer
- **THEN** the response is 403 with code `forbidden`

#### Scenario: Outside the campaign window
- **WHEN** the round runs at a local hour outside `[janela_inicio_hora, janela_fim_hora)`
- **THEN** no message is sent, the pending recipient stays `pending` with `next_attempt_at` set, and the round detail is `ritmo:fora_da_janela`

### Requirement: Recipient send lifecycle and completion
The worker SHALL reserve a recipient by moving it `pending -> sending` (compare-and-set on `status = 'pending'`), record `sent`/`failed` from the send outcome, and set the campaign to `completed` with `completed_at` when no recipient remains in `pending`, `queued` or `sending`.

#### Scenario: Last recipient sent
- **WHEN** the worker sends the last pending recipient of a `running` campaign
- **THEN** on a subsequent round the campaign's `status` becomes `completed` and `completed_at` is set

### Requirement: Organization campaign defaults
`GET /api/v1/settings/campanhas` SHALL return `{ configuracao, padrao }` read from `organizations.settings.campanhas`, and `PATCH` SHALL merge the validated object into `organizations.settings` under the `campanhas` key without overwriting other keys, rejecting an end hour not after the start hour with 422 `validation_failed`.

#### Scenario: Invalid window in defaults
- **WHEN** a manager PATCHes a default window whose end is not after its start
- **THEN** the response is 422 with code `validation_failed` and `organizations.settings` is unchanged
