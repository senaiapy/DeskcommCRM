## ADDED Requirements

### Requirement: Extra form fields configured per source
`webhook_sources.form_fields` (jsonb, default `[]`, migration 0595) SHALL hold up to 20 questions validated by `webhookFormFieldsSchema` (`key` matching `^[a-z][a-z0-9_]*$` up to 40 chars and not a reserved identity, consent or `utm_` key, unique per source; `label` 1–120 chars; `type` among `text`, `textarea`, `number`, `currency`, `select`, `checkbox`; `options` 1–30 only for `select`), accepted by `POST /api/v1/webhook-sources` and `PATCH /api/v1/webhook-sources/[id]`, and used by the source screen to generate its HTML form.

#### Scenario: Reserved key
- **WHEN** a manager saves a source with a form field whose `key` is `email`
- **THEN** the response is 400 `invalid_request` and `form_fields` is unchanged

#### Scenario: List without options
- **WHEN** a field has `type = "select"` and no `options`
- **THEN** the response is 400 `invalid_request`

### Requirement: Numeric and currency answers are validated at capture
`POST /api/v1/webhooks/in/[token]` SHALL normalize the payload values of the source's `number` and `currency` fields with `normalizarCamposNumericos()` (case-insensitive key; currency accepts BRL such as `R$ 1.250,00`), store them as JSON numbers in the lead's `custom_fields`, and answer 422 `invalid_request` with `details.fields` while recording the capture with `outcome = 'recusado'` and `reject_reason = 'campo_numerico_invalido'` when a value is not numeric.

#### Scenario: BRL value
- **WHEN** a capture posts `valor = "R$ 1.250,50"` for a `currency` field `valor`
- **THEN** the lead's `custom_fields.valor` is `1250.5`

#### Scenario: Letters in a number field
- **WHEN** a capture posts `idade = "trinta"` for a `number` field `idade`
- **THEN** the response is 422 with `details.fields = ["idade"]`, no lead is created, and the capture row has `reject_reason = 'campo_numerico_invalido'`
