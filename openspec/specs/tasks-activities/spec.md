# tasks-activities Specification

## Purpose
Tasks and activities cover internal work reminders and the lead timeline. `crm_tasks` (migration 0210) holds internal to-dos with optional deadline (`due_date`), `priority`, `status`, and optional links to `lead_id` / `contact_id` / `assigned_to`; task plans are ordered task sequences stored in `organizations.settings.task_plans` and applied to a lead by the automation action `apply_task_plan`. `crm_lead_activities` is the polymorphic, append-only lead timeline (`type`, `source_module`/`source_id`, `actor_kind`, `reason`, `evidence`, `payload`, `performed_at`) whose `type` vocabulary lives only in TypeScript (`lib/leads/activity-vocabulary.ts`) and is written through `emitLeadActivity` (`lib/leads/activity-emitter.ts`). Entry points: `GET`/`POST /api/v1/tasks`, `PATCH`/`DELETE /api/v1/tasks/[id]`, `GET`/`PATCH /api/v1/settings/task-plans`, `GET /api/v1/leads/[id]/timeline`, `GET /api/v1/reports/activities`; libraries `lib/tarefas/*` (`tipos.ts`, `atividade.ts`, `criar-tarefa.ts`, `plano.ts`), `lib/automation/actions/{create-task,apply-task-plan}.ts`; pages `app/app/tasks` (list + calendar), `app/app/tasks/planos` (plan editor), `app/app/activities` (activity and tag report). There is no due/overdue reminder cron for tasks: overdue is computed on read (`estaAtrasada`).

## Requirements

### Requirement: Task table shape and constraints
`crm_tasks` SHALL carry `organization_id` (cascade), a non-blank `title` (`crm_tasks_titulo_nao_vazio`), nullable `due_date`, `priority` in `low|medium|high|urgent` (default `medium`), `status` in `pending|in_progress|done|cancelled` (default `pending`), and `lead_id` / `contact_id` / `assigned_to` / `created_by` foreign keys with `ON DELETE SET NULL`.

#### Scenario: Lead deleted
- **WHEN** a `crm_leads` row referenced by a task is deleted
- **THEN** the task remains with `lead_id = NULL`

#### Scenario: Invalid priority at the database
- **WHEN** a row is inserted with `priority = 'critical'`
- **THEN** the insert fails on `crm_tasks_priority_check`

### Requirement: Task RLS is org-scoped with agent writes
RLS on `crm_tasks` SHALL allow SELECT to members of the organization (`fn_user_org_ids()`) or platform admins, and INSERT/UPDATE/DELETE only to members with `fn_role_at_least(organization_id, 'agent')` or a full platform admin, with all privileges revoked from `anon`.

#### Scenario: Viewer writes a task directly
- **WHEN** a `viewer` inserts into `crm_tasks` through PostgREST with the session client
- **THEN** the `crm_tasks_write` policy rejects the row

### Requirement: Listing tasks
`GET /api/v1/tasks` SHALL require role `viewer` or higher, filter by the active `organization_id`, accept optional `status`, `priority`, `lead_id`, `contact_id`, `due_from`, `due_to` and `aberto=true` (status `pending` or `in_progress`), order by `due_date` ascending with nulls last then `created_at` descending, cap at 500 rows, and answer 422 `validation_failed` on invalid query parameters.

#### Scenario: Open tasks only
- **WHEN** a user calls `GET /api/v1/tasks?aberto=true`
- **THEN** the response `data.tasks` contains only tasks with status `pending` or `in_progress` of the active organization

#### Scenario: Bad filter
- **WHEN** the query carries `status=late`
- **THEN** the response is 422 `validation_failed`

### Requirement: Creating and editing tasks
`POST /api/v1/tasks` and `PATCH /api/v1/tasks/[id]` SHALL call `requireSupportWrite()`, require role `agent` or higher, validate the body with Zod (title 1–255, ISO `due_date` with offset), set `organization_id` and `created_by` from the session, write an audit (`crm_task.created` / `crm_task.updated`), and answer 422 `validation_failed` when the body is invalid, the PATCH body is empty, or the linked lead/contact violates its foreign key (23503).

#### Scenario: Task created
- **WHEN** an agent posts `{ "title": "Ligar de volta", "due_date": "2026-10-10T13:00:00-03:00" }`
- **THEN** the response is 201 with `data.task` having `status = 'pending'`, `priority = 'medium'` and `created_by` equal to the caller

#### Scenario: Empty patch
- **WHEN** `PATCH /api/v1/tasks/[id]` is sent with `{}`
- **THEN** the response is 422 `validation_failed` and `updated_at` is unchanged

### Requirement: Deleting a task reports what happened
`DELETE /api/v1/tasks/[id]` SHALL require role `agent` or higher, delete only within the active `organization_id`, return 404 `not_found` when no row was deleted, and write an audit `crm_task.deleted` otherwise.

#### Scenario: Task of another organization
- **WHEN** an agent deletes a task id that belongs to a different organization
- **THEN** the response is 404 `not_found` and no row is removed

### Requirement: Tasks tied to a lead appear on its timeline
Creating a task with a `lead_id` SHALL write a `task_created` row to `crm_lead_activities` (`source_module = 'tarefas'`, `source_id` = task id), and a PATCH that moves a task to `status = 'done'` from another status SHALL write `task_completed`, while tasks without `lead_id` write no activity.

