## Why
DeskcommCRM's voice and third-party integration surface (WhatsApp calls through WaCalls, SIP telephony through Asterisk,
paid-ads attribution and conversions, Nuvemshop, external databases, Google Calendar sync) exists only as code and
free-form docs under `docs/specs/`. Closing these OpenSpec coverage gaps lets the catalog be compared side by side with
TEMPLATE_CRM, TEMPLATE_CRM_V1 and TEMPLATE_ERP_v1; capability ids follow
`/home/gamba/PROJECTS/CRM_ERP_V1/openspec-capability-map.md` (group "voice / integrations").

## What Changes
- Documentation only: retro-specs of code that already exists. No code, schema, migration, environment or compose change.
- Every requirement was checked against a handler, library, worker, migration or compose file (evidence in `tasks.md`).
- `payment-gateways` from the capability map is not specified: no payment gateway code exists in this repository (only
  stale comments mention Asaas).
- **BREAKING:** none.

## Capabilities

### New Capabilities
- `voice-whatsapp-calls`: per-organization opt-in WhatsApp voice calls through WaCalls (pairing, dial, answer, hang-up, event bridge into `voice_calls`).
- `voip-asterisk`: SIP trunk, phone numbers and the Asterisk + OpenAI Realtime voice-agent worker (`provider='sip'` rows in `voice_calls`).
- `ads-attribution`: Meta/Google Ads click capture, trackable links, attribution read and idempotent purchase/stage conversion sending, Meta insights read.
- `nuvemshop`: Nuvemshop OAuth connection, webhook subscription and HMAC receivers, including the three LGPD webhooks.
- `external-db`: read-only connector to an external PostgreSQL with network policy, live introspection and AI read tools.
- `google-calendar`: per-user Google Calendar OAuth connection, calendar selection and the refresh/sync/push crons.

## Impact
- Adds only files under `openspec/changes/retro-voice-integrations/` (`.openspec.yaml`, `proposal.md`, `design.md`,
  `tasks.md`, `specs/<id>/spec.md`). No product file, no other change and nothing in `openspec/specs/` is touched.
