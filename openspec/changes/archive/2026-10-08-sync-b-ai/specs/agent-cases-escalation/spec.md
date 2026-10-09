## ADDED Requirements

### Requirement: Case chat availability uses the same resolver as the answer
`GET /api/v1/ai/cases/{id}/chat` SHALL compute `ia_configurada` by calling `resolveOrgLlmConfig` with the case agent's provider and credential when the persona is the case agent (no override for the default persona), returning `false` only on `LlmNotConfiguredError` and `null` (logged) on any other failure, without hiding the rest of the case state.

#### Scenario: Organization credential but no agent on the case
- **WHEN** a case without agent belongs to an organization with a validated credential and no environment key
- **THEN** `ia_configurada` is `true`

#### Scenario: Credential cannot be decrypted
- **WHEN** resolving the credential throws an error other than `LlmNotConfiguredError`
- **THEN** `ia_configurada` is `null` and the response still carries the persona and case status
