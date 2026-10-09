## Why

The AI specs were written against upstream commit `477b77678`. HEAD is 491 upstream commits later and the AI code drifted: the usage screen now sums in Postgres (migration 0586), the reseller plan got its own AI ceiling on the installation key, the sentiment classifier and the media derivation now obey the spend ceiling, the ChatGPT subscription login was rebuilt (single-use state, per-organization model list, publish check in migration 0592), short classifications fall back to a cheaper curated model, new Gemini 3.x rows were priced (0599, 0600), the agent turn formats text for WhatsApp and re-reads the agenda before the checkpoint, the flywheel stopped paying the judge twice, follow-up `move_lead` learned the loss reason, the confirmation sweep no longer stops follow-ups, channel pauses reach the Central, and the external-database tools follow the module switch. This change brings the catalog back in line with the code.

## What Changes

- Spec-only: no product code, schema, route or environment variable is changed by this change.
- `ai-credentials-budget`: 9 added requirements (usage via `fn_uso_de_ia`, plan AI ceiling, `ref_kind` scoping of budget notices, `podeGastarComIa`, unpriced-call flag, ChatGPT subscription login, subscription model listing, economic tier, Gemini 3.x catalog).
- `ai-agents-runtime`: 5 added requirements (WhatsApp formatter, classifiers only on new messages, agenda re-read at closing, test run blocked on failed media, publish check for `custom`/`openai-assinatura`).
- `ai-flywheel`, `followup-flows`, `agent-cases-escalation`, `media-transcription`, `external-db`: added requirements for the new behavior. No existing requirement became false, so nothing is modified or removed.
- BREAKING: none.

## Capabilities

### New Capabilities
- none

### Modified Capabilities
- `ai-credentials-budget`
- `ai-agents-runtime`
- `ai-flywheel`
- `followup-flows`
- `agent-cases-escalation`
- `media-transcription`
- `external-db`

## Impact

Only `openspec/specs/*` after archive. Overlaps: the plan ceiling reads `fn_limite_do_plano` from `billing-plans` (change `sync-a-platform`); the `canal_pausado` notice is opened by the channel pause routes of `whatsapp-waha`.