#### Scenario: Completing a lead task
- **WHEN** a task with `lead_id` and `status = 'pending'` is patched to `status = 'done'`
- **THEN** one `task_completed` activity exists for that lead with `source_id` equal to the task id

#### Scenario: Re-saving a done task
- **WHEN** a task already `done` is patched again with `status = 'done'`
- **THEN** no new `task_completed` activity is written

### Requirement: Task plans are stored in organization settings
`GET /api/v1/settings/task-plans` (viewer+) SHALL return only the valid plans parsed from `organizations.settings.task_plans`, and `PATCH /api/v1/settings/task-plans` SHALL call `requireSupportWrite()`, require role `manager`, validate `planos` with `planosSchema` (max 50 plans, 1–30 steps, `vence_em_dias` 0–365, `atribuir_a` `dono_do_lead` or `{ usuario_id }`) answering 422 `validation_failed`, and merge the list into `settings` without dropping other keys.

#### Scenario: Agent saves plans
- **WHEN** an `agent` sends `PATCH /api/v1/settings/task-plans`
- **THEN** the `requireRole` denial is returned and `organizations.settings` is unchanged

#### Scenario: Plan saved preserving other settings
- **WHEN** a manager saves a valid plan list
- **THEN** `organizations.settings.task_plans` equals the list and other keys of `settings` are preserved

### Requirement: Applying a task plan is idempotent per lead
`aplicarPlanoDeTarefas` (used by the automation action `apply_task_plan`) SHALL create the plan's tasks in declared order with `due_date` = now + `vence_em_dias`, validate every step against the lead before the first insert, record a `task_plan_applied` activity on the lead, and return `ja_aplicado = true` without writing when that plan was already applied to the same lead.

#### Scenario: Rule fires twice
- **WHEN** an automation applies plan `onboarding` to the same lead twice
- **THEN** the tasks exist once and the second call returns `ja_aplicado = true`

#### Scenario: Plan without lead
- **WHEN** the plan is applied without a lead
- **THEN** the result is `{ ok: false, codigo: "sem_alvo" }` and no task is inserted

### Requirement: Activity rows are typed in TypeScript and evidenced for AI
`crm_lead_activities.type` SHALL have no CHECK constraint, with emitters using the `ActivityType` union in `lib/leads/activity-vocabulary.ts`, while `actor_kind` SHALL be one of `user|ai|system|rule|contact` and `crm_lead_activities_ai_needs_evidence` SHALL require `run_ids`, `trace_ids` or `llm_call_ids` in `evidence` when `actor_kind = 'ai'` (`emitLeadActivity` downgrades an unevidenced AI actor to `system`).

#### Scenario: AI activity without evidence
- **WHEN** `emitLeadActivity` is called with an AI actor and no evidence ids
- **THEN** the inserted row has `actor_kind = 'system'` and `actor_agent_id` set

### Requirement: Only real touches refresh last_activity_at
The trigger `trg_update_last_activity_at` (AFTER INSERT on `crm_lead_activities`) SHALL update `crm_leads.last_activity_at` and `contacts.last_activity_at` only for types in its positive list (`ai_turn`, `note`, `lead_edited`, `stage_changed`, `next_action_approved`, `voice_call`).

#### Scenario: Cooling activity
- **WHEN** a `lead_cooled` activity is inserted for a lead
- **THEN** `crm_leads.last_activity_at` of that lead is unchanged

#### Scenario: Stage change
- **WHEN** a `stage_changed` activity is inserted
- **THEN** `crm_leads.last_activity_at` becomes at least the activity's `performed_at`

### Requirement: Lead timeline is paginated and visibility-scoped
`GET /api/v1/leads/[id]/timeline` SHALL authenticate with `getUser()` (401 `unauthenticated`), resolve the lead through RLS (404 `not_found` when invisible), return activities ordered by `performed_at` desc then `id` desc with `limit` clamped to 1–100 (default 50), optional repeated `type` filters, and `meta.has_more` / `meta.cursor`, answering 400 `invalid_cursor` for an undecodable cursor.

#### Scenario: Second page
- **WHEN** a client calls the timeline with the `meta.cursor` of the previous page
- **THEN** only activities older than the last row of that page are returned

#### Scenario: Forged cursor
- **WHEN** `cursor=abc` is passed
- **THEN** the response is 400 `invalid_cursor`

### Requirement: Period activity report respects lead visibility
`GET /api/v1/reports/activities` SHALL require role `viewer` or higher, accept `days` (1 to the route maximum, default 7) and answer 422 `validation_failed` otherwise, and call `fn_activity_report` with the session client so the `crm_lead_activities_select` policy restricts an `agent` to activities of leads they can view.

#### Scenario: Agent in own mode
- **WHEN** an `agent` whose organization uses `visibility_mode = 'own'` requests the report
- **THEN** the counts include only activities of leads owned by that agent

### Requirement: Toggling a closed task reopens it as pending
The task list toggle (`alternarConcluida` in `hooks/tasks/useTasks.ts`) SHALL send `PATCH /api/v1/tasks/[id]` with the status from `proximoEstadoAoAlternar()` in `lib/tarefas/tipos.ts`, which returns `pending` for a task in `done` or `cancelled` and `done` otherwise.

#### Scenario: Reopening a cancelled task
- **WHEN** a user toggles a task whose `status = 'cancelled'`
- **THEN** the task is saved with `status = 'pending'` and no `task_completed` activity is written

#### Scenario: Completing a pending task
- **WHEN** a user toggles a task whose `status = 'pending'`
- **THEN** the task is saved with `status = 'done'`
