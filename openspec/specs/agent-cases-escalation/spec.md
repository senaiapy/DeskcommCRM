# agent-cases-escalation Specification

## Purpose
Agent cases and escalation is the loop that keeps a conversation the AI cannot finish from dying: the agent's notice center (`agent_inbox_items`, `/app/ai/inbox`), human cases opened by the AI (`agent_cases`, `/app/ai/cases`), the handoff orchestrator (including the low-sentiment trigger), the automatic return of stale handoffs to the AI, the stale-case reminder cron and the next-step/closing of `demandas`. The AI turn itself is covered by `ai-agents-runtime`; queue assignment between attendants by `routing-attendants`; the conversation inbox by `inbox-conversations`; the event bus that delivers `ai.sentiment_alert` by `event-bus-workers`.

## Requirements

### Requirement: Notice center listing
`GET /api/v1/ai/inbox` SHALL require role `agent`, read `agent_inbox_items` filtered by the active organization (never rows with `organization_id` null), accept `status` in `open|ack|resolved|all` (default `open`) and `limit` 1–200 (default 50), return open items layered by severity `critical`, `warn`, `info` and include `open_count`, answering 422 `validation_failed` on an invalid query.

#### Scenario: Critical notice is never hidden by newer ones
- **WHEN** an organization has one old `critical` open item and more than `limit` newer `info` open items
- **THEN** the response lists the `critical` item first and `open_count` counts every open item of the organization

#### Scenario: Invalid status filter
- **WHEN** the query is `?status=deleted`
- **THEN** the response is HTTP 422 with code `validation_failed`

### Requirement: Notice status transitions
`PATCH /api/v1/ai/inbox/{id}` SHALL accept a strict body `{status: "open"|"ack"|"resolved"}` from role `agent`, update only the row of the active organization, audit `ai.inbox_item_status_changed`, and answer 400 `invalid_request` for a non-UUID id, 404 `not_found` when no row matched and 409 `state_conflict` when reopening would collide with an identical open notice (unique violation `23505`).

#### Scenario: Reopen blocked by duplicate
- **WHEN** an agent sets a resolved notice back to `open` while an identical notice is already open in the organization
- **THEN** the response is HTTP 409 `state_conflict` and the row stays resolved

#### Scenario: Notice of another organization
- **WHEN** the id belongs to a notice of a different organization
- **THEN** the response is HTTP 404 `not_found`

### Requirement: Bulk resolve of open notices
`POST /api/v1/ai/inbox/resolve-all` SHALL require role `agent`, set `status='resolved'` on every `open` row of `agent_inbox_items` of the active organization only, return `resolved_count`, and write the `ai.inbox_item_status_changed` audit (with `bulk: true`, `count` and a null `resource_id`) only when at least one row changed.

#### Scenario: Nothing open
- **WHEN** the organization has no open notices
- **THEN** the response is `{resolved_count: 0}` and no audit row is written

### Requirement: Human case listing and detail
`GET /api/v1/ai/cases` (query `status` `open|resolved`, default `open`) and `GET /api/v1/ai/cases/{id}` SHALL require role `agent`, read `agent_cases` of the active organization restricted to the conversations the caller's session can see under RLS, and answer 404 `not_found` for a case that is absent or not visible.

#### Scenario: Case outside the agent's visible conversations
- **WHEN** an `agent` requests a case whose conversation RLS hides from that user
- **THEN** the response is HTTP 404 `not_found`

### Requirement: Case state machine on reply
`POST /api/v1/ai/cases/{id}/reply` SHALL require role `agent`, accept a strict body `{action: "resolved"|"need_lead_info"|"escalate", body: 1..4000 chars}`, act only on a case in `awaiting_human`, move it to `resolved`, `awaiting_lead` or `escalated` respectively, enqueue a `case_reply_turn` job in the same transaction for `resolved`/`need_lead_info`, audit `ai.case_replied`, and answer 409 `invalid_state` when the case is not `awaiting_human` or another person answered first.

#### Scenario: Answer that unblocks the AI
- **WHEN** an agent posts `{action:"need_lead_info", body:"..."}` on an `awaiting_human` case
- **THEN** `agent_cases.status` becomes `awaiting_lead`, a `job_queue` row of kind `case_reply_turn` exists for the case contact, and the response is `{status:"awaiting_lead"}`

#### Scenario: Case already answered
- **WHEN** the case is already `resolved`
- **THEN** the response is HTTP 409 `invalid_state` and no job is enqueued

### Requirement: Internal case chat with the AI
`GET` and `POST /api/v1/ai/cases/{id}/chat` SHALL require role `agent`, answer 404 when the case conversation is not visible to the caller's session, store turns in `agent_case_chat_messages` deduplicated by `turn_id`, rate-limit questions at 12 per user and 60 per organization per 60 s and 120 human questions per case per 24 h (HTTP 429 `rate_limited`), and refuse with 422 `reply_context_unavailable` when the case contact is anonymized.

#### Scenario: Anonymized contact
- **WHEN** an agent asks about a case whose contact has `is_anonymized=true`
- **THEN** the response is HTTP 422 `reply_context_unavailable`, no model is called and no chat row is written

#### Scenario: Per-case ceiling
- **WHEN** the case already has 120 human chat rows in the last 24 hours
- **THEN** the response is HTTP 429 `rate_limited` with `Retry-After: 3600`

