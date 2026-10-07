## Why

DeskcommCRM has no OpenSpec specs yet. This change records, as a retro-spec, what the AI layer of the product already does, so that the four folders of CRM_ERP_V1 (DeskcommCRM, TEMPLATE_CRM, TEMPLATE_CRM_V1, TEMPLATE_ERP_v1) can be compared capability by capability using the shared ids of `openspec-capability-map.md`, before any refactor is decided.

## What Changes

- Documentation only: ten new capability specs describing the existing AI agents runtime, credentials and budget, knowledge base / RAG, routers / skills / org memory, follow-up flows, cases and escalation, the improvement flywheel, the MCP server, prospecting and media transcription.
- No product code, schema, migration, route or environment variable is added or changed.
- **BREAKING:** none. Nothing in the running system changes; a requirement that the code does not meet was removed rather than written.

## Capabilities

### New Capabilities
- `ai-agents-runtime`: agent definitions, versions, publish and test runs, the agent-worker turn, before-send guardrails (`before_send_traces`), `send_ledger`, pause and runs listing.
- `ai-credentials-budget`: BYO provider keys encrypted at rest, providers, the model catalog sync cron, cost/pricing, budget enforcement and usage (`llm_calls`).
- `ai-knowledge-rag`: knowledge sources (text, URL, upload), the `rag-indexer` worker, chunking, embeddings, semantic search and the conversations batch cron.
- `ai-routers-skills-memory`: AI routers and members, router decision log, Jev classifier, routing results, skills (install/import/restore) and versioned org memory.
- `followup-flows`: visual follow-up graphs, publish/disable/rollback, enrollments, the flow worker cron, the no-agent follow-up cron and return promises.
- `agent-cases-escalation`: agent inbox items, human cases (reply, chat, alert), handoff orchestrator, sentiment-driven handoff, automatic hand-back and stale-case watcher, demandas.
- `ai-flywheel`: live judge and distiller, improvement proposals and apply, golden candidates, style adjustments, evolution view.
- `mcp-server`: `/api/mcp` Streamable HTTP server with `dsk_` token auth, rate limit, audit, tool catalog, external MCP server client.
- `prospecting`: prospecting lists and sends, delivery guard, prospecting cron and prospecting agents.
- `media-transcription`: media persist and derive workers (transcription, image description) and their outputs.

### Modified Capabilities
- none

## Impact

- Files: only `openspec/changes/retro-ai/**`.
- Overlaps cited in the Purpose sections: `event-bus-workers`, `api-tokens`, `api-rest-contract`, `audit-log`, `rate-limit`, `file-storage`, `data-retention`, `lgpd-privacy`, `routing-attendants`, `inbox-conversations`, `whatsapp-waha`.
