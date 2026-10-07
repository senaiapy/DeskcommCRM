## Purpose
The automated gates a change must pass before it reaches `main`: the `verify` and `invariants` aggregate checks in `.github/workflows/ci.yml`, the `e2e` aggregate in `.github/workflows/e2e.yml`, the Postgres invariant suite run by `pnpm test:db`, and the repository-specific lints `lint:channels` and `lint:role-rank`. Image publication (`imagens-ok`) and build size (`build-and-size`) are part of `deploy`; the invariants' content (RLS isolation, vocabulary) belongs to the capabilities they protect, e.g. `multi-tenancy-rls`.

## ADDED Requirements

### Requirement: CI runs on every pull request and on pushes to main
`.github/workflows/ci.yml` SHALL trigger on `pull_request` and on `push` to `main`, and `.github/workflows/e2e.yml` SHALL trigger on `pull_request` and `push` targeting `main`.

#### Scenario: Pull request opened
- **WHEN** a pull request is opened against `main`
- **THEN** both the `ci` and the `e2e` workflows start

### Requirement: verify-parte runs the static and unit gates
The `verify-parte` job SHALL run, with a 15-minute timeout, `pnpm cercas`, `pnpm typecheck`, `pnpm lint`, `pnpm lint:channels`, `pnpm lint:role-rank`, `pnpm checar:colisao-de-migration`, `pnpm test:unit --project produto` sharded in 3 parts and `pnpm test:shell`.

#### Scenario: Channel provider named in a feature file
- **WHEN** a pull request adds a provider name to a file under `app/` outside `lib/channels/`
- **THEN** the `Channel provider leak` step of `verify-parte` fails

### Requirement: verify is the required aggregate of the shards
The job named exactly `verify` SHALL run with `if: always()` after `verify-parte` and SHALL fail unless `needs.verify-parte.result` is `success`, so skipped or cancelled shards also fail the required check.

#### Scenario: One shard cancelled
- **WHEN** shard 2 of `verify-parte` is cancelled
- **THEN** the `verify` check fails

### Requirement: invariants runs the Postgres suite per supported major
The `invariants-majors` job SHALL run `pnpm test:db` and then `pnpm test:db:update` against `TEST_DB_IMAGE = pgvector/pgvector:pg<major>` for each major from `invariants-alcance` (15 and 17 outside pull requests or when the PR reaches the Postgres floor, 17 otherwise) with `fail-fast: false`, and the job named exactly `invariants` SHALL fail unless that matrix succeeded.

#### Scenario: Push to main
- **WHEN** a commit is pushed to `main`
- **THEN** the invariant suite runs on both pg15 and pg17 and `invariants` is green only if both pass

### Requirement: test:db applies the baseline twice and refuses to run without vitest
`scripts/test-db.sh` SHALL exit 1 before starting any container when `vitest` is not on `PATH`, SHALL default `TEST_DB_IMAGE` to `pgvector/pgvector:pg15`, SHALL apply `supabase/baseline.sql` in install mode and again in update mode with `ON_ERROR_STOP=1`, and SHALL then run `tests/invariants/**` through `vitest.db.config.ts`.

#### Scenario: Missing vitest
- **WHEN** `bash scripts/test-db.sh` runs without `vitest` on `PATH`
- **THEN** it prints an error and exits 1 without applying the baseline

### Requirement: Invariant suite configuration
`vitest.db.config.ts` SHALL include only `tests/invariants/**/*.test.ts`, run files sequentially (`fileParallelism: false`) with a 30 s test timeout and 60 s hook timeout, and load `tests/db/banco-limpo-por-arquivo.ts` as setup file, while `vitest.config.ts` (used by `test:unit`) excludes that tree.

#### Scenario: Unit run does not exercise RLS
- **WHEN** `pnpm test:unit` runs
- **THEN** no file under `tests/invariants/` is executed

### Requirement: lint:channels is a shrinking ratchet
`scripts/lint-channels.ts` SHALL scan `app`, `lib`, `components` and `workers`, allow provider names only under `lib/channels/`, and exit 1 both when a file outside the known-debt list names a provider and when a known-debt entry no longer names one.

#### Scenario: Debt entry cleaned but not removed from the list
- **WHEN** a file listed in the known debt no longer names a provider
- **THEN** `pnpm lint:channels` exits 1 asking for the entry to be deleted

### Requirement: lint:role-rank forbids hand-rolled role checks in routes
`scripts/lint-role-rank.ts` SHALL scan every non-test `.ts`/`.tsx` file under `app/api` and exit 1 when any contains `ROLE_RANK[`, directing authors to `requireRole` or `roleAtLeast`.

#### Scenario: Route compares ROLE_RANK directly
- **WHEN** a route file contains `ROLE_RANK[user.role] >= ROLE_RANK.manager`
- **THEN** `pnpm lint:role-rank` exits 1 listing that file

### Requirement: Every e2e spec is either run or declared out of CI
`tests/unit/e2e-cobertura-completa.test.ts` (executed by `test:unit`, hence by `verify`) SHALL fail when a `tests/e2e/*.spec.ts` file is in none of the spec lists of `.github/workflows/e2e.yml`, and the job named exactly `e2e` SHALL run with `if: always()` after `e2e-alcance` and `e2e-parte` and fail unless `e2e-alcance` succeeded and `e2e-parte` succeeded, accepting skipped parts only on a `pull_request` whose reach output is `nao`.

#### Scenario: New spec not wired into CI
- **WHEN** a pull request adds `tests/e2e/nova.spec.ts` without listing it in `e2e.yml`
- **THEN** the `verify` check fails on `e2e-cobertura-completa`

### Requirement: Local governance shortcut
The `gov:verify` script in `package.json` SHALL run `pnpm typecheck`, `pnpm lint`, `pnpm lint:channels`, `pnpm lint:role-rank` and `pnpm test:unit` in sequence, stopping at the first failure, and SHALL NOT run `test:db` or `test:e2e`.

#### Scenario: Schema change verified only with gov:verify
- **WHEN** a contributor runs `pnpm gov:verify` after changing `supabase/baseline.sql`
- **THEN** the invariant suite is not exercised and only `pnpm test:db` would cover it
