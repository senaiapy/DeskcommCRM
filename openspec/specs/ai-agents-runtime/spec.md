# ai-agents-runtime Specification

## Purpose
The AI agent runtime: agents and their immutable versions (`ai_agents`, `ai_agent_versions`) managed under `/api/v1/ai/agents/**`, the long-running `agent-worker` that drains `ai_agent.dispatch_requested` into the durable `job_queue` and runs turns, the `before_send` gate chain with its durable trace (`before_send_traces`), and the exactly-once send intent ledger (`send_ledger`). Credentials, model catalog and spend ceilings are covered by `ai-credentials-budget`; routing between agents, skills and organization memory by `ai-routers-skills-memory`; knowledge retrieval by `ai-knowledge-rag`; follow-up graphs by `followup-flows`.

## Requirements

### Requirement: Agent creation always drafts version 1
`POST /api/v1/ai/agents` SHALL require role `admin` (session) or scope `mcp:write` (Bearer token) via `resolveAuthDual`, take `organization_id` from the session or token row, convert a legacy body without `version` into the version-create body, and insert both an `ai_agents` row and a draft `ai_agent_versions` row, answering 422 `validation_failed` on a schema error.

#### Scenario: Legacy body no longer creates a mute agent
- **WHEN** a client posts a body without `version` (old `rag_bot` format)
- **THEN** the handler still writes a draft row in `ai_agent_versions` for the new agent instead of an `ai_agents` row with no version

#### Scenario: Invalid scope leaves no orphan
- **WHEN** the version payload references a pipeline or knowledge source outside the organization
- **THEN** the response is 422 with the scope error code and no `ai_agents` row is inserted

### Requirement: Publishing a version is atomic and pre-validated
`POST /api/v1/ai/agents/{id}/publish` SHALL require role `admin` (token scope `config:write`, token role `admin`), reject ids not present in the MCP tool catalog with 422 `tool_id_invalid`, publish through the RPC `fn_publish_ai_agent_version`, map the codes in `PUBLISH_ERROR_CODES` to 404 (`agent_not_found`, `version_not_found`) or 422 (all others), and emit an `event_log` row `ai_agent.published`.

#### Scenario: Version of another agent
- **WHEN** `version_id` belongs to a different agent or organization
- **THEN** the response is 404 `version_not_found`

#### Scenario: Credential not validated
- **WHEN** the RPC raises `credential_not_validated`
- **THEN** the response is 422 with code `credential_not_validated` and the published pointer does not move

### Requirement: Test run is a dry run recorded in ai_agent_runs
`POST /api/v1/ai/agents/{id}/versions/{vid}/test` SHALL require role `admin`, insert an `ai_agent_runs` row with `is_dry_run = true` and `status = 'running'`, run the turn through `testAgentVersion` without sending to the channel, and finish the row as `completed` or `failed` (error code `preview_failed`).

#### Scenario: Successful test
- **WHEN** an admin tests a draft version with a sample message
- **THEN** the response contains `run_id`, `final_text` and `guardrails`, and the `ai_agent_runs` row ends with `status = 'completed'`, never `ok`

### Requirement: Pause and archive stop an agent without deleting history
`POST /api/v1/ai/agents/{id}/pause` SHALL require role `admin`, set `ai_agents.paused_at`, and answer 409 `state_conflict` for an archived agent; `DELETE /api/v1/ai/agents/{id}` SHALL soft-archive by setting `archived_at`, clearing `published_version_id` and `is_active = false`.

#### Scenario: Pausing an archived agent
- **WHEN** an admin pauses an agent whose `archived_at` is set
- **THEN** the response is 409 `state_conflict` and nothing is written

