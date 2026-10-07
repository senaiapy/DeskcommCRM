## Context

Retro-spec of the AI layer of DeskcommCRM (Next.js 16 + Supabase). The specs describe the code as it is on `main` (commit 4eb556a) on 2026-10-07; they are a baseline for cross-folder comparison, not a design for new work.

## How evidence was gathered

- Seed material: `CLAUDE.md`, `docs/specs/10-spec-ai-agents-runtime.md`, `11-spec-mcp-server-internal.md`, `12-spec-ai-agents-ui.md`, `13-spec-governanca-atendimento.md`, `15-spec-casos-humanos.md`, `jev-roteador-sob-demanda.md`, `promessas-com-evidencias-consultadas.md`, `docs/prd/05-prd-ai-rag-handoff.md`.
- Authority: the code. Every requirement was checked by opening the route, worker, lib module or `supabase/baseline.sql` definition it names; requirements that could not be backed were dropped or narrowed (e.g. `before_send_traces` is persisted only when the attempt has a `jobId`). Each task in `tasks.md` cites the lines read.
- Two reviewers split the ten capabilities; `openspec validate retro-ai --strict` was run after each.

## Decisions

- `lib/routing` + cron `routing-worker` is human attendant routing, not AI: it is left to `routing-attendants` and excluded from `ai-routers-skills-memory`.
- "Promises" has two meanings in the code: the `semantic_promise` before-send guardrail (spec'd in `ai-agents-runtime`) and return promises stored as `cron_jobs` rows with `job_kind='followup_turn'` (spec'd in `followup-flows`).
- The judge and distiller run inside the agent worker loop, not as a cron; golden candidates have no route or screen, so they are spec'd as runtime behaviour inside `ai-flywheel`.

## Inconsistencies found (docs vs code, code vs code)

- Cron auth failure codes disagree: `app/api/v1/cron/sync-model-catalog/route.ts` answers 401 `unauthorized`; `agent-dispatcher`, `kb-conversations-batch`, `followup-flow-worker`, `followup-sem-agente` answer 403 `forbidden`.
- `app/api/v1/cron/agent-dispatcher` is a permanent no-op (`{skipped:true, deprecated:true}`); the real consumer is the agent-worker drain (`lib/agent-engine/edge/crm/drain.ts`).
- `workers/ai-response-worker.ts` header still describes generating replies, but `elegivelParaWorkerLegado` always returns false (`lib/ai/agents/no-ar.ts:34`).
- `POST /api/v1/ai/agents` legacy-body default model is `anthropic/claude-sonnet-5` while the column default is `claude-sonnet-4-6`.
- The enroll 409 message says "1 por lead na organização" (`lib/followup/enroll.ts:150`) but the unique index `idx_followup_enrollments_one_live` is per `(pointer_id, contact_id)`.
- `CLAUDE.md` / Spec 11 §7 call the MCP rate limit an "Upstash sliding window"; `lib/mcp/rate-limit.ts` is a fixed window (`checkRateLimit`).
- `CLAUDE.md` describes the AI stack as Vercel AI Gateway with Anthropic primary; the code also routes through OpenRouter, custom OpenAI-compatible endpoints and `ai_purpose_bindings`.
- `agent_inbox_items.kind` CHECK is first defined narrowly in `supabase/baseline.sql:~6330` and widened by a later appendix (adds `case_stale`, `midia_nao_lida`).

## Declared debts

- `publicarMemoriaDaOrg` (`lib/ai/memoria-da-org.ts:34`) computes the next version with select-max + insert (not atomic).
- `ai_invocations` is no longer written; `llm_calls` is the only telemetry table.
- `jev_router_decisions` is read through an `as 'jev_observacoes'` cast because generated types are stale.
- Error codes used but not registered in `lib/api/errors.ts`: `read_failed`, `save_failed`, `invalid_body` (style-adjustments), `prospecting_unavailable`, `reply_context_unavailable`, `proposal_not_found`, `proposal_already_applied`, `proposal_type_unsupported`, `agent_not_published`, `publish_failed`.
- `style-adjustments` route builds no `requestId`; `PATCH /api/v1/demandas/[id]` answers bad JSON with 422 `validation_failed` and leaks raw DB `error.message` on 500.
- The handoff orchestrator writes `api_audit_log` directly (`lib/ai/handoff/orchestrator.ts:384`) instead of `audit()`.
- Flywheel proposal types `golden_case` and `reentry_trigger` are allowed by the schema but never produced and rejected on apply; nothing reads `golden_candidates` except retention.
- `workers/media-derive-worker.ts` reads `SUPABASE_DB_URL` and provider keys from raw `process.env`, bypassing `lib/env.ts`.
- `lib/prospecting/provider.ts:46` hardcodes `api.apify.com` despite a per-organization provider choice in `provedor.ts`.

## Risks / Trade-offs

- A retro-spec freezes current behaviour, including the inconsistencies above; fixing them is a separate change with its own MODIFIED deltas.
