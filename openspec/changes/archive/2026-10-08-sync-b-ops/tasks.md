## 1. Re-verify the operations capabilities against HEAD

- [x] 1.1 `event-bus-workers`: 2 added. Evidence: docker/scheduler/entrypoint.sh:135, docker/scheduler/entrypoint.sh:137, docker/scheduler/entrypoint.sh:143, lib/dev/kick-local-pipeline.ts:140, lib/dev/kick-local-pipeline.ts:157, lib/dev/kick-local-pipeline.ts:227, lib/event-log/drain.ts:36, lib/event-log/drain.ts:153, lib/event-log/dispatcher.ts:97, docker-compose.prod.yml:45, docker-compose.local.yml:24, lib/env.ts:318
- [x] 1.2 `local-dev-stack`: 1 added. Evidence: docker-compose.yml:84, docker-compose.local.yml:64
- [x] 1.3 `selfhost-install-update`: 5 added. Evidence: hostgator-setup-kit/install.sh:311, hostgator-setup-kit/install.sh:342, hostgator-setup-kit/install.sh:349, hostgator-setup-kit/install.sh:815, hostgator-setup-kit/install.sh:1523, hostgator-setup-kit/install.sh:1538, hostgator-setup-kit/_common.sh:2456, hostgator-setup-kit/_common.sh:2492, hostgator-setup-kit/install.sh:2418, hostgator-setup-kit/update.sh:852, hostgator-setup-kit/install.sh:236, hostgator-setup-kit/install.sh:2209, hostgator-setup-kit/_common.sh:571, hostgator-setup-kit/update.sh:838
- [x] 1.4 `backup-restore`: 2 added. Evidence: hostgator-setup-kit/backup.sh:18, hostgator-setup-kit/backup.sh:20, scripts/backup-db.sh:25, hostgator-setup-kit/backup.sh:76, hostgator-setup-kit/backup.sh:82
- [x] 1.5 `health`: 1 added. Evidence: hostgator-setup-kit/healthcheck.sh:59
- [x] 1.6 `ci-quality-gates`: 3 added. Evidence: .github/workflows/ci.yml:158, .github/workflows/ci.yml:241, .github/workflows/ci.yml:347, .github/workflows/ci.yml:348, .github/workflows/ci.yml:393, lib/release/fragmento.ts:47, lib/release/fragmento.ts:201, scripts/cortar-release.ts:99, .github/workflows/release.yml:27, .github/workflows/e2e.yml:2163
- [x] 1.7 `observability`: 1 added. Evidence: lib/agent-engine/obs/metrics.ts:30, lib/agent-engine/obs/metrics.ts:33, workers/agent-worker/main.ts:482

## 2. Validate

- [x] 2.1 `openspec validate sync-b-ops --strict --no-interactive` passes