### Requirement: The agent-worker is the single consumer of dispatch events
The `agent-worker` process (`workers/agent-worker/main.ts`) SHALL refuse to boot when any of `job_queue`, `lead_checkpoints`, `agent_inbox_items`, `send_ledger` is missing, drain `ai_agent.dispatch_requested` from `event_log` into `job_queue` rows of kind `inbound_turn` deduplicated by `(organization_id, source_event_id)`, and dispatch jobs by `kind` to handlers for `inbound_turn`, `followup_turn`, `case_reply_turn`, `operator_turn`, `approved_reply` and `transactional_delivery`.

#### Scenario: Harness schema missing
- **WHEN** the worker starts against a database without `send_ledger`
- **THEN** boot fails with "schema do harness ausente" naming the missing table

#### Scenario: Retired cron route
- **WHEN** the scheduler calls `GET /api/v1/cron/agent-dispatcher` with a valid cron secret
- **THEN** the response is 200 `{ skipped: true, deprecated: true }` and no event is consumed; a missing secret yields 403 `forbidden`

### Requirement: Failed jobs back off and dead jobs escalate
On job failure the queue SHALL return the job to `pending` with exponential backoff capped at 120 seconds, and when `attempts >= max_attempts` SHALL set `status = 'dead'` and insert an `agent_inbox_items` row of kind `job_dead` in the same statement.

#### Scenario: Fifth failure
- **WHEN** a job fails with `attempts` equal to `max_attempts`
- **THEN** its status becomes `dead` and a `job_dead` inbox item names the job kind and attempts

### Requirement: Every outbound candidate passes the before_send chain
Before sending, the runtime SHALL evaluate `BEFORE_SEND_GATES` in order (stop, lgpd, pacing, messaging_window, spinning, promise, semantic_promise, factual_claim, case_promise, internal_vocabulary, clinical_claim, agenda_stall, disclosure), stop at the first veto marking later gates `skipped`, and, when the attempt carries a `jobId`, persist the per-gate trace in `before_send_traces` with `vetoed_gate` and `vetoed_code`.

#### Scenario: Promise gate vetoes
- **WHEN** the promise gate vetoes a candidate during a queued turn
- **THEN** the message is not sent and a `before_send_traces` row records `vetoed_gate` with every later gate as `skipped`

### Requirement: Semantic promise evidence is bounded and server-collected
The commercial-evidence collector for `semantic_promise` SHALL accept only results of the turn's own consult tools, admit knowledge only from source types `faq`, `documento` and `catalogo`, and cap evidence at 20 items, 4,000 characters per item and 16,000 characters in total.

#### Scenario: Conversation history as evidence
- **WHEN** a knowledge hit comes from a `conversas` source
- **THEN** it is not passed to the classifier as evidence of an offer

### Requirement: Sends are exactly-once by intent through send_ledger
`sendWithLedger` SHALL insert one `send_ledger` row per `(job_id, seq)` holding only the SHA-256 `body_hash`, skip the send when the prior row is `accepted` (`already_sent`) or `vetoed` (`blocked`), and rotate to a new idempotency key only when the prior row is `failed`.

#### Scenario: Retry after an accepted send
- **WHEN** a job retries a turn whose message `seq` is already `accepted`
- **THEN** no second message is sent and the outcome is `already_sent` with the original `crm_message_id`

### Requirement: The legacy ai-response-worker no longer replies
`elegivelParaWorkerLegado` SHALL return `false`, so `workers/ai-response-worker.ts` never produces a normal outbound reply; it remains only for recovery of legacy `rag_bot` agents without a published version.

#### Scenario: Message received for a published agent
- **WHEN** a `message.received` event reaches the legacy worker
- **THEN** the worker returns without sending, and the reply comes from the agent-worker turn

### Requirement: Run history is read from llm_calls
`GET /api/v1/ai/runs` SHALL require role `manager`, read `llm_calls` filtered by the session `organization_id`, accept `purpose`, `status`, `provider` and `limit` query filters validated by Zod, and answer 422 `invalid_query` on invalid filters.

#### Scenario: Bad limit
- **WHEN** a manager calls `GET /api/v1/ai/runs?limit=abc`
- **THEN** the response is 422 `invalid_query`, not 500
