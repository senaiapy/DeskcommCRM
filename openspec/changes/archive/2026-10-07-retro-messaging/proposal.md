## Why

DeskcommCRM's messaging layer (WhatsApp via WAHA, the official Meta channel, social channels, the inbox, templates, routing, opt-out, campaigns, automation rules and webhooks) is documented only in `CLAUDE.md` and `docs/specs/03`, `13`, `21`, `22`, and part of that prose has drifted. This change writes a retro-spec of what the code does today, under the shared ids of `openspec-capability-map.md`, so messaging can be compared with TEMPLATE_CRM, TEMPLATE_CRM_V1 and TEMPLATE_ERP_v1.

## What Changes

- Documentation only: eleven new capability specs. No code, schema, env var or route changes.
- Specs follow the code where it disagrees with the docs; `design.md` lists every disagreement.
- **BREAKING:** none in the product. What breaks is reliance on three `CLAUDE.md` claims that the code does not meet: "campaign 1 msg/5s", "webhooks HMAC SHA512" as a general rule, and the existence of an outbound-webhook subscription system.

## Capabilities

### New Capabilities
- `whatsapp-waha`: WAHA sessions, pairing, webhook auth and ingest, dedupe, send pacing, groups, stuck-message recovery.
- `whatsapp-official-meta`: Meta Cloud API channel, `X-Hub-Signature-256` webhook, 24h window (`janela_fechada`), templates.
- `social-channels`: Instagram and Facebook inbox through the Zernio social provider.
- `inbox-conversations`: conversations and messages, claim/transfer/close/snooze, notes, drafts, media, realtime.
- `message-templates-quick-replies`: `message_templates` (quick replies) and `campaign_templates`.
- `routing-attendants`: routing modes, `routing-worker` cron, attendant availability and presence.
- `opt-out-consent`: `lib/opt-out/deteccao.ts`, block on ingest, agent guardrails, unblock, consent per purpose.
- `campaigns-broadcast`: campaign state machine, legal basis, recipient snapshot, suppressions, paced sending cron.
- `automation-rules`: rule CRUD, trigger vocabulary, event_log consumer, runs, postponement, resend.
- `inbound-webhooks`: webhook sources with `path_token`, signed lead capture, dedupe, capture history.
- `outbound-webhooks`: the `call_webhook` automation action — envelope, signing, SSRF guard, retries.

### Modified Capabilities
- None.

## Impact

- Files added: `openspec/changes/retro-messaging/**` only.
- No seed id dropped. One area found without a spec: the WhatsApp partner providers (`app/api/v1/channels/partner`, `graph-partner`), noted in `design.md` as a candidate id `whatsapp-partner-providers`.
