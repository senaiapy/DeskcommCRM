## ADDED Requirements

### Requirement: Agent-created deal in "own" mode belongs to the agent
`POST /api/v1/leads` SHALL, when the caller's role is `agent`, the body names neither `owner_user_id` nor `owner_agent_id`, and the active organization's `settings.visibility_mode` (read with the admin client from the trusted active organization) is `own`, set `owner_user_id` to the caller before creating the lead.

#### Scenario: Agent creates a deal without owner in "own" mode
- **WHEN** an `agent` of an organization with `visibility_mode = 'own'` posts a valid lead without owner fields
- **THEN** the response is 201 and the lead has `owner_user_id` equal to the caller

#### Scenario: Default mode
- **WHEN** the same request comes from an organization with `visibility_mode = 'own_and_unassigned'`
- **THEN** the lead is created without owner

### Requirement: Visibility refusal on a lead write is a permission error
Lead write handlers in `app/api/v1/leads/_handler.ts` SHALL map a Postgres `42501` whose message names `"crm_leads"` (the RLS check of `fn_can_view_lead`) to HTTP 403 `forbidden` with a translated message, while any other `42501` stays HTTP 500.

#### Scenario: Agent assigns a colleague in "own" mode
- **WHEN** an `agent` in `visibility_mode = 'own'` creates a lead with `owner_user_id` of another member
- **THEN** the response is 403 `forbidden` instead of 500 and no row is inserted

### Requirement: The MCP move tool forwards the loss reason
The MCP tool `crm_move_lead_stage` SHALL accept an optional `lost_reason` (string, at most 500 characters) and SHALL pass it to the move handler, so that an agent moving a deal into a lost stage is checked against the funnel's loss vocabulary instead of having the reason dropped.

#### Scenario: An agent marks a deal as lost through MCP
- **WHEN** an agent calls `crm_move_lead_stage` with a lost-stage target and `lost_reason: "price"`
- **THEN** the deal moves with `status = 'lost'` and `lost_reason = 'price'`, the same outcome as `POST /api/v1/leads/[id]/move`

