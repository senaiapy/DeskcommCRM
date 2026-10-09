## ADDED Requirements

### Requirement: Model text is normalized to WhatsApp formatting before the gates
The `send_message` tool of the agent turn SHALL pass the model's body through `formatarParaWhatsApp` (`lib/agent-engine/agent/formato-whatsapp.ts`) before any `before_send` gate runs, turning escaped `\n`/`\r\n` sequences into real line breaks, Markdown headings and `**text**`/`__text__` into WhatsApp `*text*`, collapsing three or more line breaks into two and trimming the result.

#### Scenario: Escaped line break and Markdown bold
- **WHEN** the model calls `send_message` with the body `**Oi**\n\nTudo bem?` written with literal backslash-n characters
- **THEN** the candidate evaluated by the gates and sent to the customer is `*Oi*` followed by a blank line and `Tudo bem?`

### Requirement: Auxiliary classifiers run only on a new customer message
`classificadoresDoTurno` SHALL enable the stage classifier (purpose `stage_classifier`) only when the stage knob is on, the job kind is `inbound_turn` or the turn is a preview, and the conversational agent has `update_lead_state`, and SHALL enable the manipulation classifier only when the organization's anti-manipulation layer is on, the job kind is `inbound_turn` or preview, and the last customer message is not blank.

#### Scenario: Follow-up turn
- **WHEN** a `followup_turn` or `case_reply_turn` job runs for a contact
- **THEN** no stage-classifier or manipulation-classifier model call is made for that turn

### Requirement: The closing checkpoint re-reads the contact's agenda
After a non-preview turn whose tool results include any of `crm_list_appointments`, `crm_find_free_slots`, `crm_book_appointment`, `crm_find_and_book_appointment`, `crm_reschedule_appointment`, `crm_cancel_appointment`, `crm_confirm_appointment` or `crm_set_appointment_outcome` (or whose opening carried a commitments block), the runtime SHALL rebuild the contact's commitments with `buildCompromissosBlock` and pass it to the `checkpoint` model call under the heading `## Agenda verificada depois das ações deste turno`, and a failed re-read SHALL be logged and replaced by a fixed caution text instead of failing the job.

#### Scenario: Appointment booked during the turn
- **WHEN** the agent books an appointment with `crm_book_appointment` and replies
- **THEN** the `checkpoint` call receives the re-read agenda block listing the new appointment

#### Scenario: Re-read fails
- **WHEN** the agenda query throws after the reply was sent
- **THEN** the job is not failed and the checkpoint receives the text saying the agenda could not be re-read

### Requirement: A test run with media that failed to prepare is not approved
`POST /api/v1/ai/agents/{id}/versions/{vid}/test` SHALL answer `status: 'blocked'` whenever any item of the dry-run `midia` list carries a `falha`, even when text candidates exist, and SHALL pass that list to `avaliarRespostaDeTeste` for the `guardrails` field.

#### Scenario: Product photo could not be prepared
- **WHEN** the dry run produces a text candidate but the product photo preparation failed
- **THEN** the response has `status: 'blocked'` and the `ai_agent_runs` row still ends `completed`

### Requirement: Publishing a custom or subscription model checks the credential's own model list
`fn_publish_ai_agent_version` (migration `20261007205134_0592_publicar_com_login_por_assinatura.sql`) SHALL, for a version whose `provider` is `custom` or `openai-assinatura`, accept the `model` only when it is listed in `ai_provider_credentials.models_available` of the version's credential in the same organization, raising `model_not_found` otherwise, while native providers keep being checked against `ai_models`.

#### Scenario: Subscription model not offered to this account
- **WHEN** an admin publishes an `openai-assinatura` version whose model is absent from the organization credential's `models_available`
- **THEN** the publish route answers 422 `model_not_found` and `published_version_id` does not move
