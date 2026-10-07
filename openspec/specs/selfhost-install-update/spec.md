# selfhost-install-update Specification

## Purpose
Self-host installation and update of DeskcommCRM on a VPS: the Ubuntu installers and the `hostgator-setup-kit/` scripts, the published images and compose profiles, and the in-app "update now" flow in which the app only records a request (`system_update_runs`) and a host cron agent (`hostgator-setup-kit/agent.sh`) executes `update.sh` and reports back. The backup taken during an update is detailed in `backup-restore`, the health probe the update waits on in `health`, and the scheduler container's cron table in `jobs-scheduler`.

## Requirements

### Requirement: Version state is readable by any session but actionable only by the server owner
`GET /api/v1/system/version` SHALL return 401 `unauthenticated` without a session, SHALL return only `{ current_version, is_owner: false }` when `is_platform_admin` is false, and SHALL return `latest_version`, `update_available`, `off_release`, `compare_failed`, `has_known_release`, `agent_online`, `just_updated`, `notes` and the latest `run` from `system_version` (id = 1) and `system_update_runs` to a platform admin.

#### Scenario: Regular member sees only the installed version
- **WHEN** a logged-in user whose `is_platform_admin` is false calls `GET /api/v1/system/version`
- **THEN** the response is 200 with only `current_version` and `is_owner: false`

#### Scenario: Anonymous call is refused
- **WHEN** `GET /api/v1/system/version` is called without a session
- **THEN** the response is 401 with error code `unauthenticated`

### Requirement: Derived update availability and agent liveness
`GET /api/v1/system/version` SHALL set `update_available` true only when `latest_version` is non-empty, differs from the running version and the run is not inside the post-success window (`RUN_STALE_AFTER_MS` = 15 minutes), and SHALL set `agent_online` true only when `system_version.agent_last_seen_at` is less than 24 hours old.

#### Scenario: Host agent silent for more than a day
- **WHEN** `system_version.agent_last_seen_at` is older than 24 hours
- **THEN** the owner response carries `agent_online: false`

#### Scenario: Success just finished and host has not reported yet
- **WHEN** the latest run has `status = 'success'` with `finished_at` less than 15 minutes ago and `system_version.updated_at` is earlier than that `finished_at`
- **THEN** the response carries `just_updated: true` and `update_available: false` without promoting `to_version` to `current_version`

### Requirement: A stale dispatched run is reported as unknown, never stored as unknown
The version route SHALL report a run whose stored `status` is `dispatched` and whose `dispatched_at` is older than 15 minutes as `status: "unknown"`, while the `system_update_runs.status` CHECK only admits `dispatched`, `success`, `failed` and `failed_rolled_back`.

#### Scenario: Agent died mid-update
- **WHEN** the latest run is still `dispatched` 16 minutes after `dispatched_at`
- **THEN** `GET /api/v1/system/version` returns `run.status = "unknown"` and the row in `system_update_runs` keeps `status = 'dispatched'`

### Requirement: Requesting an update creates exactly one dispatched run
`POST /api/v1/system/update` SHALL pass `requireSupportWrite` and `requirePlatformAdminEscrita` (full platform-admin scope, no pending MFA), SHALL return 409 `state_conflict` when `latest_version` is empty or equals `current_version`, and SHALL insert a `system_update_runs` row with `status = 'dispatched'`, `from_version`, `to_version` and `requested_by`, auditing `system.update_requested`.

#### Scenario: Owner clicks update with a newer release announced
- **WHEN** a full-scope platform admin calls `POST /api/v1/system/update` and `system_version.latest_version` differs from `current_version`
- **THEN** a `system_update_runs` row with `status = 'dispatched'` is created and the response carries `run_id` and `to_version`

#### Scenario: Already on the latest version
- **WHEN** `latest_version` equals `current_version`
- **THEN** the response is 409 `state_conflict` and no run is inserted

### Requirement: Only one dispatched run at a time, with stale-run recovery
The update route SHALL return 409 `state_conflict` while a non-stale `dispatched` run exists, SHALL close a `dispatched` run older than 15 minutes as `failed` with `finished_at` and a `log_tail` explanation before inserting the new one, and SHALL map a `23505` violation of the partial unique index `uniq_system_update_runs_dispatched` to 409 `state_conflict`.

#### Scenario: Double click
- **WHEN** two `POST /api/v1/system/update` requests race and the second insert hits `uniq_system_update_runs_dispatched`
- **THEN** the second response is 409 `state_conflict`

#### Scenario: Abandoned run unblocks a new request
- **WHEN** the only `dispatched` run is older than 15 minutes and the owner requests an update
- **THEN** that run becomes `failed` with `finished_at` set and a new `dispatched` run is created

