## Context
DeskcommCRM is a Supabase-based Next.js 16 CRM (supabase-js, Supabase Auth, Realtime, `supabase/baseline.sql` +
`supabase/migrations/`, `event_log` drained by crons and `workers/`). `openspec/specs/` is still empty for this repo,
and the sibling changes `retro-ai`, `retro-compliance-ops`, `retro-crm-core`, `retro-messaging` and `retro-platform`
cover the neighbouring capability ids named in each Purpose (`whatsapp-channel`, `agenda-scheduling`, `lgpd`,
`ai-agents`, `mcp-server`, CRM core and queue/platform capabilities).

## Evidence method
For each capability the route handlers, `lib/` modules, workers, crons, compose services and baseline/migration DDL were
read. A requirement was kept only when a `file:line` backs it; wording follows the code, not `docs/specs/*`. Each task in
`tasks.md` lists the lines read.

## Decisions
- `voice_calls` is shared: WhatsApp calls (`provider` wacalls, `/api/v1/voice/*`) belong to `voice-whatsapp-calls`;
  SIP rows (`provider='sip'`, `/api/v1/calls`, `/api/v1/voip/trunk`, `/api/v1/phone-numbers`, `workers/voice-agent`)
  belong to `voip-asterisk`.
- `google-calendar` is separable from `agenda-scheduling`: it has its own OAuth routes (`/api/v1/agenda/google/*`),
  tables (`calendar_connections`, `calendar_connection_calendars`, `calendar_external_events`, `calendar_oauth_nonces`),
  `lib/agenda/google/*` and three crons. Appointment booking/availability is left to `agenda-scheduling`.
- `ads-attribution` includes the conversion retry route `POST /api/v1/leads/{id}/conversion/retry` because it only
  feeds the conversion handlers; lead state transitions themselves are left to CRM core.
- `nuvemshop` stops at opening `lgpd_requests`; redaction/export processing is `lgpd`.
- `external-db` includes the two MCP tools because the module exists for them; the MCP runtime is left to `mcp-server`.
- `payment-gateways` dropped: no `lib/asaas`, no gateway routes, crons or tables; only comments in
  `lib/campanhas/rodizio.ts:16`, `lib/campanhas/rodada.ts:137` and `app/api/v1/cron/campaign-worker/route.ts:7` mention
  Asaas and an `asaas-reconcile` cron that does not exist.
- Left out (not specified): WaCalls WebRTC SDP proxy details (`/api/v1/voice/calls/{id}/webrtc`), AudioSocket framing
  internals, Meta insights table math, Google Meet link extraction, Nuvemshop onboarding step UI.

## Doc/code inconsistencies found
- `openspec/config.yaml` and `CLAUDE.md` correctly describe Supabase here (unlike the TEMPLATE_CRM fork); no `lib/supabase` name-shim issue applies to this repo.
- `app/api/v1/calls/route.ts:5` says the voice-agent finds outbound calls on `StasisStart` with `args[0] === "outbound"`, but `asterisk/extensions.conf` and `workers/voice-agent/index.ts` (header and `handleChannelDestroyed`) state outbound calls never pass through Stasis; `handleStasisStart` treats every call as inbound.
- `.env.example:177` sets `AUDIOSOCKET_PORT=8090` while `asterisk/extensions.conf:35,41` dials `voice-agent:9092` and the worker defaults to 9092 (`workers/voice-agent/index.ts:67`); the voice-agent compose service uses `env_file: .env`, so copying the example breaks inbound/outbound audio.
- `ARI_URL`/`ARI_USERNAME`/`ARI_PASSWORD` are injected only into the `voice-agent` service (`docker-compose.prod.yml:397`), yet `POST /api/v1/calls` runs `originateCall` in the `app` container (`lib/voip/ariClient.ts:21-23`), and `.env.example` documents ARI credentials as "not read by the app".
- `guardarTrunk` stores `endpoint_name` `org-<id>-trunk-endpoint` without a `PJSIP/` prefix (`lib/voip/guardar-trunk.ts:20`) while `originateCall` splits it on `/` (`lib/voip/ariClient.ts:90`); `asterisk/pjsip.conf` is a static mounted file, so trunk settings saved in `voip_trunk_settings` never reach Asterisk.
- `VOIP_TRUNK_ENDPOINT` and `VOIP_DEFAULT_CALLER_ID` are read via `process.env` but are absent from `lib/env.ts` and `.env.example`.
- `app/api/v1/voice/opt-in/route.ts` header cites migration 0234 for `org_voice_calls` RLS; 0234 only adds `voice_calls` to the realtime publication, the table is in `0236_opt_in_de_chamada_de_voz`. 0234's comment calls itself a forward-fix of "0206" (the file is 0233).
- `GET /api/v1/voice/calls/history` does not filter `provider`, so SIP calls appear in the WhatsApp call history, while `GET /api/v1/calls` filters `provider='sip'`.
- `docs/specs/06-spec-nuvemshop-lgpd.md` describes an `EcommercePlatformAdapter`, sync workers and order/product ingestion; the code only logs webhooks and emits `nuvemshop.<event>` events that no handler consumes, and nothing emits `nuvemshop.product_synced`, which `workers/rag-indexer.handler.ts:16` listens for.
- `app/actions/integrations/disconnectNuvemshop.ts:6` says disconnect "clears tokens"; the update (line 48) only sets `status`/`status_reason`, and webhooks stay registered at the store.
- The Nuvemshop callback stores the app client secret as the per-tenant `webhook_secret_encrypted` and generates `webhook_path_token`, but the registered webhook URLs do not use the path token.

## Debts
- No handler consumes `nuvemshop.order_*`, `nuvemshop.product_*` or `nuvemshop.app_uninstalled` events (`app/uninstalled` does not mark the integration disconnected).
- `POST /api/v1/calls` takes `toNumber` from the body (WhatsApp dialing reads the phone from the contact) and does not check `is_blocked` before originating.
- `mapStatusParaApi` maps `end_reason='contato_pessoal'` to `completed`; harmless only because personal contacts are filtered from the list.
- Missed SIP calls do not create `agent_inbox_items` (only WaCalls does).
- Google Calendar has no Google push-notification channel; "push" cron means outbound reconciliation, inbound is polling.
