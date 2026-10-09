## ADDED Requirements

### Requirement: Starting a conversation lets the user choose the number
"Iniciar conversa no Inbox" from the contacts list SHALL offer the channels returned by `candidatosParaConversaNova()` (message-capable provider, `phone_number` set, not archived, not `metadata.disabled`, not an e2e seed session), allow choosing only those also returned by `elegiveisParaConversaNova()` (`lerEstadoDoCanal(status).utilizavel`), ask only when more than one is eligible, and send the choice as `channel_session_id` to `POST /api/v1/conversations/open-with-contact`.

#### Scenario: Two connected numbers
- **WHEN** an organization has two working WhatsApp numbers and the user starts a conversation from a contact
- **THEN** a selector asks which number to use and the conversation opens with the chosen `channel_session_id`

#### Scenario: One number down
- **WHEN** one of two numbers is in `FAILED`
- **THEN** it is listed with its state but cannot be chosen, and with a single eligible number the conversation opens without asking

### Requirement: Inbox shows the channel a conversation came through
The conversation list item and the conversation header SHALL label the conversation's channel with `rotuloDoCanalDaConversa()`, which uses `channel_sessions.display_name`, falls back to `phone_number`, and returns null when both are empty.

#### Scenario: Named connection
- **WHEN** a conversation's channel has `display_name = 'Loja Centro'`
- **THEN** the list and the open conversation header show "Loja Centro" instead of the phone number