### Requirement: Team WhatsApp alert configuration
`GET` and `PUT /api/v1/ai/cases/alerta` SHALL require role `admin`, and `PUT` SHALL validate a strict body (`channel_session_id` uuid, E.164 `telefone`, `ligado` boolean, optional `rotulo`, `confirma_contato`), persist through `fn_definir_aviso_de_caso`, map its refusals to 403 `forbidden_role`/`mfa_required` or 422 `validation_failed`/`aviso_canal_invalido`/`aviso_numero_da_propria_org`/`aviso_numero_de_cliente`, and audit `ai.case_alert_settings_changed` with the phone masked.

#### Scenario: Organization's own number
- **WHEN** an admin sets as alert destination a number that is one of the organization's connected channels
- **THEN** the response is HTTP 422 `aviso_numero_da_propria_org`

### Requirement: Central handoff orchestrator
`triggerHandoff` in `lib/ai/handoff/orchestrator.ts` SHALL set `conversations.bot_silenced_until='infinity'`, `last_handoff_at` and `last_handoff_reason`, emit `ai.handoff_triggered` into `event_log`, insert an `api_audit_log` row with action `ai.handoff_triggered`, and skip with reason `idempotent_5s` when the same reason fired on the conversation less than 5 seconds earlier.

#### Scenario: Duplicate trigger within 5 seconds
- **WHEN** two triggers with reason `low_sentiment` hit the same conversation 2 seconds apart
- **THEN** only the first silences the bot and emits `ai.handoff_triggered`; the second returns `triggered:false` with reason `idempotent_5s`

### Requirement: Low-sentiment handoff
The `ai-sentiment-worker.v1` consumer of `message.received` SHALL score the inbound message, emit `ai.sentiment_alert` when the score is below the published agent's `sentiment_threshold` (default `0.3`), and the `ai-handoff-from-sentiment.v1` consumer SHALL call `triggerHandoff` with reason `low_sentiment` for the message's conversation, skipping with `service_boundary_stale` when the message no longer belongs to that conversation.

#### Scenario: Angry customer
- **WHEN** an inbound message scores 0.1 and the agent has no custom threshold
- **THEN** an `ai.sentiment_alert` event is emitted and the conversation gets `last_handoff_reason='low_sentiment'`

### Requirement: Automatic return of stale handoffs
`GET|POST /api/v1/cron/handoff-devolucao` SHALL require `INTERNAL_CRON_SECRET` or `INTERNAL_SECRET` (HTTP 403 `forbidden` otherwise), consider only organizations with `settings.routing.handoff_return_after_minutes` set (5–1440 minutes), scan at most 500 non-group conversations held by a human, return the overdue ones through `devolverAtendimentoAoAgente`, and audit `conversation.handoff_auto_return_run` only when something was returned or failed.

#### Scenario: Organization without the deadline
- **WHEN** no organization has `handoff_return_after_minutes` configured
- **THEN** the run returns `{organizacoes:0, examinadas:0, devolvidas:0, falhas:0}` and writes no audit

#### Scenario: Missing cron secret
- **WHEN** the request carries no valid bearer or `x-cron-secret`
- **THEN** the response is HTTP 403 `forbidden`

### Requirement: Stale case reminders
`GET|POST /api/v1/cron/case-stale-watcher` SHALL, for `agent_cases` in `awaiting_human` whose `updated_at` is older than 24 h and `followup_attempts < 3`, insert one open `agent_inbox_items` row of kind `case_stale` (severity `warn`, `ref_kind='agent_case'`) unless one is already open for the case, then increment `followup_attempts`, and SHALL apply the same 3-reminder ceiling to unacknowledged `passagens_de_atendimento` via `cobrancas`.

#### Scenario: Third reminder is the last
- **WHEN** a case untouched for 3 days has `followup_attempts=2`
- **THEN** a `case_stale` notice is opened, `followup_attempts` becomes 3 and later runs no longer select the case

### Requirement: Demand next step and closing
`PATCH /api/v1/demandas/{id}` SHALL require role `agent` and either set `proximo_passo` (3–500 chars) and optional `proximo_passo_em` on an open demand (`fechada_em is null`) of the active organization, auditing `demanda.proximo_passo_definido`, or close it with `{action:"encerrar", expected_revision, desfecho}` through `fn_demanda_encerrar`, answering 409 `conflict` on a stale revision and 404 `not_found` for an absent or already closed demand.

#### Scenario: Next step on a closed demand
- **WHEN** an agent sets a next step on a demand whose `fechada_em` is filled
- **THEN** the response is HTTP 404 `not_found` and the row is unchanged

#### Scenario: Concurrent close
- **WHEN** the `expected_revision` sent no longer matches the demand
- **THEN** the response is HTTP 409 `conflict`

### Requirement: Case chat availability uses the same resolver as the answer
`GET /api/v1/ai/cases/{id}/chat` SHALL compute `ia_configurada` by calling `resolveOrgLlmConfig` with the case agent's provider and credential when the persona is the case agent (no override for the default persona), returning `false` only on `LlmNotConfiguredError` and `null` (logged) on any other failure, without hiding the rest of the case state.

#### Scenario: Organization credential but no agent on the case
- **WHEN** a case without agent belongs to an organization with a validated credential and no environment key
- **THEN** `ia_configurada` is `true`

#### Scenario: Credential cannot be decrypted
- **WHEN** resolving the credential throws an error other than `LlmNotConfiguredError`
- **THEN** `ia_configurada` is `null` and the response still carries the persona and case status
