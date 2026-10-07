# data-retention Specification

## Purpose
Age-based purging of operational history: the retention policy interpreter in `lib/retencao/politica.ts`, the
daily `data-retention` cron with its batched `security definer` purge functions (including
`fn_expurgar_auditoria_vencida` for `api_audit_log`), the `media-retention` cron for message media, and the
`webhook-log-retention` cron for raw webhook bodies and lead captures. The append-only property of the audit
table is covered by `audit-log`; data-subject erasure is covered by `lgpd-privacy`; cron scheduling and the
cron secret are covered by `event-bus-workers`.

## Requirements

### Requirement: Retention knobs have a default and a floor
`interpretarRetencao` SHALL return the default days when the environment value is empty or not an integer and SHALL raise any value below the floor to the floor, each with a warning; the defaults/floors SHALL be `JOB_QUEUE_RETENTION_DAYS` 90/7, `AUDIT_LOG_RETENTION_DAYS` 1825/90, `CALENDAR_MIRROR_RETENTION_DAYS` 90/7, `CASE_CHAT_RETENTION_DAYS` 365/90, `PASSAGEM_RETENTION_DAYS` 1825/90, `CASE_ALERT_RETENTION_DAYS` 180/30, `PROSPECCAO_RETENTION_DAYS` 365/90, `JEV_OBSERVACOES_RETENTION_DAYS` 90/30, `DRAFT_RETENTION_DAYS` 30/7 and `GOLDEN_CANDIDATES_RETENTION_DAYS` 90/30.

#### Scenario: Value below the floor
- **WHEN** `AUDIT_LOG_RETENTION_DAYS=10`
- **THEN** the cron purges with 90 days and logs a warning naming the key and the floor

#### Scenario: Non-numeric value
- **WHEN** `JOB_QUEUE_RETENTION_DAYS=90dias`
- **THEN** the default of 90 days is used and a warning is logged

### Requirement: Daily history pruning cron
`GET|POST /api/v1/cron/data-retention` SHALL authorize with `autorizaCron` (403 `forbidden` otherwise) and drain, in batches of 1000 and at most 20 batches per purge, `fn_podar_fila_de_jobs`, `fn_expurgar_auditoria_vencida`, `fn_expurgar_espelho_da_agenda`, `fn_expurgar_nonces_de_oauth` (1 day), `fn_expurgar_conversa_do_caso_vencida`, `fn_expurgar_passagens_vencidas`, `fn_expurgar_avisos_de_caso_vencidos`, `fn_expurgar_prospeccao_vencida`, `fn_expurgar_observacoes_do_jev`, `fn_expurgar_candidatos_do_golden`, expired `conversation_drafts` and `fn_enfileirar_midia_vencida`, stopping each purge at the first incomplete batch.

#### Scenario: Backlog larger than one run
- **WHEN** more than 20 000 audit rows are past the retention
- **THEN** the run deletes 20 batches and reports `auditoria_tem_resto: true`

#### Scenario: Purge function failure
- **WHEN** one purge RPC returns an error
- **THEN** the response is 500 `internal_error` and an audit row `retention.sweep_run` with `falhou: true` is written

### Requirement: Audit purge function has its floor in the body
`public.fn_expurgar_auditoria_vencida(p_retencao_dias, p_limite)` SHALL be `security definer`, delete at most `least(greatest(p_limite,1),10000)` rows of `api_audit_log` ordered by `created_at` whose `created_at` is older than `greatest(coalesce(p_retencao_dias,1825),90)` days, return the deleted count, and be executable only by `service_role`.

#### Scenario: Caller asks for 30 days
- **WHEN** the function is called with `p_retencao_dias = 30`
- **THEN** only rows older than 90 days are deleted

#### Scenario: Authenticated role
- **WHEN** a session with role `authenticated` calls the function through the REST API
- **THEN** execution is denied

### Requirement: Sweeps audit only when they had an effect
The `data-retention` and `media-retention` crons SHALL write an `api_audit_log` row `retention.sweep_run` (organization null, counts in `metadata`) only when at least one row was deleted or enqueued, or when the sweep failed.

#### Scenario: Nothing expired
- **WHEN** a run finds no expired row in any purge
- **THEN** the response is 200 with zero counts and no `retention.sweep_run` row is written

### Requirement: Retention cron resumes incomplete anonymizations
The `data-retention` cron SHALL scan contacts with `is_anonymized = true` through `varrerRedacoesIncompletas`, redact leftover lead titles and activities with the same `lib/lgpd/cascata.ts` functions used by `POST /api/v1/lgpd/anonymize`, and audit each completed contact as `lgpd.anonymize_catchup` with `metadata.origem = 'cron.data-retention'`.

#### Scenario: Residue found
- **WHEN** an anonymized contact still has an unredacted lead title
- **THEN** the title is redacted and the response counts it in `anonimizacoes_completadas`

### Requirement: Per-organization media retention
`GET|POST /api/v1/cron/media-retention` SHALL authorize with `autorizaCron` and call `fn_enfileirar_midia_vencida(p_limite => 500)` up to 10 times per run; the function SHALL only expire media of organizations with `organizations.media_retention_enforced = true`, use `greatest(coalesce(media_retention_days,365),30)` days, null `messages.media_storage_path` and `media_url`, set `metadata.media_status = 'expired'`, and skip organizations with a `lgpd_requests` row in `received` or `processing`.

#### Scenario: Retention configured below the floor
- **WHEN** an organization stores `media_retention_days = 5`
- **THEN** only media older than 30 days is expired

#### Scenario: LGPD request in progress
- **WHEN** the organization has a `lgpd_requests` row in `processing`
- **THEN** none of its media is enqueued in that run

#### Scenario: Function error
- **WHEN** `fn_enfileirar_midia_vencida` fails
- **THEN** the response is 500 `media_retention_failed`

### Requirement: Webhook archive pruning
`GET /api/v1/cron/webhook-log-retention` SHALL authorize with `autorizaCron`, accept `lote` (default 500, max 5000), first clear `raw_body`, `payload_parsed` and `headers` and stamp `archived_at` on `webhook_events_log` rows older than `WEBHOOK_LOG_BODY_RETENTION_DAYS` (default 7), delete rows older than `WEBHOOK_LOG_ROW_RETENTION_DAYS` (default 90), and then prune `webhook_lead_captures` by `LEAD_CAPTURE_RETENTION_DAYS`.

#### Scenario: Body archived
- **WHEN** a `webhook_events_log` row is 8 days old and not archived
- **THEN** its `raw_body` becomes null and `archived_at` is set, and it is not selected again on the next run

#### Scenario: Lead capture purge fails
- **WHEN** the `webhook_lead_captures` delete fails
- **THEN** the response is 500 `webhook_lead_captures_retention_failed` with the archive result in `details`

#### Scenario: Archive purge fails
- **WHEN** the `webhook_events_log` step fails but the capture purge succeeds
- **THEN** the capture purge still runs and the response is 500 `webhook_events_log_retention_failed`
