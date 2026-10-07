# inbound-webhooks Specification

## Purpose
Inbound webhooks ("webhook sources") let external forms and systems (landing pages, Respondi, RD Station, Elementor, Zapier/n8n) create leads in a tenant through a public URL whose path token identifies the organization. Key paths: UI `app/app/webhooks` (SourcesTab, CreateSourceDialog, SourceDetail, CapturasTab); API `app/api/v1/webhook-sources` (GET/POST), `webhook-sources/[id]` (PATCH/DELETE), `webhook-sources/[id]/events` (GET), public `app/api/v1/webhooks/in/[token]` (POST), capture history `app/api/v1/lead-captures` (GET); libs `lib/webhooks/inbound.ts` (field mapping + HMAC), `lib/webhooks/captacao.ts`, `lib/webhooks/{respondi,rdstation,elementor}.ts`, `lib/webhooks/secrets.ts`, `lib/webhooks/assinatura.ts` (header name), `lib/operacao/entradas-automaticas.ts`; tables `webhook_sources`, `webhook_lead_captures`, `webhook_events_log`; cron `app/api/v1/cron/webhook-log-retention`. Integrator doc: `docs/webhooks/autorizacao-ia-formulario.md`.

## Requirements

### Requirement: Manager-only source management
Handlers under `/api/v1/webhook-sources` and `GET /api/v1/lead-captures` SHALL call `requireRole("manager")`, mutating handlers SHALL call `requireSupportWrite()` first, and RLS policies `webhook_sources_select` / `webhook_sources_manager_write` and `webhook_lead_captures_manager_read` SHALL restrict rows to the caller's organizations.

#### Scenario: Create a source
- **WHEN** a manager calls `POST /api/v1/webhook-sources` with `name`, `default_pipeline_id` and `default_stage_id`
- **THEN** the response is 201 with a row whose `path_token` was generated server-side (`randomBytes(24)` base64url) and whose `kind = 'lead_capture'`

### Requirement: Secret is write-only and encrypted
A source `secret` (16–200 chars) SHALL be stored only as `webhook_sources.secret_encrypted` via `encryptWebhookSecret`, reads SHALL expose only `has_secret`, and when encryption is unavailable POST/PATCH SHALL fail with 422 `encryption_unavailable`.

#### Scenario: Listing sources
- **WHEN** a manager calls `GET /api/v1/webhook-sources` for a source that has a secret
- **THEN** the item contains `has_secret: true` and no secret or `secret_encrypted` field

### Requirement: Organization resolved from the path token only
`POST /api/v1/webhooks/in/{token}` SHALL resolve the tenant exclusively from `webhook_sources.path_token` (service-role lookup) and use that row's `organization_id`, `default_pipeline_id` and `default_stage_id`, never values from the body, returning 404 `not_found` for a token shorter than 8 characters, unknown, or of an inactive source (`is_active = false`).

#### Scenario: Inactive source
- **WHEN** a payload is posted to the token of a source with `is_active = false`
- **THEN** the response is 404 with code `not_found` and no lead is created

### Requirement: Per-token rate limit
The public capture endpoint SHALL allow at most 60 requests per minute per path token (`checkRateLimit("webhook_in:<token>", 60, 60)`) and answer excess requests with 429 `rate_limited` and header `Retry-After: 60`.

#### Scenario: Burst over limit
- **WHEN** the 61st request in one minute arrives for the same token
- **THEN** the response is 429 with code `rate_limited` and `Retry-After: 60`

### Requirement: HMAC signature for sources with a secret
When the source has `secret_encrypted`, the endpoint SHALL require header `x-deskcomm-signature` equal to the hex HMAC-SHA256 of the raw body (compared with `timingSafeEqual`), failing closed when the secret cannot be decrypted, and SHALL answer 401 `unauthenticated` while recording a `webhook_lead_captures` row with `outcome = 'recusado'` and `reject_reason` `assinatura_invalida` or `assinatura_indecifravel`.

#### Scenario: Wrong signature
- **WHEN** a signed source receives a body whose `x-deskcomm-signature` does not match
- **THEN** the response is 401 with code `unauthenticated`, an audit `webhook.inbound_invalid_signature` is written and a `recusado` capture row is recorded

#### Scenario: Source without secret
- **WHEN** a source without a secret receives an unsigned JSON body with a phone field
- **THEN** the request is accepted and a lead is created

### Requirement: Payload formats and field mapping
The endpoint SHALL accept `application/json` and `application/x-www-form-urlencoded` on the same URL, map name/phone/email through `field_map` or the default aliases (plus Respondi, RD Station and Elementor shapes), put `utm_*` keys into source metadata, and return 400 `invalid_request` for malformed JSON or when no name, phone or email can be mapped (recording `reject_reason = 'sem_campo_mapeavel'`).

#### Scenario: Unmappable form
- **WHEN** a JSON body contains none of the name, phone or email aliases
- **THEN** the response is 400 with code `invalid_request` and a capture row with `outcome = 'recusado'` is stored

### Requirement: Idempotency by external_id
A capture carrying `external_id` (or the Respondi/RD Station equivalent, truncated to 255 chars) SHALL return the existing lead instead of creating a new one, backed by the unique index `uniq_crm_leads_org_source_external`, recording the capture with `outcome = 'duplicado'`.

#### Scenario: Retry of the same submission
- **WHEN** the same payload with `external_id = "abc"` is posted twice
- **THEN** both responses are 200 with the same `data.lead_id` and the second capture row has `outcome = 'duplicado'`

### Requirement: Lead creation and response
A valid capture SHALL create the lead through `createLeadHandler` with actor `{ type: "webhook_source" }`, log the raw request in `webhook_events_log` (provider `generic`, `event_type = 'lead_capture.received'`, without `authorization`/`cookie` headers), record a `criado` capture, and respond 200 `{ lead_id }`, or 303 to `redirect_to` for form-encoded posts when the source has one.

#### Scenario: HTML form with redirect
- **WHEN** a form-urlencoded submission is accepted by a source with `redirect_to` set
- **THEN** the response is a 303 redirect to `redirect_to`

### Requirement: AI authorization requires a signed source
`PATCH /api/v1/webhook-sources/{id}` SHALL refuse enabling `authorize_ai_on_capture` on a source without a secret, or removing the secret while it is enabled, with 422 `signature_required`.

#### Scenario: Enable AI on unsigned source
- **WHEN** a manager PATCHes `authorize_ai_on_capture: true` on a source with no secret
- **THEN** the response is 422 with code `signature_required`

### Requirement: Capture history and retention
`GET /api/v1/lead-captures` SHALL list `webhook_lead_captures` of the caller's organization with keyset pagination (400 `invalid_cursor` on a bad cursor) and filters `source_id` and `outcome` (`criado`, `duplicado`, `recusado`), and the cron `webhook-log-retention` SHALL prune captures older than the retention policy (default `RETENCAO_CAPTACAO_DIAS_PADRAO = 365` days).

#### Scenario: Filter rejected captures
- **WHEN** a manager calls `GET /api/v1/lead-captures?outcome=recusado`
- **THEN** the response is 200 with only rows of the active organization whose `outcome = 'recusado'`
