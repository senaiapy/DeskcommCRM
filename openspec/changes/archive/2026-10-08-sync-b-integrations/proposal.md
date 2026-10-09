## Why

The integration, notification and report specs were written against upstream commit `477b77678`. HEAD is 491 upstream commits later: the inbound-message push now reaches only the responsible people who can see the conversation (migration 0612) and the browser asks the server whether to alert, a per-channel report landed (0590) and the attendant/channel functions cut by the window (0596), the funnel and loss reports read in pages, the voice agent records its usage, Google Ads and Nuvemshop OAuth returns became single-use, the Google client secret is format-checked and the privacy page declares the Google data use, and official-catalog extensions install again (0511). This change brings the catalog back in line with the code.

## What Changes

- Spec-only: no product code changes.
- `notifications`: the requirement "Inbound message push goes to the organization" is REMOVED (no longer true) and replaced by an ADDED requirement with the new recipient rule; one added requirement for `GET /api/v1/conversations/{id}/aviso-de-mensagem`.
- `metrics-reports`: 3 added; `voip-asterisk`: 1 added; `ads-attribution`: 1 added; `nuvemshop`: 1 added; `google-calendar`: 2 added; `extensions`: 1 added.
- BREAKING: none.

## Capabilities

### New Capabilities
- none

### Modified Capabilities
- `notifications`
- `metrics-reports`
- `voip-asterisk`
- `ads-attribution`
- `nuvemshop`
- `google-calendar`
- `extensions`

## Impact

Only `openspec/specs/*` after archive. The voice usage row feeds the usage screen of `ai-credentials-budget`.
