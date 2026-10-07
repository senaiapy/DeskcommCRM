# ai-credentials-budget Specification

## Purpose
Bring-your-own provider keys (`ai_provider_credentials`), per-purpose provider/model bindings (`ai_purpose_bindings`), the model catalog (`ai_models`) kept fresh by the `sync-model-catalog` cron, cost computation in fractional cents, the monthly spend ceiling (`ai_budgets`) and the usage report read from `llm_calls`. Agent publishing that consumes a credential is covered by `ai-agents-runtime`; the Jev classifier toggle that requires a validated key is covered by `ai-routers-skills-memory`.

## Requirements

### Requirement: API keys are stored only AES-256-GCM encrypted
`POST /api/v1/ai/credentials` SHALL require role `admin`, accept `provider` from `IDS_COM_CHAVE`, encrypt `api_key` with AES-256-GCM using env `AI_CRED_AES_KEY` (exactly 32 bytes base64) into `api_key_encrypted`, `api_key_iv` and `api_key_tag`, and return the row only from the view `ai_provider_credentials_safe`, which exposes `api_key_last4` but no ciphertext.

#### Scenario: Listing credentials
- **WHEN** a manager calls `GET /api/v1/ai/credentials`
- **THEN** the rows come from `ai_provider_credentials_safe` and no encrypted column crosses the HTTP boundary

#### Scenario: Duplicate label
- **WHEN** an admin creates a second key with the same provider and label
- **THEN** the response is 409 `label_already_used`

### Requirement: Only the custom provider accepts a base URL
The credential create route SHALL answer 422 `validation_failed` when `provider = 'custom'` has no `base_url`, when `base_url` does not start with `http://` or `https://`, or when a native provider sends a `base_url`.

#### Scenario: Native provider with endpoint
- **WHEN** an admin posts an OpenAI key with `base_url`
- **THEN** the response is 422 and nothing is stored

### Requirement: A credential referenced by any version cannot be deleted
`DELETE /api/v1/ai/credentials/{id}` SHALL require role `admin` and answer 409 `credential_in_use` with `details.count` when any `ai_agent_versions` row of the organization (draft, published or superseded) references the credential.

#### Scenario: Key used by a superseded version
- **WHEN** the only reference is a superseded version
- **THEN** the delete is still refused with 409 and `details.immutable_count` counts that version

### Requirement: Purpose bindings refuse incompatible choices at write time
`PUT /api/v1/ai/providers` SHALL require role `admin`, upsert `ai_purpose_bindings` on `(organization_id, purpose)`, answer 404 `ponto_desconhecido` for an unknown purpose, and answer 422 `credencial_invalida` or `credencial_de_outro_provedor` when `credential_id` is missing in the organization or belongs to another provider.

#### Scenario: Key of another provider
- **WHEN** an admin binds an Anthropic model to an OpenAI credential
- **THEN** the response is 422 `credencial_de_outro_provedor` and the binding is not written

### Requirement: The model catalog syncs from OpenRouter without deleting rows
`GET|POST /api/v1/cron/sync-model-catalog` SHALL authorize with `INTERNAL_CRON_SECRET` or `INTERNAL_SECRET` (401 `unauthorized` otherwise), upsert `ai_models` rows of `source = 'openrouter'` on `(provider, model_id)`, mark vanished rows with `deprecated_at` instead of deleting, never touch `source = 'manual'` rows, and abort the round with 200 `recusado: true` when the source returns fewer than 50% (`PISO_DE_SANIDADE`) of the known active models.

#### Scenario: Suspicious source response
- **WHEN** OpenRouter returns 100 models while 400 are known
- **THEN** no row is deprecated and the response is 200 with `recusado: true`

### Requirement: Cost is computed in fractional cents from pricing tables
`computeCost` SHALL price tokens from `ai_pricing` rows with `superseded_at is null`, fall back to `ai_models.input_price_per_million_cents`/`output_price_per_million_cents` when the model is not in `ai_pricing`, return unrounded cents, and return 0 when neither table prices the model.

#### Scenario: Model only in the catalog
- **WHEN** a call uses an OpenRouter model absent from `ai_pricing`
- **THEN** its cost is computed from the catalog prices, not recorded as zero

### Requirement: The budget ladder off, avisar, bloquear is enforced server-side
`PATCH /api/v1/ai/budget` SHALL require role `admin`, accept `alarm_threshold_pct` only in 50..99, refuse `off → bloquear` with 422 `invalid_state_transition`, refuse any mode other than `off` with `monthly_limit_cents < 100` with 422, set `enforcement_effective_at = now() + 72h` when arming `bloquear` (immediately when `confirmar_imediato` is true), and clear it when disarming.

#### Scenario: Skipping the warning step
- **WHEN** an admin moves the mode from `off` directly to `bloquear`
- **THEN** the response is 422 `invalid_state_transition` with `details.degrau_faltante = 'avisar'`

#### Scenario: Arming with grace period
- **WHEN** an admin moves from `avisar` to `bloquear` without `confirmar_imediato`
- **THEN** `ai_budgets.enforcement_effective_at` is 72 hours in the future

### Requirement: The ceiling blocks a model call only when all conditions hold
`decidirOrcamento` SHALL let the call proceed when mode is `off`, env `AI_BUDGET_ENFORCEMENT` is `off`, the purpose is in `PURPOSES_ISENTOS`, or the ceiling is below 100 cents, SHALL only warn below the ceiling or in `avisar` mode, and when it blocks `runModelCall` SHALL throw `LlmBudgetExceededError` and open at most one `agent_inbox_items` row of kind `budget_exceeded` with severity `critical`.

#### Scenario: Ceiling reached in bloquear after grace
- **WHEN** spend reaches the ceiling, the grace period has passed and a warning was already issued this month
- **THEN** the call is refused before reaching the provider and one open `budget_exceeded` inbox item exists

### Requirement: Usage is aggregated from llm_calls only
`GET /api/v1/ai/usage` SHALL require role `manager`, read `llm_calls` filtered by the session `organization_id` and UTC day range, and answer 422 `validation_failed` on invalid filters; `GET /api/v1/ai/budget` SHALL require role `manager`.

#### Scenario: Viewer requests usage
- **WHEN** a user with role `viewer` calls `GET /api/v1/ai/usage`
- **THEN** the request is refused by `requireRole` and no usage is returned
