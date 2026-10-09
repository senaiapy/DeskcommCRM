## 1. Re-verify each crm-core capability against HEAD (spec-only sync)

- [x] 1.1 `contacts` (2 modified: CPF pair written together or not at all, with `encrypt_cpf`/`decrypt_cpf` from migration 0597; duplicate scan paged). Evidence: `lib/contacts/cpf.ts:66`, `app/api/v1/contacts/_handler.ts:493`, `app/api/v1/contacts/_handler.ts:666`, `app/api/v1/contacts/import/route.ts:255`, `supabase/migrations/20261008021533_0597_cpf_cifra_at_rest.sql:50`, `supabase/migrations/20261008021533_0597_cpf_cifra_at_rest.sql:63`, `supabase/migrations/20261008021533_0597_cpf_cifra_at_rest.sql:105`, `app/api/v1/contacts/duplicates/route.ts:53`, `app/api/v1/contacts/duplicates/route.ts:104`.
- [x] 1.2 `leads-deals` (2 added). Evidence: `app/api/v1/leads/route.ts:86`, `app/api/v1/leads/_handler.ts:271`, `app/api/v1/leads/_handler.ts:695`.
- [x] 1.3 `pipelines-kanban` (1 modified, 1 added). Evidence: `lib/pipelines/pipeline-editing.ts:201`, `app/api/v1/pipelines/[id]/route.ts:326`, `components/kanban/KanbanBoard.tsx:238`, `lib/kanban/vizinho-na-etapa.ts:31`.
- [x] 1.4 `tasks-activities` (1 added). Evidence: `lib/tarefas/tipos.ts:82`, `hooks/tasks/useTasks.ts:113`.
- [x] 1.5 Unchanged since `477b77678` for requirement purposes: `companies-people`, `tags`, `proposals`, `products-catalog`, `finance`, `agenda-scheduling`, `imports`. Evidence: `git diff --name-only 477b77678 HEAD -- lib/crm-b2b lib/propostas lib/catalogo lib/agenda/consulta.ts app/api/v1/products app/api/v1/proposals app/api/v1/financeiro app/api/v1/imports app/api/v1/tags app/api/v1/agenda` is empty; the contact CSV import's only change is the CPF pair covered in 1.1; the `currency` custom-field input (`components/kanban/CamposObrigatoriosDialog.tsx`, `components/contacts/CustomFieldsEditor.tsx`) is UI formatting only.
- [x] 1.6 `leads-deals` (MCP): 1 added — `crm_move_lead_stage` forwards `lost_reason`. Evidence: lib/mcp/tools/leads.ts:450, lib/mcp/tools/leads.ts:454, lib/mcp/tools/leads.ts:483
- [x] 1.7 `agenda-scheduling` (MCP): 1 added — tools return `quando`/`fim_quando`. Evidence: lib/mcp/tools/agendamento.ts:31, lib/mcp/tools/agendamento.ts:523, lib/mcp/tools/agendamento.ts:644

## 2. Validate

- [x] 2.1 `openspec validate sync-a-crm-core --strict --no-interactive`. Evidence: output `Change 'sync-a-crm-core' is valid`.