### Requirement: Host agent endpoint authenticated by the internal secret
`POST /api/v1/system/agent` SHALL accept only `Authorization: Bearer` (or `x-cron-secret`) matching `INTERNAL_CRON_SECRET` or `INTERNAL_SECRET` by constant-time comparison, returning 401 `unauthorized` otherwise and 422 `validation_failed` for a body that is not one of `heartbeat`, `run_progress` or `run_result`.

#### Scenario: Wrong secret
- **WHEN** the agent posts with a bearer token that matches neither secret
- **THEN** the response is 401 `unauthorized` and nothing is written

### Requirement: Heartbeat records the host state and answers with a boolean
A `heartbeat` payload SHALL update the singleton `system_version` row (id = 1, CHECK id = 1) with `current_version`, `current_sha`, `off_release`, `latest_version`, `compare_failed`, `has_known_release`, `changelog_raw`, `agent_last_seen_at` and `updated_at`, and SHALL respond `{ update_requested, run_id }` derived from the single `dispatched` run, returning 500 `internal_error` when that lookup fails.

#### Scenario: Owner requested an update
- **WHEN** the agent sends a `heartbeat` while a `dispatched` run exists
- **THEN** the response is `update_requested: true` with that run's `run_id`

### Requirement: Run outcomes are write-once
`run_progress` SHALL set `last_step` (`backup`, `codigo` or `banco`) only on a `dispatched` run (409 `state_conflict` otherwise), and `run_result` SHALL move a run only from `dispatched` to `success`, `failed` or `failed_rolled_back` (409 `invalid_state_transition` otherwise), setting `log_tail` and `finished_at` and auditing `system.update_finished`.

#### Scenario: Agent reports the same result twice
- **WHEN** a `run_result` arrives for a run whose `status` is already `success`
- **THEN** the response is 409 `invalid_state_transition` and the stored outcome is unchanged

### Requirement: The host agent runs one update at a time and rolls back on failure
`hostgator-setup-kit/agent.sh` SHALL exit 0 silently when both `INTERNAL_CRON_SECRET` and `INTERNAL_SECRET` are empty or when `NEXT_PUBLIC_APP_URL` is empty, SHALL take a non-blocking `flock` on `.update.lock` before running `update.sh`, SHALL report `failed` without touching containers when `update.sh` exits with `REFUSED_RC` (3), and SHALL restart the previously running app/worker/scheduler images and report `failed_rolled_back` on any other non-zero exit.

#### Scenario: Update refused by preflight
- **WHEN** `update.sh` exits with code 3
- **THEN** the agent reports `status: "failed"` and does not restart any container

#### Scenario: New version fails health check
- **WHEN** `update.sh` exits with code 1 and the previous app image is known
- **THEN** the agent brings the previous images back up and reports `status: "failed_rolled_back"`

### Requirement: update.sh targets the latest published release with backup, preflight and health gate
`hostgator-setup-kit/update.sh` SHALL install the latest published stable release (or the `--to <tag>` given), SHALL refuse with exit code 3 before any backup or container stop when the preflight fails, SHALL run `backup.sh` before `git checkout` unless `--skip-backup`, and SHALL exit 1 when the app does not report `healthy` or `degraded` via `/api/v1/health` or any compose service is down after `up -d`.

#### Scenario: Same tag without --force
- **WHEN** `update.sh` runs, the checked-out tag equals the target and the local image digest matches the registry
- **THEN** it reports the installation is already current and exits 0 without touching the database

#### Scenario: Worker did not come up
- **WHEN** the app answers healthy but `servicos_fora_do_ar` lists a service
- **THEN** `update.sh` exits 1 so the agent rolls back

### Requirement: Published images, pinned channel and optional voice/telephony profiles
`docker-compose.prod.yml` SHALL run the `app` service from `${APP_IMAGE:-ghcr.io/melgarafael/deskcommcrm:stable}` with `pull_policy: ${APP_PULL_POLICY:-always}`, SHALL gate `wacalls` behind profile `voz` and `asterisk` and `voice-agent` behind profile `telefonia`, and the `Dockerfile` SHALL bake the build argument `APP_VERSION` into the runtime environment.

#### Scenario: Default production stack
- **WHEN** `docker compose -f docker-compose.prod.yml up -d` runs without `--profile`
- **THEN** `wacalls`, `asterisk` and `voice-agent` are not started

### Requirement: Installation snapshot for the onboarding screen
`GET /api/v1/system/instalacao` SHALL require `requireRole("manager")` and return the installation snapshot built by `lib/instalacao/retrato.ts` for the active organization, and SHALL require `requireRole("admin")` before performing the paid credit probe requested by `?provar=1`, reading the key from the org credential or from `ANTHROPIC_API_KEY`, `OPENAI_API_KEY` or `OPENROUTER_API_KEY`.

#### Scenario: Manager asks for the credit probe
- **WHEN** a user with role `manager` calls `GET /api/v1/system/instalacao?provar=1`
- **THEN** the admin gate refuses the request and no model call is made
