## ADDED Requirements

### Requirement: A move to a loss stage needs a loss reason at publish
`validateFlowForPublish` SHALL reject a `move_lead` node whose target stage has `is_lost = true` with error code `motivo_da_perda_ausente` when `config.lost_reason` is blank and `motivo_da_perda_invalido` when the reason is outside the pipeline's loss-reason vocabulary, so `POST /api/v1/ai/followup-flows/{id}/publish` answers 422 `validation_failed` with those codes in `details.errors`; the engine SHALL pass `lost_reason` to `moveLeadHandler` when the node has one.

#### Scenario: Loss stage without reason
- **WHEN** a manager publishes a flow whose `move_lead` node targets the pipeline's loss stage and has no `lost_reason`
- **THEN** the response is 422 `validation_failed` and `details.errors` contains `motivo_da_perda_ausente`

### Requirement: A failed lead move is recorded on the enrollment timeline
When `moveLeadHandler` throws while the engine executes a `move_lead` node, the engine SHALL insert a `followup_enrollment_events` row with `event_type = 'move_lead_failed'`, `payload.error` and `payload.codigo` (the `ApiError` code, e.g. `lost_reason_required`) and idempotency key `<step key>:falha`, logging instead of throwing when that insert itself fails.

#### Scenario: Legacy flow moves to the loss stage without reason
- **WHEN** an already published flow moves an open lead to the loss stage without `lost_reason`
- **THEN** the lead stays where it was and the enrollment timeline shows a `move_lead_failed` event

### Requirement: Appointment confirmation failures do not stop follow-ups
`GET|POST /api/v1/cron/followup-flow-worker` SHALL run `fn_appointment_confirmation_sweep` inside an isolated guard so that an RPC error or exception is logged with `logger.error`, sent to Sentry when configured and audited as `agenda.confirmation_sweep_run` with `metadata.falhou = true`, while the follow-up tick still runs, and SHALL return the outcome in `confirmation_sweep` (`{ok: true, avisos}` or `{ok: false, erro}`).

#### Scenario: Confirmation sweep RPC fails
- **WHEN** `fn_appointment_confirmation_sweep` returns an error
- **THEN** the cron answers 200 with the tick summary and `confirmation_sweep.ok = false`, and due enrollments still advanced
