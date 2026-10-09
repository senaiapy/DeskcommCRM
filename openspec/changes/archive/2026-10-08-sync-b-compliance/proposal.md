## Why

The compliance specs were written against upstream commit `477b77678`. HEAD is 491 upstream commits later: the LGPD/RGPD data file became a titular copy stripped of team-only material and is now linked for Brazilian subjects too, and the AI append-only tables gained age-based pruning in the daily retention cron (migration 0587, four new knobs). This change brings the catalog back in line with the code.

## What Changes

- Spec-only: no product code changes.
- `lgpd-privacy`: 1 added requirement (titular copy of `data.json`, link for every country). The RGPD art. 15 paragraphs route `PATCH /api/v1/settings/art15` is specified under `settings` (change `sync-a-platform`) and is not repeated here.
- `data-retention`: 1 added requirement (four new purge functions and knobs `AI_TELEMETRY_RETENTION_DAYS`, `PACING_LEDGER_RETENTION_DAYS`, `OUTBOUND_COPIES_RETENTION_DAYS`, `LEAD_CHECKPOINT_RETENTION_DAYS`).
- BREAKING: none.

## Capabilities

### New Capabilities
- none

### Modified Capabilities
- `lgpd-privacy`
- `data-retention`

## Impact

Only `openspec/specs/*` after archive.
