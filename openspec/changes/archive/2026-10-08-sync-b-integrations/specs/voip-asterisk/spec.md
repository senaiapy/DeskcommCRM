## ADDED Requirements

### Requirement: Each ended AI call leaves one usage row
When the AudioSocket bridge of an AI-handled call ends, the voice-agent worker SHALL, after finalizing `voice_calls`, write one `llm_calls` row through `registrarChamadaDeIa` with `purpose = 'voz_ao_vivo'`, the organization, agent and contact of the call, the Realtime model, the input and output tokens reported by the session, `latency_ms` equal to the call duration and `cost_cents = null`, never throwing when that insert fails.

#### Scenario: Call answered by the voice agent
- **WHEN** a SIP call handled by the AI ends after two minutes
- **THEN** one `llm_calls` row with `purpose = 'voz_ao_vivo'`, `latency_ms` about 120000 and null `cost_cents` exists for the organization

#### Scenario: Usage insert fails
- **WHEN** the `llm_calls` insert returns an error
- **THEN** the call row is still `status = 'ended'` and the failure is only logged
