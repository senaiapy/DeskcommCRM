## 1. Re-verify the compliance capabilities against HEAD

- [x] 1.1 `lgpd-privacy`: 1 added; the existing export requirement stays true (bucket `lgpd-exports`, `data.json` + `report.pdf`, hashed recipient). Evidence: workers/lgpd-export-worker.ts:193, workers/lgpd-export-worker.ts:236, lib/lgpd/copia-do-titular.ts:205, lib/lgpd/copia-do-titular.ts:310, lib/lgpd/email-delivery.ts:140
- [x] 1.2 `data-retention`: 1 added. Evidence: lib/retencao/politica.ts:309, app/api/v1/cron/data-retention/route.ts:543, supabase/migrations/20261007131344_0587_retencao_das_tabelas_da_ia.sql:134, supabase/migrations/20261007131344_0587_retencao_das_tabelas_da_ia.sql:117, supabase/migrations/20261007131344_0587_retencao_das_tabelas_da_ia.sql:264, .env.example:565, supabase/migrations/20261007131344_0587_retencao_das_tabelas_da_ia.sql:172

## 2. Validate

- [x] 2.1 `openspec validate sync-b-compliance --strict --no-interactive` passes
