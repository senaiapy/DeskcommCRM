## ADDED Requirements

### Requirement: Media model calls are metered and pass the spend ceiling
Before sending an image to the vision model or audio to a conversation model, the derive worker SHALL run `aplicarOrcamento` with purpose `visao_de_imagem` or `transcricao_de_audio`, and when it raises `LlmBudgetExceededError` SHALL not contact the provider, return the unreadable marker and open the `midia_nao_lida` notice explaining the ceiling; every vision or transcription call that left the server, successful or failed, SHALL be written to `llm_calls` with that purpose, its model and its computed cost (null for service transcription, whose origin is in `ORIGENS_DA_TRANSCRICAO_POR_SERVICO`).

#### Scenario: Ceiling reached in bloquear mode
- **WHEN** a customer sends a photo while the organization budget blocks AI calls
- **THEN** no vision call is made, the message gets the unreadable marker and a `midia_nao_lida` notice names the spend ceiling

#### Scenario: Image described
- **WHEN** an image is described successfully
- **THEN** one `llm_calls` row with purpose `visao_de_imagem`, `status = 'ok'` and the model's cost exists for the organization
