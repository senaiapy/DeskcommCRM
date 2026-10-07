## Context

Retro-spec of the compliance and operations layer of DeskcommCRM (Next.js 16 + Supabase, self-hosted with the hostgator setup kit). Baseline: `main` at 4eb556a on 2026-10-07.

## How evidence was gathered

- Seeds: `CLAUDE.md`, `docs/doctrine/{packaging,extensoes,versionamento}.md`, `docs/specs/06-spec-nuvemshop-lgpd.md`, `07-spec-events-workers.md`, `08-spec-deploy-observability.md`, `extensoes-declarativas-v1.md`, `modulo-instalado-onda-2.md`, `docs/ATUALIZANDO.md`.
- Authority: the code — route handlers, `lib/*`, `workers/*`, `hostgator-setup-kit/*.sh`, `docker-compose*.yml`, `docker/scheduler/entrypoint.sh`, `.github/workflows/*.yml` and the last definition of each SQL function in `supabase/baseline.sql`. Requirements that could not be backed were dropped; each task cites the lines read.
- Two reviewers split the sixteen capabilities; `openspec validate retro-compliance-ops --strict` was run by each.

## Decisions

- `honorarios` gets its own id `honorarios-module` (three routes, a page, its own SQL provisioner and pay function) and `installable-modules` describes only the generic catalog/installer.
- There is no backup/restore HTTP API; `backup-restore` is spec'd on the shell scripts only.
- `lib/agent-engine/health/*` and cron `channel-health` are channel health (`whatsapp-waha`), not `health`.

## Inconsistencies found (docs vs code, code vs code)

- LGPD export/redact workers write status `pending_review`, which `lgpd_requests_status_check` (`supabase/baseline.sql:1642`) rejects; the error is not checked, so the request silently stays `processing`. Left out of the spec.
- `signPdfPades` (`lib/lgpd/pades-signer.ts`) always returns `signed_pades:false` even with `LGPD_SIGNING_KEY` set; the export PDF is not signed.
- The Nuvemshop redaction callback in `workers/lgpd-redact-worker.ts:62-76` is a stub returning `not_implemented`.
- `POST /api/v1/lgpd/requests/[id]/approve` documents 202 but returns 200 (and stores `status_code: 202` in the idempotency record).
- Anonymize still audits `storage_media_deletion: "deferred_epic_08"` although media is queued by trigger since migration 0391; `CLAUDE.md` says media is removed during the cascade, but deletion is asynchronous via `storage_redaction_queue`; `lgpd.consent_changed` is listed in `lib/audit/actions.ts` but never emitted.
- Cron secrets are `INTERNAL_CRON_SECRET` / `INTERNAL_SECRET` (`lib/auth/cron-auth.ts`), not `CRON_SECRET`; cron routes answer 403 while some comments and the scheduler message say 401.
- `storage-redaction` route header says "Vercel Cron"; it runs from `docker/scheduler/entrypoint.sh`.
- `CLAUDE.md` says `verify` = typecheck + lint + test:unit; CI `verify-parte` also runs `cercas`, `lint:channels`, `lint:role-rank`, `checar:colisao-de-migration`, `test:shell`, and `test:unit` runs only project `produto` in 3 shards.
- `scripts/backup-db.sh` writes custom-format `deskcomm-*.dump`; `hostgator-setup-kit/restore.sh` only accepts `db-*.sql.gz`, so nothing restores the former.
- `lib/dev/kick-local-pipeline.ts` lives under `dev/` but runs in production inbound paths (`webhooks/in/[token]`, `lib/waha/ingest.ts`, `lib/channels/pos-entrada.ts`).
- `lint-role-rank` error says "fora de lib/auth/" but scans only `app/api`.
- Honorarios table routes map only `42P01` to 409 `module_not_installed`; PostgREST reports a missing table as `PGRST205`, so before install they likely answer 500 (only the pay RPC path is spec'd).
- `alerts-platform` realtime channel is subscribed to and exported (`lib/realtime/channels.ts`) but nothing broadcasts on it.

## Declared debts

- Route handlers mint their own `randomUUID()` request id and ignore the `x-request-id` echoed by `proxy.ts`; `/api/v1/health` uses `NextResponse.json` instead of `ok()`.
- `lib/logger.ts` does no scrubbing; Sentry scrubbing does not cover stdout.
- `restore.sh` runs without `ON_ERROR_STOP` / single transaction; a fatal mid-restore leaves a partial database.
- Media retention runs only where `media_retention_enforced = true`; the bucket grows unbounded otherwise.
- Error codes `media_expired` and `bad_gateway` are used with `fail()` but not registered in `lib/api/errors.ts`; media routes use `console.error` instead of `logger`.
- Honorarios create POSTs ignore `Idempotency-Key`.
- Server-side Web Push ignores per-user preferences (stored only in browser `localStorage`); inbound pushes go to every subscription of the organization.
- `dev:crons` drives 3 of the scheduler's crons; `package.json` `db:migrate` is a TODO no-op; `update.sh --skip-backup` lets an update run without a backup.

## Risks / Trade-offs

- A retro-spec freezes current behaviour; fixing the items above is separate work with MODIFIED deltas.
