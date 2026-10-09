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

### Requirement: Usage totals are summed in Postgres by fn_uso_de_ia
`GET /api/v1/ai/usage` SHALL obtain its totals from one call to the `security invoker` function `public.fn_uso_de_ia(p_org, p_desde, p_ate, p_agent_id, p_purpose)` (migration `20261007131331_0586_uso_de_ia_agregado_no_banco.sql`) on the session client, never by summing `llm_calls` rows in TypeScript, and SHALL answer 500 `internal_error` when the RPC fails or returns a payload that does not match `usoDeIaSchema`.

#### Scenario: More than 1000 calls in the period
- **WHEN** an organization has 5000 `llm_calls` rows in the requested range
- **THEN** the daily totals include the most recent days instead of stopping at the 1000th row

#### Scenario: Function returns an unexpected shape
- **WHEN** `fn_uso_de_ia` returns a payload that fails `usoDeIaSchema`
- **THEN** the response is 500 `internal_error` and no partial usage is shown

### Requirement: The plan AI ceiling blocks only the installation key
When billing is on (`cobrancaLigadaComMemo`) and the call uses the installation key (`origemDaChave = 'chave_da_instalacao'`), `aplicarOrcamento` SHALL read `fn_limite_do_plano(org, 'ia_usd_cents')` and `fn_gasto_de_ia_do_mes(org)` through `SQL_TETO_DO_PLANO` before the organization budget, and when `decidirTetoDoPlano` returns `bloquear` SHALL throw `LlmBudgetExceededError` and open an `agent_inbox_items` row of kind `budget_exceeded` with `ref_kind = 'plano'`; env `AI_BUDGET_ENFORCEMENT=off` and purposes in `PURPOSES_ISENTOS` SHALL bypass it, and the same statement SHALL resolve the open `plano` item once spend is below the ceiling or the ceiling is null.

#### Scenario: Organization with its own key
- **WHEN** the plan ceiling is exhausted but the call uses the organization's own credential
- **THEN** the call proceeds and no `ref_kind = 'plano'` item is opened

#### Scenario: Ceiling reached on the installation key
- **WHEN** the month's spend reaches the plan's `ia_usd_cents` and the call uses the installation key
- **THEN** the call is refused before reaching the provider and one open `budget_exceeded` item with `ref_kind = 'plano'` exists

#### Scenario: Month turns over
- **WHEN** the next call runs after spend fell below the ceiling
- **THEN** the open `plano` item becomes `status = 'resolved'`

### Requirement: Organization budget notices never touch the plan notice
The retraction in `PATCH /api/v1/ai/budget`, the retraction CTE of `SQL_ORCAMENTO` and the open-notice count in `getBudgetStatus` SHALL only consider `agent_inbox_items` of kind `budget_exceeded`/`budget_warning` whose `ref_kind` is null or `ai_budget`.

#### Scenario: Admin raises the organization budget
- **WHEN** an admin raises `ai_budgets.monthly_limit_cents` while a plan `budget_exceeded` item is open
- **THEN** the plan item (`ref_kind = 'plano'`) stays open

### Requirement: Side classifiers stop at the organization ceiling
`podeGastarComIa(organizationId, purpose)` SHALL answer `pode: true` without further reads when `ai_budgets.enforcement_mode` is `off` or missing, otherwise apply the same pure `decidirOrcamento` used by the engine with this month's `budget_warning` count, answer `pode: false` only on a `bloquear` verdict, never open an inbox item, and answer `pode: true` with `leitura_falhou` when any read fails; `workers/ai-sentiment-worker.ts` SHALL call it with purpose `sentiment_classify` before any paid call and skip with reason `orcamento_de_ia_estourado` when it refuses.

#### Scenario: Ceiling reached in bloquear mode
- **WHEN** an inbound message arrives for an organization in `bloquear` mode whose month spend is past the ceiling and a warning was already issued
- **THEN** the sentiment consumer returns `skipped` with `orcamento_de_ia_estourado` and no `llm_calls` row is written for the message

#### Scenario: Budget read fails
- **WHEN** the `ai_budgets` read returns an error
- **THEN** the classifier still runs and the failure is logged

