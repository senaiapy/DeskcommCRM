## ADDED Requirements

### Requirement: Toggling a closed task reopens it as pending
The task list toggle (`alternarConcluida` in `hooks/tasks/useTasks.ts`) SHALL send `PATCH /api/v1/tasks/[id]` with the status from `proximoEstadoAoAlternar()` in `lib/tarefas/tipos.ts`, which returns `pending` for a task in `done` or `cancelled` and `done` otherwise.

#### Scenario: Reopening a cancelled task
- **WHEN** a user toggles a task whose `status = 'cancelled'`
- **THEN** the task is saved with `status = 'pending'` and no `task_completed` activity is written

#### Scenario: Completing a pending task
- **WHEN** a user toggles a task whose `status = 'pending'`
- **THEN** the task is saved with `status = 'done'`
