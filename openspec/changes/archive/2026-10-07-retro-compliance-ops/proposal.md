## Why

DeskcommCRM has no OpenSpec specs yet. This change records, as a retro-spec, what the compliance and operations layer already does (LGPD, retention, extensions and modules, event bus, self-host install/update, backup, health, metrics, notifications, storage, realtime, observability, CI and the local stack), so the four folders of CRM_ERP_V1 (DeskcommCRM, TEMPLATE_CRM, TEMPLATE_CRM_V1, TEMPLATE_ERP_v1) can be compared capability by capability with the shared ids of `openspec-capability-map.md`.

## What Changes

- Documentation only: sixteen new capability specs of code that already exists.
- No product code, schema, migration, route, compose file, script or environment variable is added or changed.
- Requirements the code does not meet were left out (e.g. the `pending_review` LGPD status, which the database CHECK rejects; see design.md).
- **BREAKING:** none.

## Capabilities

### New Capabilities
- `lgpd-privacy`: LGPD request ledger and deadlines, preview, approval, anonymize, export (PDF) and redact workers, media deletion queue, SLA watcher.
- `data-retention`: `lib/retencao/politica.ts` defaults and floors, the daily `data-retention` cron, `fn_expurgar_auditoria_vencida`, opt-in media retention, webhook log archive retention.
- `extensions`: declarative extension packages — catalogs, install, configuration, open, revert, remove, operation cancel, platform gate and error mapping.
- `installable-modules`: module catalog and install route over `lib/modulos` and the SQL installer.
- `honorarios-module`: the official honorarios module — contracts, installments, pay function and its SQL provisioner.
- `event-bus-workers`: `event_log` + `emit_event`, drain and dispatcher, cron auth, scheduler and worker containers; triggers never do HTTP.
- `selfhost-install-update`: hostgator setup kit, installers, `update.sh`, system version/update runs, host agent, compose profiles and images.
- `backup-restore`: `backup.sh` / `restore.sh` and the pre-update backup.
- `health`: `GET /api/v1/health`, container healthchecks, per-tenant admin health.
- `metrics-reports`: metrics (attendants, friction, funnel, lost) and reports (activities, finance, tags).
- `notifications`: Web Push subscriptions, VAPID, push delivery handler and service worker.
- `file-storage`: private bucket `whatsapp-media`, signed media URLs, storage redaction queue and cleanup.
- `realtime`: Supabase Realtime publication, realtime token endpoint and channel hooks.
- `observability`: Sentry configuration and scrubbing, structured logger, `X-Request-Id`.
- `ci-quality-gates`: GitHub Actions required jobs, `test:db` invariants, `lint:channels`, `lint:role-rank`, e2e coverage gate.
- `local-dev-stack`: `docker-compose.local.yml`, `scripts/local-supabase.sh`, `scripts/local-stack.sh`, `dev:crons`, local pipeline kick.

### Modified Capabilities
- none

## Impact

- Files: only `openspec/changes/retro-compliance-ops/**`.
- Dropped: none. `jobs-scheduler` (scheduler clock tick `app/api/v1/system/relogio/tick`) and `deploy` are left to their own ids; the per-number WhatsApp health circuit belongs to `whatsapp-waha`.
