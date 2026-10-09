# notifications Specification

## Purpose
User notifications: browser Web Push (VAPID) subscriptions under `/api/v1/notifications/push`, the
`web-push-inbound.v1` event handler that turns `event_log` events into pushes, the service worker
`public/notify-sw.js`, and the in-page toast/notification delivery gated by per-browser preferences. The event
drain that runs the handler is covered by `event-bus-workers`; the live in-app updates are covered by `realtime`;
transactional e-mail is covered by `platform-email`.

## Requirements

### Requirement: Push availability depends on VAPID keys
`GET /api/v1/notifications/push` SHALL require `requireRole("viewer")` and return `{enabled, public_key}`, where `enabled` is true only when both `VAPID_PUBLIC_KEY` and `VAPID_PRIVATE_KEY` are non-empty and `public_key` is null when the public key is empty; the VAPID subject SHALL be `mailto:` the installation `SUPPORT_EMAIL` when it contains `@`, else `NEXT_PUBLIC_APP_URL`.

#### Scenario: Keys not configured
- **WHEN** `VAPID_PRIVATE_KEY` is empty
- **THEN** the response is 200 `{enabled: false}`

### Requirement: Subscribe a browser endpoint
`PUT /api/v1/notifications/push` SHALL call `requireSupportWrite` and `requireRole("viewer")`, answer 503 `unavailable` when VAPID is not configured, validate `{endpoint (URL), keys: {p256dh, auth}}` (422 `validation_failed`), upsert `push_subscriptions` on the unique `endpoint` with `organization_id` and `user_id` from the session, and audit `notification_prefs.changed` with `metadata.channel = 'web_push'`.

#### Scenario: Re-subscribing the same browser
- **WHEN** the same endpoint is sent twice
- **THEN** a single `push_subscriptions` row exists with the latest keys

#### Scenario: Invalid body
- **WHEN** `keys.auth` is missing
- **THEN** the response is 422 `validation_failed`

### Requirement: Unsubscribe only one's own endpoint
`DELETE /api/v1/notifications/push` SHALL call `requireSupportWrite` and `requireRole("viewer")`, take `{endpoint}` and delete only the `push_subscriptions` row with that endpoint and the caller's `user_id`; RLS policy `push_subscriptions_own` SHALL restrict every access to rows of the caller's organizations and `user_id = auth.uid()`.

#### Scenario: Another user's endpoint
- **WHEN** a user sends the endpoint registered by a colleague
- **THEN** no row is deleted and the response is 200

### Requirement: Push handler consumes domain events
The handler `web-push-inbound.v1` SHALL consume `message.received`, `message.group_received`, `lead.assigned`, `lead.won`, `lead.lost`, `user.mentioned` and `central.aviso_criado`, declare `naOrgParada: "pula"`, and record `skipped` with `vapid_ausente` when VAPID is not configured.

#### Scenario: VAPID absent
- **WHEN** a `message.received` event is drained without VAPID keys
- **THEN** the handler result is `skipped` with detail `vapid_ausente` and nothing is sent

### Requirement: Lead and mention pushes go to one user
For `lead.assigned` (payload `to_user_id`, else the lead owner), `lead.won` and `lead.lost` (lead owner) and `user.mentioned` (payload `to_user_id`) the handler SHALL send only to that user's subscriptions in the organization, with tags `lead-assigned:{id}`, `lead-won:{id}`, `lead-lost:{id}` or `mention:{conversation_id}`, and SHALL skip with `sem_lead` when the event names no lead.

#### Scenario: Lead won
- **WHEN** `lead.won` is drained for a lead owned by user U
- **THEN** only U's subscriptions receive "Lead ganho"

### Requirement: Expired subscriptions are pruned
`enviarPushDaOrg` SHALL truncate the body to 140 characters and delete a `push_subscriptions` row when the push service answers 404 or 410, counting it in `gone`; other send errors SHALL be logged and not delete the row.

#### Scenario: Browser unsubscribed
- **WHEN** the push service answers 410 for an endpoint
- **THEN** that `push_subscriptions` row is deleted

### Requirement: Service worker shows the tray only when no tab is visible
`public/notify-sw.js` SHALL, on `push`, show a notification only when no window client is visible, using the payload's `title`, `body`, `tag` and `href` (default `/app/inbox`), and open `data.href` on `notificationclick`.

#### Scenario: Tab in foreground
- **WHEN** a push arrives while a CRM tab is visible
- **THEN** the service worker shows no tray notification

### Requirement: In-page delivery honors per-browser preferences
`entregarAviso` SHALL show the in-app toast only when `canalLigado(category, "in_app")` and raise a browser notification only when `canalLigado(category, "push")`, where preferences are stored in `localStorage` key `notify.prefs.v1`; `shouldNotifyInbound` SHALL ignore outbound messages, synthetic `reassinado` deliveries and the conversation already open in a focused tab.

#### Scenario: Conversation already open
- **WHEN** an inbound message arrives for the conversation open in the focused tab
- **THEN** no in-page notification is raised

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
