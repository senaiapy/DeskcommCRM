## Purpose
Live screen updates through Supabase Realtime: the `supabase_realtime` publication that decides which tables
stream `postgres_changes`, the short-lived token endpoint that authenticates the browser socket, and the shared
`useRealtimeChannel` hook (with `lib/realtime/channels.ts`) that subscribes, reconnects and signals missed
events. Push and toast delivery are covered by `notifications`; the session itself is covered by
`auth-session`; row visibility is covered by `multi-tenancy-rls`.

## ADDED Requirements

### Requirement: Published tables
The baseline SHALL ensure publication `supabase_realtime` exists and contains `messages`, `conversations`, `crm_leads`, `ai_agents`, `ai_agent_runs`, `ai_knowledge_sources`, `crm_lead_activities`, `crm_lead_risk_states`, `crm_lead_reactivations`, `calendar_appointments`, `voice_calls`, `user_organizations` and `conversation_notes`, adding each idempotently, and SHALL keep `crm_lead_scores` out of it.

#### Scenario: Update re-applies the baseline
- **WHEN** `update.sh` re-applies the baseline on a database where `messages` is already published
- **THEN** no error is raised and the table stays published once

#### Scenario: Score recalculation
- **WHEN** a lead score is recalculated
- **THEN** no realtime change is streamed for `crm_lead_scores`

### Requirement: Token endpoint for the realtime socket
`GET /api/v1/auth/realtime-token` SHALL validate the user with `getUser()` and return `{access_token, expires_at}` with `cache-control: no-store, max-age=0`, or 401 `unauthenticated` when there is no valid user or session token.

#### Scenario: Anonymous call
- **WHEN** the endpoint is called without a session cookie
- **THEN** the response is 401 `unauthenticated` with `cache-control: no-store`

### Requirement: Socket authenticated before each subscription
`prepareRealtimeAuthentication` SHALL fetch the token from `/api/v1/auth/realtime-token` (cached until near expiry and coalesced across concurrent channels) and call `realtime.setAuth(token)` before a channel subscribes, so `postgres_changes` are filtered by RLS for that user; when no token is available the channel status SHALL be `channel_error` and a reconnect SHALL be scheduled.

#### Scenario: Token fetch fails
- **WHEN** the token endpoint answers 401
- **THEN** `useRealtimeChannel` reports `channel_error` and retries later

### Requirement: Per-instance channel names
`useRealtimeChannel({name, postgresChanges?, broadcast?, onChange, enabled})` SHALL subscribe to a channel named `{name}::{instanceId}#{attempt}`, default `schema` to `public`, pass `filter` only when given, and expose `status` as `connecting`, `subscribed`, `channel_error`, `timed_out` or `closed` (`closed` when `enabled` is false).

#### Scenario: Two components on the same board
- **WHEN** two mounted components use `name: "kanban-P1"`
- **THEN** each gets its own channel and unmounting one does not close the other

### Requirement: Reconnect with capped backoff
On `CHANNEL_ERROR`, `TIMED_OUT` or `CLOSED` the hook SHALL remove the old channel and resubscribe after `min(30000, 1000 * 2^attempt)` milliseconds, and SHALL cancel pending retries on unmount.

#### Scenario: Socket drops repeatedly
- **WHEN** the channel errors six times in a row
- **THEN** the waits are 1s, 2s, 4s, 8s, 16s and 30s

### Requirement: Synthetic delivery after resubscribing
When a channel reaches `SUBSCRIBED` after at least one failed attempt, the hook SHALL reset the attempt counter and call `onChange({tipo: "reassinado"})` once, so consumers refetch what was missed while disconnected.

#### Scenario: Message sent while offline
- **WHEN** the socket reconnects after a message was inserted during the outage
- **THEN** the inbox listener receives `{tipo: "reassinado"}` and refetches the conversation list

### Requirement: Consumers scope subscriptions by tenant or entity
The inbox SHALL subscribe to `conversations` with `organization_id=eq.{orgId}`, inbound alerts to `messages` INSERT with `organization_id=eq.{orgId}`, the kanban board to `crm_leads` with `pipeline_id=eq.{pipelineId}`, the lead timeline to `crm_lead_activities` INSERT with `lead_id=eq.{leadId}` and agent runs to `ai_agent_runs` with `agent_id=eq.{agentId}`, each disabled (`*-disabled`) when the id is absent.

#### Scenario: No active organization
- **WHEN** the inbox mounts before the organization id is known
- **THEN** the channel is `inbox-disabled` and no subscription is opened
