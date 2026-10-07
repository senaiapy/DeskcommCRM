## Purpose
Opt-out ("quem pediu para sair, sai") and per-purpose consent for contacts. A single detection rule in `lib/opt-out/deteccao.ts` decides whether an inbound text is an opt-out request in Portuguese or Spanish: `ehPedidoDeOptOut` (unambiguous: the whole message is an opt-out keyword, or a cessation verb phrase whose object is communication) and `ehOptOutProvavel` (unambiguous plus ambiguous phrases). Channel ingests (WAHA, Meta Cloud, Zernio/social) call `aplicarEfeitosPosEntrada` in `lib/channels/pos-entrada.ts`, which writes `contacts.is_blocked` before any lead is created; the agent runtime (`lib/agent-engine/agent/human-handoff.ts`, `inbound-turn.ts`) uses the same module to escalate, and the `before_send` stop gate (`lib/agent-engine/guardrails/before-send.ts`) vetoes sends to blocked contacts. Admins reverse a block through `POST /api/v1/contacts/[id]/unblock`. Consent is the `contacts.consent` jsonb map (purposes `marketing`, `transactional`, `profiling`), edited through `PATCH /api/v1/contacts/[id]`. Campaign suppression and eligibility (`lib/campanhas/elegibilidade.ts`) read the same `is_blocked` flag and are specified in `campaigns-broadcast`.

## ADDED Requirements

### Requirement: Unambiguous opt-out detection
`ehPedidoDeOptOut` SHALL return true only when the accent-stripped, lower-cased message, reduced to letters, equals one of `PALAVRAS_DE_OPT_OUT` (`stop`, `parar`, `pare`, `sair`, `cancelar`, `descadastrar`, `remover`, `unsubscribe`, `baja`, `bajar`, `salir`, `desuscribir`, `desuscribirme`) or when it matches a phrase pattern that pairs a cessation verb with a communication object (e.g. "pare de me mandar", "não quero mais receber", "no quiero recibir"), and SHALL return false for those words used inside an unrelated sentence.

#### Scenario: Isolated keyword
- **WHEN** the inbound text is `"PARAR!"`
- **THEN** `ehPedidoDeOptOut` returns true

#### Scenario: Keyword inside a business question
- **WHEN** the inbound text is `"tem como parar a dor?"`
- **THEN** `ehPedidoDeOptOut` returns false

#### Scenario: Spanish request with communication object
- **WHEN** the inbound text is `"no quiero recibir más mensajes"`
- **THEN** `ehPedidoDeOptOut` returns true

#### Scenario: Cessation verb with a non-communication object
- **WHEN** the inbound text is `"pare de mandar o pedido nesse endereço"`
- **THEN** `ehPedidoDeOptOut` returns false

### Requirement: Ingest writes the block
For every 1:1 inbound message on WAHA, Meta Cloud and Zernio/social channels, `aplicarEfeitosPosEntrada` SHALL, as its first effect and only when `ehPedidoDeOptOut` is true, update the contact (filtered by `organization_id` and `id`) to `is_blocked = true`, `blocked_reason = 'stop_keyword'`, `blocked_at = now()` and write an audit row `contact.blocked` with `metadata.reason = 'stop_keyword'`.

#### Scenario: Customer replies "stop"
- **WHEN** a contact sends `"stop"` on a WhatsApp conversation
- **THEN** the contact row has `is_blocked = true` and `blocked_reason = 'stop_keyword'`
- **AND** an audit row with action `contact.blocked` exists for that contact

#### Scenario: Group message
- **WHEN** a participant writes `"stop"` in an enabled WhatsApp group
- **THEN** no contact is blocked, because group ingest skips post-ingest effects

### Requirement: Open deals close on opt-out
After the block is written, the ingest SHALL close every `crm_leads` row of that contact with `status = 'open'` as lost with reason `opted_out_of_messages`, and a failure there SHALL be logged without failing message ingestion.

#### Scenario: Contact with an open deal opts out
- **WHEN** a contact with one open lead sends an unambiguous opt-out
- **THEN** that lead is closed as lost with reason `opted_out_of_messages`

### Requirement: Agent runtime uses the same rule
The agent runtime SHALL detect opt-out through `detectAmbiguousOptOut`, which delegates to `ehOptOutProvavel` from `lib/opt-out/deteccao.ts`, and on a match SHALL hand the conversation off to a human with reason `suspected_optout` instead of replying, without itself setting `is_blocked`.

#### Scenario: Ambiguous stop signal
- **WHEN** a pending inbound text matches only an ambiguous opt-out phrase
- **THEN** the turn ends in a human handoff with reason `suspected_optout` and the contact's `is_blocked` is unchanged

### Requirement: Blocked contacts never receive agent messages
The `before_send` stop gate SHALL veto every agent send with code `contato_bloqueado` when the contact, read directly from `contacts` by `organization_id` and `id`, has `is_blocked` (or `is_personal`/`force_human`) set.

#### Scenario: Agent tries to message a blocked contact
- **WHEN** the agent prepares a reply to a contact with `is_blocked = true`
- **THEN** the send is vetoed with code `contato_bloqueado` and nothing is sent

### Requirement: Admin-only unblock
`POST /api/v1/contacts/[id]/unblock` SHALL require `requireSupportWrite` and role `admin`, return 422 `validation_failed` for a non-UUID id and 404 `not_found` when the contact is not in the caller's organization, and otherwise set `is_blocked = false`, `blocked_reason = null`, `blocked_at = null` and write an audit row `contact.unblocked` while keeping the earlier `contact.blocked` row.

#### Scenario: Agent tries to unblock
- **WHEN** a user with role `agent` calls the unblock route
- **THEN** the request is refused by `requireRole("admin")` and the contact stays blocked

#### Scenario: Admin unblocks
- **WHEN** an admin unblocks a blocked contact of the organization
- **THEN** the response is 200 with `is_blocked: false` and an audit row `contact.unblocked` is written

### Requirement: Consent is a per-purpose map merged on update
`contacts.consent` SHALL be a NOT NULL jsonb defaulting to purposes `marketing`, `transactional` and `profiling` (each with `granted_at`, `source`, `version`), and `PATCH /api/v1/contacts/[id]` (role `agent`) SHALL shallow-merge the submitted `consent` object into the stored map and record the touched purposes as `consent_scopes` in the `contact.updated` audit metadata.

#### Scenario: Recording transactional consent keeps marketing consent
- **WHEN** a contact with a granted `marketing` consent is patched with `{ "consent": { "transactional": { ... } } }`
- **THEN** the stored `consent` still contains the previous `marketing` entry and the new `transactional` entry
- **AND** the `contact.updated` audit metadata includes `consent_scopes: ["transactional"]`
