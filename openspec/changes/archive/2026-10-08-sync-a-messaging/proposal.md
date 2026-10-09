## Why

The messaging specs were written against upstream commit `477b77678`. HEAD is 491 upstream commits later and the messaging code drifted: pausing a connection now opens a Central notice (migration 0589), WAHA sends store the canonical external id and memoize positive check-exists answers, a conversation started from the paired phone creates the deal, the closed-window template preview renders the whole message, starting a conversation lets the user pick the number, the inbox labels the channel, automation rules can be scoped to one webhook source, and webhook sources carry their own form fields with numeric validation (migration 0595).

## What Changes

- Spec-only: no product code changes.
- New requirements in `whatsapp-waha` (4), `whatsapp-official-meta` (1), `inbox-conversations` (2), `automation-rules` (2), `inbound-webhooks` (2).
- No existing requirement became false; none is modified or removed.
- BREAKING: none.

## Capabilities

### New Capabilities
(none)

### Modified Capabilities
- `whatsapp-waha`
- `whatsapp-official-meta`
- `inbox-conversations`
- `automation-rules`
- `inbound-webhooks`

## Impact

Only `openspec/specs/*` after archive. No API, schema or runtime change.
