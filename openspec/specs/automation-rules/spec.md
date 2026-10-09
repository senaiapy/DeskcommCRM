# automation-rules Specification

## Purpose
Automation rules are per-organization "when event X, if conditions, do actions" definitions executed asynchronously from `event_log`. Key paths: UI `app/app/webhooks` (tab RulesTab/RuleEditor/ActivityTab) and the org kill-switch page `app/app/settings/automacoes` (ai_decide toggle); API `app/api/v1/automation-rules` (GET/POST), `automation-rules/[id]` (PATCH/DELETE), `automation-rules/[id]/runs` (GET), `automation-rules/runs` (GET), `automation-rules/runs/[runId]/resend` (POST); libs `lib/automation/engine.ts` + `engine.handler.ts` (event_log consumer `automation-rules`), `lib/automation/conditions.ts`, `lib/automation/actions/*`, `lib/schemas/webhooks.ts` (TRIGGER_EVENTS, action schemas); tables `automation_rules`, `automation_rule_runs`; crons `app/api/v1/cron/event-log-drain` (every minute), `lead-date-field-due` and `lead-time-triggers` (hourly, emit time-based trigger events).

## Requirements

### Requirement: Manager-only rule management
Every handler under `/api/v1/automation-rules` SHALL call `requireRole("manager")`, and POST, PATCH, DELETE and the resend POST SHALL call `requireSupportWrite()` first, with RLS policies `automation_rules_select` / `automation_rules_manager_write` restricting rows to `fn_user_org_ids()` and manager rank.

#### Scenario: Create a rule
- **WHEN** a manager calls `POST /api/v1/automation-rules` with a valid name, `trigger_event`, conditions and 1–10 actions
- **THEN** the response is 201 with the created row, whose `organization_id` is the caller's active organization and `is_active` defaults to `false`

#### Scenario: Delete a rule
- **WHEN** a manager calls `DELETE /api/v1/automation-rules/{id}` for an existing rule of the organization
- **THEN** the response is 204 and an audit `automation.rule_deleted` is written

#### Scenario: Unknown rule
- **WHEN** PATCH or DELETE targets an id not visible to the caller
- **THEN** the response is 404 with code `not_found`

### Requirement: Closed trigger vocabulary
`trigger_event` SHALL be one of `TRIGGER_EVENTS` from `lib/schemas/webhooks.ts` (e.g. `lead.created`, `lead.stage_changed`, `lead.won`, `lead.lost`, `lead.reopened`, `lead.assigned`, `message.received`, `message.failed`, `lead.tag_added`, `contact.tag_added`, `contact.birthday`, `appointment.*`, `lead.date_field_due`, `lead.silent_for`, `lead.stage_stale`), and an invalid body SHALL return 400 `invalid_request`.

#### Scenario: Time trigger without configuration
- **WHEN** a rule is created with `trigger_event = 'lead.silent_for'` and no valid `trigger_config`
- **THEN** the response is 400 with code `invalid_request` and no row is inserted

### Requirement: Lead-loop prevention
POST and PATCH SHALL reject (400 `invalid_request`) a rule whose trigger and actions would re-trigger themselves on the same lead (`acoesQueFechamLaco`), including a PATCH that changes only the trigger or only the actions.

#### Scenario: Partial PATCH that closes the loop
- **WHEN** a PATCH changes only `actions` so that, combined with the stored `trigger_event`, an action rewrites the lead that fires the trigger
- **THEN** the response is 400 with code `invalid_request` and the rule is unchanged

### Requirement: Webhook secrets encrypted at rest
On POST and PATCH, any `call_webhook` action `config.secret` SHALL be replaced by `config.secret_enc` via `encryptRuleActionSecrets` before writing `automation_rules.actions`, and when encryption is unavailable the request SHALL fail with 422 `encryption_unavailable`.

#### Scenario: Encryption key missing
- **WHEN** a manager saves a rule with a `call_webhook` secret on an installation without the encryption key
- **THEN** the response is 422 with code `encryption_unavailable` and nothing is written

### Requirement: Asynchronous execution via event_log
Rules SHALL execute only through the `automation-rules` event_log handler (registered in `lib/event-log/register-handlers.ts`, drained by `cron/event-log-drain`), which loads active rules of the event's `organization_id` with matching `trigger_event`, evaluates `conditions` (ops `eq`, `neq`, `contains`) against the built context, and runs each action in order.

#### Scenario: Conditions not met
- **WHEN** an event matches an active rule's trigger but its conditions evaluate false
- **THEN** no action runs and no `automation_rule_runs` row is written (handler detail `no_match`)

