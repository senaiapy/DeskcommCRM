## Purpose
Prospecting is outbound cold outreach driven by an AI agent: an organization stores a data-provider credential, runs a paid company search that fills `prospecting_candidates`, selects candidates, starts a campaign (`prospecting_campaigns`) and the `prospecting` cron sends the first message to one candidate at a time under daily and interval limits, rechecking consent and conversation state right before sending. The screen is `/app/prospecting`. Broadcast campaigns to existing contacts are covered by `campaigns-broadcast`; opt-out and consent by `opt-out-consent`; the agent that answers the replies by `ai-agents-runtime`; dual (cookie or `dsk_` token) auth by `api-rest-contract`.

## ADDED Requirements

### Requirement: Prospecting overview read
`GET /api/v1/prospecting` SHALL accept a session with role `admin` or a `dsk_` token with scope `mcp:read`, and return for the caller's organization `configured`, the last 50 `prospecting_campaigns`, up to 5000 `prospecting_candidates` with a derived `progress` (`qualified`, `replied` or the candidate status), published automatic agents, non-archived channels whose provider allows free-form messages outside the service window, and open pipeline stages, with `Cache-Control: no-store`.

#### Scenario: Channel without free-form outbound
- **WHEN** an organization has a channel whose provider capabilities lack `freeformOutsideWindow`
- **THEN** that channel is absent from `channels` in the response

### Requirement: Prospecting actions and their audit
`POST /api/v1/prospecting` SHALL require role `admin` (session) or scope `mcp:write` (token), validate the body against the `action` union `configure|search|start|pause|resume|adjust_pace|select|select_in_queue|discard_unselected` (422 `validation_failed` otherwise), and after a successful action insert an `api_audit_log` row with action `prospecting.changed` and `metadata.operation` equal to the action.

#### Scenario: Pace change is audited with before and after
- **WHEN** an admin sends `{action:"adjust_pace", id, daily_limit:20, interval_minutes:30}`
- **THEN** the audit metadata contains `previous` and `next` pace values

### Requirement: Tokens cannot start sending
A request authenticated by a `dsk_` token SHALL be limited to the actions `configure`, `search` and `pause`, and any other action SHALL be refused with HTTP 403 `forbidden` so that starting or resuming outreach always requires a person on the screen.

#### Scenario: Token tries to start a campaign
- **WHEN** a token with scope `mcp:write` posts `{action:"start", ...}`
- **THEN** the response is HTTP 403 `forbidden` and the campaign status is unchanged

### Requirement: Encrypted provider credential
The `configure` action SHALL accept an `api_key` of 10–500 characters and store it only encrypted in `prospecting_settings.credential_encrypted` (one row per organization, upserted), and the search SHALL decrypt it server-side for the provider call.

#### Scenario: Reconfigure
- **WHEN** an admin configures a new key for an organization that already has one
- **THEN** the same `prospecting_settings` row holds the new ciphertext and the response is `{configured:true}`

### Requirement: Idempotent search per request id
The `search` action SHALL require a UUID `request_id`, return the existing campaign when `prospecting_campaigns` already has that `(organization_id, request_id)`, otherwise insert the campaign and start the provider run, recording `search_status='running'` on success or `search_status='unknown'` with an error message when the start cannot be confirmed.

#### Scenario: Double click on search
- **WHEN** the same `request_id` is posted twice
- **THEN** only one campaign exists and both responses return it

### Requirement: One running campaign per organization and a per-organization lock
Mutating prospecting work SHALL run under the Postgres advisory lock `prospecting:<organization_id>` (HTTP 409 when it is held), `start` SHALL only accept a `draft` campaign whose search `succeeded`, and `start` or `resume` SHALL fail with HTTP 409 while another campaign of the organization is `running`.

#### Scenario: Second campaign
- **WHEN** an admin starts a campaign while another one of the organization is `running`
- **THEN** the response is HTTP 409 with code `prospecting_unavailable` and the message to pause the current campaign first

### Requirement: Pause takes effect without the lock
The `pause` action SHALL set `status='paused'` on the organization's `running` campaign without waiting for the lock and answer 404 when no running campaign matches, and the delivery guard SHALL refuse a send whose campaign is no longer `running`.

#### Scenario: Pause during a send
- **WHEN** a campaign is paused after the message was generated but before it is delivered
- **THEN** the delivery guard throws HTTP 409 and the message is not sent

### Requirement: Prospecting cron
`GET|POST /api/v1/cron/prospecting` SHALL require `INTERNAL_CRON_SECRET` or `INTERNAL_SECRET` (HTTP 403 `forbidden` otherwise), process up to 20 operating organizations (`fn_org_operante`) with a running campaign or search for at most 180 seconds, mark searches stuck in `starting` for over 2 minutes as `unknown` and candidates stuck in `sending` for over 10 minutes as `failed` without retrying them, synchronize running searches and send at most the next candidate of the running campaign.

#### Scenario: Interrupted send is not repeated
- **WHEN** a candidate has `status='sending'` with `attempted_at` 15 minutes ago
- **THEN** the next cron run sets it to `failed` and never sends it again automatically

#### Scenario: Suspended organization
- **WHEN** an organization with a running campaign is not operating
- **THEN** the cron does not select it and nothing is sent

### Requirement: Send pacing limits
The sender SHALL count attempts of the last 24 hours (including failures) and defer the campaign by updating `next_send_at` when the campaign reached its `daily_limit` (1–50, default 10), the organization reached 50 attempts, the cold-outreach warm-up ceiling for the number's age is reached, or `interval_minutes` (5–1440, default 15) has not elapsed since the last attempt.

#### Scenario: Daily limit reached
- **WHEN** a campaign with `daily_limit=10` already has 10 attempts in the last 24 hours
- **THEN** no message is sent and `next_send_at` moves to 24 hours after the oldest attempt of the organization in that window

### Requirement: Delivery guard on consent and conversation state
Immediately before sending, the guard SHALL refuse with HTTP 409 when the candidate is no longer `sending` on the same conversation, the contact is blocked, `force_human`, anonymized, has `consent.marketing.declined_at` or lacks a valid legal basis, or the conversation already has inbound or outbound messages, an assigned user, a closed/resolved/archived/claimed status or an active `bot_silenced_until`.

#### Scenario: Contact already talking to the company
- **WHEN** the candidate's conversation has `last_inbound_at` set
- **THEN** the send is refused with HTTP 409 and no message is delivered

### Requirement: Prospecting agent setup routes
`POST /api/v1/prospecting/agents` and `POST /api/v1/prospecting/agents/prepare` SHALL require role `admin`, validate the body with `prospectingAgentSetupSchema` (422 `validation_failed`), and `GET`/`PATCH /api/v1/prospecting/agents/session` and `POST /api/v1/prospecting/agents/chat` SHALL require role `admin` with every read and write scoped to the active organization.

#### Scenario: Manager opens the setup chat
- **WHEN** a `manager` posts to `/api/v1/prospecting/agents/chat`
- **THEN** the response is HTTP 403 `forbidden_role`
