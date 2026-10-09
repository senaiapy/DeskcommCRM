## ADDED Requirements

### Requirement: Inbound message push goes to the responsible people who can see the conversation
For `message.received` the handler SHALL skip with `canal_desativado` when the event's channel session is disabled and with `contato_pessoal` when the contact is personal, otherwise build a payload titled with the contact label (or "Nova mensagem"), body "Mídia" for non-text messages, tag `msg:{conversation_id}`, href `/app/inbox?id={conversation_id}` and, for non-anonymized contacts with an avatar, a 300-second signed avatar URL, and SHALL send it only to the subscriptions returned by `fn_push_inscricoes_que_veem_a_conversa` (migration `20261008133818_0612_push_so_a_quem_ve_a_conversa.sql`, which applies `fn_can_view_conversation` per subscriber), narrowed by `destinatariosDaMensagem`: the conversation's assignee plus the organization admins; else the owners of the contact's open `crm_leads` plus the admins; else everyone who can see it (also for group conversations or when reading the assignment fails).

#### Scenario: Personal contact
- **WHEN** a message arrives from a contact with `is_personal = true`
- **THEN** the result is `skipped` with `contato_pessoal`

#### Scenario: Image message on an assigned conversation
- **WHEN** an inbound message of type `image` arrives from a named contact on a conversation assigned to attendant A
- **THEN** only A's and the admins' subscriptions receive a push whose title is the contact name and body is "Mídia"

#### Scenario: Nobody responsible
- **WHEN** the conversation has no assignee and the contact has no open lead with an owner
- **THEN** every subscriber allowed by `fn_can_view_conversation` receives the push

#### Scenario: Visibility read fails
- **WHEN** `fn_push_inscricoes_que_veem_a_conversa` returns an error
- **THEN** no push is sent and the failure is logged

### Requirement: In-page inbound alert asks the server whether to notify
`GET /api/v1/conversations/{id}/aviso-de-mensagem` SHALL require role `viewer`, answer 404 `not_found` for a non-UUID id or a conversation the session cannot read in the active organization, return `{avisar}` computed by `carregarDestinatariosDaMensagem` and `usuarioRecebeAviso` for the calling user (`true` when the rule has no answer), and answer 500 `internal_error` when the assignment read fails; `useInboundMessageAlerts` SHALL consult it before raising the toast or browser notification.

#### Scenario: Conversation assigned to someone else
- **WHEN** an agent who is not admin calls the route for a conversation assigned to another attendant
- **THEN** the response is `{avisar: false}`

## REMOVED Requirements

### Requirement: Inbound message push goes to the organization
**Reason**: Since migration 0612 and `lib/notifications/destinatarios-da-mensagem.ts` the push no longer goes to every subscription of the organization; it goes to the responsible people who can see the conversation.
**Migration**: Replaced by "Inbound message push goes to the responsible people who can see the conversation", which keeps the skip reasons and payload of the removed requirement.
