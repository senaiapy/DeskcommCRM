## ADDED Requirements

### Requirement: A scheduled round judges only the latest unjudged turn per contact
`collectRecentTurns` SHALL select, per `contact_id`, only the most recent `job_queue` row of kind `inbound_turn` with status `done` that has an `agent_turn` `llm_calls` row, drop the contact when that turn already has a `flywheel_judge_verdicts` row for the same `dataset` and `dimension`, and, in the scheduled loop, consider only jobs created within twice `FLYWHEEL_INTERVAL_MS` (`janelaMs`), so no judge call is paid twice for the same turn.

#### Scenario: Latest turn already judged
- **WHEN** a contact's most recent done turn already has a `live`/`memory_hygiene` verdict
- **THEN** the round makes no `flywheel_judge` call for that contact, not even for an older turn of the same contact

#### Scenario: Old history
- **WHEN** the scheduled round runs with `FLYWHEEL_INTERVAL_MS=21600000`
- **THEN** jobs created more than 12 hours ago are not collected
