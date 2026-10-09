## Why

The operations specs were written against upstream commit `477b77678`. HEAD is 491 upstream commits later: the scheduler stopped writing the cron secret into the crontab, the message webhook drains only its follow-up handlers when the worker drains (`EVENT_LOG_WORKER_DRAINS`), the worker health port is bound to loopback, the installer validates Supabase by GoTrue, stops guessing the Traefik network, seeds the CPF key and cleans its temp file, the update warning depends on the Supabase kind, backups became owner-only, the diagnostic prints the agent's real last failure, CI gained a fourth `verify-parte` part, majors are cut only on request, e2e keeps its evidence, and agent jobs record queue wait and wall time. This change brings the catalog back in line with the code.

## What Changes

- Spec-only: no product code changes.
- `event-bus-workers`: 2 added; `local-dev-stack`: 1 added; `selfhost-install-update`: 5 added; `backup-restore`: 2 added; `health`: 1 added; `ci-quality-gates`: 3 added; `observability`: 1 added. No existing requirement became false.
- BREAKING: none.

## Capabilities

### New Capabilities
- none

### Modified Capabilities
- `event-bus-workers`
- `local-dev-stack`
- `selfhost-install-update`
- `backup-restore`
- `health`
- `ci-quality-gates`
- `observability`

## Impact

Only `openspec/specs/*` after archive. New environment variables documented: `EVENT_LOG_WORKER_DRAINS`, `CRON_AUTH_DIR` (scheduler only).