### Requirement: Execution guards
The engine SHALL skip an event when `metadata.caused_by_rule` is set or `metadata.request_id` starts with `rule:` (anti-loop depth 1), when the `entity_kind` does not match the trigger's expected entity, when the contact is personal (`is_personal`), or when the event's channel is disabled.

#### Scenario: Event caused by a rule
- **WHEN** the drained event carries `metadata.caused_by_rule = true`
- **THEN** the handler returns `skipped` with detail `caused_by_rule` and no rule runs

### Requirement: Run log and statuses
Each rule execution SHALL insert an `automation_rule_runs` row with `status` in (`success`, `partial`, `failed`, `adiado`) and per-action `actions_result`, `failed` when every action failed or was skipped, `partial` when some did, `adiado` when an action is postponed, and SHALL update `automation_rules.last_run_at` and `run_count`.

#### Scenario: One of two actions fails
- **WHEN** a rule with two actions runs and exactly one action returns `failed`
- **THEN** the inserted run has `status = 'partial'` and an audit row is written for the non-success run

### Requirement: Postponement until send window
When an action's `postponeUntil` returns a time (e.g. outside the channel send window), the engine SHALL execute no action of the applicable rules, insert a run with `status = 'adiado'` and an action result `postponed`, and return `retry` with `retry_at` so the drain reschedules the event.

#### Scenario: WhatsApp action outside the window
- **WHEN** a rule with `send_whatsapp_message` matches an event while the channel window is closed
- **THEN** an `automation_rule_runs` row with `status = 'adiado'` is written and the event is rescheduled without running any action

### Requirement: Run history endpoints
`GET /api/v1/automation-rules/runs` and `GET /api/v1/automation-rules/{id}/runs` SHALL return `automation_rule_runs` of the caller's organization ordered by `created_at` descending, with a capped `limit`.

#### Scenario: Recent activity
- **WHEN** a manager calls `GET /api/v1/automation-rules/runs?limit=20`
- **THEN** the response is 200 with at most 20 runs of the active organization, newest first, each with `automation_rules.name`

### Requirement: Resend webhook actions of a run
`POST /api/v1/automation-rules/runs/{runId}/resend` SHALL re-execute only the rule's current `call_webhook` actions against the original `event_log` row, insert a new run (201), return 409 `event_gone` when the event no longer exists, and 409 `no_actions_to_resend` when the rule has no `call_webhook` action left.

#### Scenario: Rule lost its webhook action
- **WHEN** resend is requested for a run whose rule no longer contains any `call_webhook` action
- **THEN** the response is 409 with code `no_actions_to_resend` and no run is inserted

#### Scenario: Original event pruned
- **WHEN** resend is requested for a run whose `event_id` is null or missing from `event_log`
- **THEN** the response is 409 with code `event_gone`

### Requirement: Lead-created rules can be scoped to one webhook source
`POST /api/v1/automation-rules` and `PATCH /api/v1/automation-rules/{id}` SHALL accept `trigger_config.webhook_source_id` only as a UUID (400 `invalid_request` otherwise) of a `webhook_sources` row of the active organization (422 `invalid_request` "A fonte escolhida não pertence a esta empresa." otherwise), and the engine SHALL skip a `lead.created` rule with that key when the lead's `source_metadata.webhook_source_id` differs.

#### Scenario: Rule limited to one form
- **WHEN** a rule on `lead.created` has `trigger_config.webhook_source_id = A` and a lead arrives through webhook source B
- **THEN** the rule does not run for that lead

#### Scenario: Source from another organization
- **WHEN** a manager saves a rule whose `webhook_source_id` belongs to another organization
- **THEN** the response is 422 and the rule is not saved

### Requirement: Saving from the screen keeps configuration the screen does not edit
For date, time and new-contact triggers, the rule editor (`app/app/webhooks/_components/RuleEditor.tsx`) SHALL build the saved `trigger_config` with `configAoSalvarDaTela()` (`lib/automation/config-ao-salvar.ts`), merging the edited keys over the stored `trigger_config` when the trigger is unchanged and starting from `{}` when the trigger changed.

#### Scenario: Pipeline filter survives a screen save
- **WHEN** a `lead.silent_for` rule stored with `trigger_config = { dias: 3, pipeline_id: P, stage_id: S }` is edited and saved from the screen without changing its trigger
- **THEN** the saved `trigger_config` still has `pipeline_id = P` and `stage_id = S`

#### Scenario: Trigger changed on the screen
- **WHEN** the same rule is saved with its trigger changed to `lead.stage_stale`
- **THEN** the saved `trigger_config` holds only the keys the screen edits