### Requirement: The unpriced-call flag ignores failed calls and service transcription
`getBudgetStatus` SHALL set `gasto_incompleto` from `llm_calls` rows of the current UTC month with `cost_cents is null` and `status = 'ok'`, excluding rows with purpose `transcricao_de_audio` whose `origem_da_escolha` is in `ORIGENS_DA_TRANSCRICAO_POR_SERVICO` (`servico_da_instalacao`, `padrao_openai_compativel`).

#### Scenario: Provider refused the call
- **WHEN** the only unpriced row of the month has `status = 'erro'`
- **THEN** `gasto_incompleto` is false

### Requirement: ChatGPT subscription login is single-use and bound to one account
The `conectarLoginCodex` server action SHALL require an organization admin outside read-only support, verify the `state` HMAC with `INTERNAL_SECRET`, insert the state nonce into `calendar_oauth_nonces` before exchanging the code (a duplicate answers `estado_invalido`), require the token scopes `chatgpt.tokens.use.direct` and `resource.invoke` (`plano_nao_autorizado` otherwise), verify the `id_token` against issuer `https://auth.openai.com` and the state nonce (`identidade_invalida` otherwise), refuse `conta_diferente` when the stored login has another `subject`, and audit `ai.login_codex_conectado`.

#### Scenario: Same return URL pasted twice
- **WHEN** an admin submits a return URL whose state nonce was already used
- **THEN** the action answers `estado_invalido` and no token exchange happens

#### Scenario: Plan without API usage
- **WHEN** the exchanged tokens lack the `resource.invoke` scope
- **THEN** the action answers `plano_nao_autorizado` and nothing is stored

### Requirement: Subscription models are listed per organization
`GET /api/v1/ai/providers/{provider}/models` SHALL require role `manager` before validating the provider (404 `not_found` for an unsupported one), and for `openai-assinatura` SHALL list the models the organization's ChatGPT login exposes with `visibility = 'list'`, answer 409 `credential_invalid` when the organization has no usable login, and write the listed ids into `ai_provider_credentials.models_available` of the organization's active `openai-assinatura` credential.

#### Scenario: Subscription not connected
- **WHEN** a manager requests `/api/v1/ai/providers/openai-assinatura/models` for an organization without a ChatGPT login
- **THEN** the response is 409 `credential_invalid`

#### Scenario: Viewer probes providers
- **WHEN** a viewer requests the models of any provider id
- **THEN** the response is 403 before any 404 reveals which providers exist

### Requirement: Short classifications default to the provider's cheapest curated model
For purposes in `PONTOS_DE_TIER_ECONOMICO` (`stage_classifier`, `jailbreak_detect`) with no purpose binding and no env knob, the resolver SHALL pick the cheapest non-embedding `ai_models` row of the same provider when that provider's catalog is curated and the model is strictly cheaper than the agent's, keeping the agent's model as reserve; when the cheap model fails the seam SHALL record the failed row with `origem_da_escolha = 'economico_coberto_pela_reserva'`, repeat the call on the reserve model, and skip the cheap model for that organization for 30 minutes (`TTL_DA_FALHA_MS`).

#### Scenario: Cheap model refused by the provider
- **WHEN** the economic model returns a provider error during a stage classification
- **THEN** the classification is retried on the agent's model and the failed `llm_calls` row carries `origem_da_escolha = 'economico_coberto_pela_reserva'`

### Requirement: Curated Gemini 3.x models are priced in both tables
Migrations `20261008040255_0599_gemini_35_flash_lite_no_catalogo.sql` and `20261008045240_0600_catalogo_gemini_3x.sql` SHALL upsert `ai_models` rows for provider `google` (`gemini-3.5-flash-lite`, `gemini-3.1-flash-lite`, `gemini-3.6-flash`, `gemini-3.7-flash`, `gemini-3.8-flash`) with `supports_tools = true` and the same input/output cents per million in `ai_pricing`, without changing the Google default model.

#### Scenario: Agent picks Gemini 3.8 Flash
- **WHEN** an agent version uses `gemini-3.8-flash`
- **THEN** its calls are priced at 75/375 cents per million tokens instead of being recorded with unknown cost
