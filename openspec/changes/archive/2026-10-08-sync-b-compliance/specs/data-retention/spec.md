## ADDED Requirements

### Requirement: AI append-only tables are pruned by the daily cron
The `data-retention` cron SHALL also drain, with the same batches of 1000 and at most 20 batches, `fn_expurgar_telemetria_de_ia_vencida` (`llm_calls`, `metrics`, `skill_activations`, `ai_router_decisions`; knob `AI_TELEMETRY_RETENTION_DAYS`, default 400, floor 100), `fn_expurgar_ritmo_de_envio_vencido` (`pacing_ledger`, keeping the last row per number; `PACING_LEDGER_RETENTION_DAYS`, 2/2), `fn_expurgar_copias_enviadas_vencidas` (`outbound_copies`; `OUTBOUND_COPIES_RETENTION_DAYS`, 30/7) and `fn_expurgar_checkpoints_superados` (`lead_checkpoints`, never the latest of a frontier nor one of a live job; `LEAD_CHECKPOINT_RETENTION_DAYS`, 180/30), all `security definer` functions of migration `20261007131344_0587_retencao_das_tabelas_da_ia.sql` with the floor in the body, executable only by `service_role`, and SHALL report `*_apagada(s)`, `lotes_*`, `*_tem_resto` and `retencao_*_dias` for each.

#### Scenario: Telemetry knob below the floor
- **WHEN** `AI_TELEMETRY_RETENTION_DAYS=30`
- **THEN** the cron purges `llm_calls` older than 100 days and logs a warning naming the key and the floor

#### Scenario: Legacy backfilled rows
- **WHEN** an `llm_calls` row older than the retention has `legacy_invocation_id` set
- **THEN** `fn_expurgar_telemetria_de_ia_vencida` does not delete it
