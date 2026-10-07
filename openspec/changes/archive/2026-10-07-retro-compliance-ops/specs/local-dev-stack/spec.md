## Purpose
The developer and local-VM stack of DeskcommCRM: Supabase run locally by the Supabase CLI and seeded from `supabase/baseline.sql` (`scripts/local-supabase.sh`), the application services built from source in `docker-compose.local.yml` and orchestrated by `scripts/local-stack.sh` (`pnpm local:*`), the one-shot `ubuntu-local-installer.sh`, the `pnpm dev:crons` loop, and the in-request pipeline accelerator in `lib/dev/kick-local-pipeline.ts`. The production install is covered by `selfhost-install-update`; the scheduler crons themselves by `jobs-scheduler`; event draining by `event-bus-workers`.

## ADDED Requirements

### Requirement: Local Supabase is seeded from the baseline, not the migration chain
`scripts/local-supabase.sh start` SHALL move `supabase/migrations` aside, run `supabase start`, restore the directory (also on exit via trap), and then apply `supabase/baseline.sql` with `psql -v ON_ERROR_STOP=1` from a `postgres:15-alpine` container after creating the `uuid-ossp`, `pgcrypto`, `vector`, `citext` and `pg_trgm` extensions.

#### Scenario: Fresh local database
- **WHEN** `scripts/local-supabase.sh start` runs on a machine without a local Supabase
- **THEN** Supabase starts without replaying `supabase/migrations/*` and the baseline is applied with errors stopping the run

### Requirement: Local Supabase script commands
`scripts/local-supabase.sh` SHALL accept `start`, `status` (JSON from `supabase status -o json`), `apply-baseline` and `stop`, and SHALL print usage and exit 2 for any other argument; `apply-baseline` SHALL fail when the status has no `DB_URL`.

#### Scenario: Unknown subcommand
- **WHEN** `scripts/local-supabase.sh reset` is run
- **THEN** it prints `Uso: ... {start|status|apply-baseline|stop}` and exits 2

### Requirement: Local stack orchestration
`scripts/local-stack.sh` SHALL accept `up`, `down`, `restart`, `status`, `logs`, `supabase-status` and `reset`; `up` SHALL start local Supabase when its status fails, ensure `.env.local` via `scripts/local-env.sh ensure`, ensure `NUVEMSHOP_OAUTH_ENCRYPTION_KEY` in the env file and in `private.app_secrets`, and run `docker compose -f docker-compose.local.yml up -d --build`; `down` SHALL stop the compose services and then local Supabase.

#### Scenario: pnpm local:up on a configured VM
- **WHEN** `pnpm local:up` runs with a non-empty `.env.local`
- **THEN** local Supabase is running and the `app`, `worker`, `scheduler`, `waha`, `redis` and `srh` services are built and started

### Requirement: Local stack refuses to run without its env file
`scripts/local-stack.sh` SHALL exit 1 with a message to run `./ubuntu-local-installer.sh` first when the env file (`LOCAL_ENV_FILE`, default `.env.local`) is missing or empty, for `restart`, `status`, `logs` and `reset`, and for `up` after `scripts/local-env.sh ensure` has had the chance to generate it.

#### Scenario: Status before installation
- **WHEN** `pnpm local:status` runs without `.env.local`
- **THEN** the script exits 1 telling the user to run `./ubuntu-local-installer.sh`

### Requirement: Local compose builds from source and keeps the WAHA dashboard on loopback
`docker-compose.local.yml` SHALL build `app` (image `deskcomm-app:local`, `pull_policy: never`, build arg `APP_VERSION` default `local`), `worker` and `scheduler` from the repository Dockerfiles, SHALL publish the app on `${APP_PORT:-3000}`, and SHALL publish WAHA only on `127.0.0.1:3030`.

#### Scenario: Another machine on the VM network
- **WHEN** a host on the same network connects to the VM's port 3030
- **THEN** the connection is refused because WAHA is bound to 127.0.0.1

### Requirement: Local installer protects an existing cloud env file
`ubuntu-local-installer.sh` SHALL copy an existing `.env.local` that lacks `DESKCOMM_ENV_MODE=local` to `.env.local.cloud-backup` before writing a new local `.env.local`, and SHALL finish by running `./scripts/local-stack.sh up`.

#### Scenario: Clone previously pointed at the cloud
- **WHEN** the installer finds a `.env.local` without `DESKCOMM_ENV_MODE=local`
- **THEN** that file is preserved as `.env.local.cloud-backup` and a warning is printed

### Requirement: Development cron loop
`scripts/dev-crons.ts` (`pnpm dev:crons`) SHALL POST, every `DEV_CRON_INTERVAL_MS` milliseconds (default 15000), to `/api/v1/cron/prospecting`, `/api/v1/cron/event-log-drain` and `/api/v1/cron/followup-flow-worker` on `NEXT_PUBLIC_APP_URL` (default `http://localhost:3000`) with `Authorization: Bearer` set to `INTERNAL_CRON_SECRET` or else `INTERNAL_SECRET`, and SHALL exit 1 when neither secret is set.

#### Scenario: Secret missing
- **WHEN** `pnpm dev:crons` runs without `INTERNAL_CRON_SECRET` and `INTERNAL_SECRET`
- **THEN** the process exits 1 naming the missing variable

#### Scenario: Database host warning
- **WHEN** the loop starts
- **THEN** it logs the target URL and the Supabase host and warns that pointing at the VPS database competes with production for `event_log` and `job_queue`

### Requirement: In-request pipeline accelerator never fails the caller
`acelerarPipelineDeEventos` and `kickLocalPipeline` in `lib/dev/kick-local-pipeline.ts` SHALL, inside the inbound/webhook request, drain `event_log` via `drainEventLog` and advance at most 6 rounds of the contact's `active` `followup_enrollments` (filtered by `organization_id`), and SHALL catch every error and log it with `logger.warn` instead of propagating it.

#### Scenario: Drain throws during a webhook
- **WHEN** `drainEventLog` throws while `POST /api/v1/webhooks/in/[token]` is being handled
- **THEN** a `[dev.pipeline]` warning is logged and the webhook response is not turned into a 5xx
