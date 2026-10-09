## ADDED Requirements

### Requirement: Agent jobs record queue wait and wall time
`recordRunMetrics` SHALL insert into `metrics`, for every finished agent job with `claim_acquired_at`, `run_queue_wait_ms` (claim time minus `created_at`, never negative) and `run_wall_ms` (database clock from the claim to the close), even when the job made no LLM call, besides the token and cost metrics written only when it made calls.

#### Scenario: Job with no model call
- **WHEN** an agent job finishes without any `llm_calls` row
- **THEN** `metrics` still receives `run_queue_wait_ms` and `run_wall_ms` for it
