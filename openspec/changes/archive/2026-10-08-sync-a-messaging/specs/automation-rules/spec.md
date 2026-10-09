## ADDED Requirements

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
