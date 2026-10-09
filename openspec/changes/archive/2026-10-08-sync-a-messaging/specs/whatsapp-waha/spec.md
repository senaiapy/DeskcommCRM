## ADDED Requirements

### Requirement: Pausing a connection opens one self-resolving Central notice
`PATCH /api/v1/channel-sessions/[id]/disabled` and `PATCH /api/v1/channel-sessions/disabled` (role `admin`) SHALL call `sincronizarAvisoDePausa` so that pausing opens one `agent_inbox_items` row with `kind = 'canal_pausado'`, `severity = 'warn'`, `ref_kind = 'channel_session'`, `ref_id` = the channel and a body naming the channel, the author and the time, pausing again updates that same open row (unique partial index `agent_inbox_canal_pausado_aberto_unico` from migration 0589), resuming resolves it with reason `reativado`, and archiving or deleting the channel (`DELETE /api/v1/channel-sessions/[id]`, social disconnect) resolves it with reason `canal_arquivado`.

#### Scenario: Pause, pause again, resume
- **WHEN** an admin pauses a channel, pauses it again, then resumes it
- **THEN** exactly one `canal_pausado` item exists for that channel and it ends with `status = 'resolved'` and a body saying the channel was reactivated

#### Scenario: Paused channel deleted
- **WHEN** a paused channel is deleted through `DELETE /api/v1/channel-sessions/[id]`
- **THEN** its open `canal_pausado` item is resolved with reason `canal_arquivado`

### Requirement: Sent WAHA message stores the canonical external id
`sendMessageHandler` SHALL store `messages.external_id` of a message sent through WAHA as `canonicalWahaExternalId()` from `lib/waha/message-id.ts`: the bare tail of the id for individual chats and the intact id for `@g.us` group chats, which is the same string the echo ingest writes, so `messages_org_external_id_unique` rejects the echo's duplicate row.

#### Scenario: WEBJS returns the serialized id
- **WHEN** a CRM send to an individual chat returns `true_5511999990000@c.us_3EB0ABC`
- **THEN** the message row is stored with `external_id = '3EB0ABC'` and the echo of that message does not create a second row

#### Scenario: Group send
- **WHEN** a CRM send to a group returns `true_123-456@g.us_3EB0ABC_5511999990000@c.us`
- **THEN** the message row keeps that id intact

### Requirement: Positive check-exists answers are memoized for ten minutes
The WAHA recipient resolution in `lib/waha/resolve-contact-whatsapp-id.ts` SHALL memoize only positive check-exists results per `(waha_session_name, phone digits)` for `TTL_DO_MEMO_CHECK_EXISTS_MS = 600000` with at most 5000 entries, and SHALL query WAHA again after a negative result or a failure.

#### Scenario: Three bubbles to the same number
- **WHEN** the agent sends three messages within a minute on one session to a Brazilian mobile that exists on WhatsApp
- **THEN** WAHA's check-exists endpoint is queried only for the first message

#### Scenario: Number not found
- **WHEN** check-exists answers that the number does not exist
- **THEN** the next send to that number queries WAHA again

### Requirement: Conversation started from the connected phone creates the deal
For an outbound-from-phone WAHA message that is not the echo of a CRM send, `handleOutboundFromUserPhone` in `lib/waha/ingest.ts` SHALL call `garantirLeadDaConversa` with origin `ORIGEM_DO_WHATSAPP_OPERADOR` (`crm_leads.source = 'whatsapp_operador'`, timeline reason "primeira mensagem enviada pelo celular") only when the contact has no `crm_leads` row at all, skipping blocked and personal contacts, and a failure SHALL be logged without failing the webhook.

#### Scenario: Operator writes first to a new number
- **WHEN** the operator sends from the paired phone to a number with no lead in the organization
- **THEN** a lead is created in the entry stage of the default pipeline with `source = 'whatsapp_operador'` and a `lead.created` event is emitted

#### Scenario: Customer already had a closed deal
- **WHEN** the operator sends a tracking code from the phone to a contact whose only lead is `won`
- **THEN** no new lead is created and the log records `motivo = "contato_ja_tem_lead"`
