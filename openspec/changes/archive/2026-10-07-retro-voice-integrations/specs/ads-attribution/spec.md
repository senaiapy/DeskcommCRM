## Purpose
Paid-ads attribution and offline conversions for Meta Ads and Google Ads: public capture endpoints turn an ad click (UTM,
`gclid`/`gbraid`/`wbraid`) into a short `[ref:XXXXXX]` token carried in the WhatsApp message, the contact is stamped with
the attribution, and event handlers report purchases and stage conversions back to the platform with an idempotent ledger
(`ad_conversion_dispatches`). It also reads Meta campaign insights for the `/app/ads/meta` screen. Lead/pipeline state is
covered by the CRM core capabilities (`crm-leads`, `kanban`); Google Calendar OAuth by `google-calendar`; the event_log
drain by the platform queue capability.

## ADDED Requirements

### Requirement: Meta landing capture by UTM
`GET /api/v1/anuncios/meta/{org}` SHALL be public, resolve the organization by `organizations.slug`, rate-limit per organization and IP (429 with `Retry-After`), keep only normalized UTM keys, insert a `meta_ads_click_refs` row with a short token, and answer an exit page to `wa.me` with `{token}` replaced in the landing `messageTemplate`.

#### Scenario: Visit without UTM
- **WHEN** the landing URL is opened without any UTM parameter
- **THEN** no `meta_ads_click_refs` row is created and the visitor is still sent to WhatsApp with the template text without a ref

#### Scenario: Unknown organization slug
- **WHEN** the `{org}` slug matches no organization or no `meta_ads_landing_pages` config exists
- **THEN** the exit page has no WhatsApp destination and nothing is stored

### Requirement: Google landing capture by click identifier
`GET /api/v1/anuncios/google/{org}` SHALL be public, rate-limited per organization and IP, store a valid `gclid`/`gbraid`/`wbraid` (unresolved URL macros refused) in `google_ads_click_refs` with a short token, and never send a conversion itself.

#### Scenario: Unresolved macro
- **WHEN** the query carries a literal macro such as `{gclid}` instead of a real identifier
- **THEN** no click ref is stored and the visitor is still sent to the configured WhatsApp without a ref

### Requirement: Trackable links
`GET /api/v1/rastreio/{id}` SHALL accept only a UUID, rate-limit 30 hits per 60 s per link and IP (429, `Retry-After: 60`), read only `ad_tracking_links` rows with `enabled=true`, take `organization_id` from the stored row, and answer with `Cache-Control: no-store` and `Referrer-Policy: no-referrer`.

#### Scenario: Disabled link
- **WHEN** a visitor opens the URL of a link whose `enabled` is false
- **THEN** the exit page has no destination

### Requirement: Ref token matched on inbound message
When an inbound message contains `[ref:XXXXXX]` (pattern `PADRAO_DO_REF`), post-ingest (lib/channels/pos-entrada.ts) SHALL match the click ref of the same `organization_id` only while `matched_at` is null, setting `matched_at` and `contact_id`, so each token attributes at most one contact.

#### Scenario: Token reused by a second contact
- **WHEN** a second contact sends a message with a token already matched
- **THEN** the update matches no row and the second contact receives no attribution from that token

### Requirement: Attribution read for sending
`lerAtribuicao` (lib/conversoes/leitura-da-atribuicao.ts) SHALL derive the platform and click from `contacts.source_metadata` (`ad_platform`, `ad_source_id`), treat a `site` origin with a Meta `utm_source` as `meta_ads` without click, and otherwise return `sem_atribuicao` or `plataforma_desconhecida` without sending.

#### Scenario: Unknown platform value
- **WHEN** a contact has `ad_source_id` and an `ad_platform` outside the known vocabulary
- **THEN** the conversion is skipped with reason `plataforma_desconhecida`

### Requirement: Purchase conversion handler
The `conversaoDeVendaHandler` registered in lib/event-log/register-handlers.ts SHALL consume `lead.won`, `lead.stage_changed` and `ad_conversion.retry_requested`, re-read `crm_leads` filtered by `organization_id` with the service client, ignore the payload `status`, and send `Purchase` only when the lead `status` is `won` and the contact has attribution.

#### Scenario: Stage change that is not a win
- **WHEN** a `lead.stage_changed` event arrives for a lead whose `status` is `open`
- **THEN** the handler returns skipped `nao_e_ganho` and writes no ledger row

#### Scenario: Read failure
- **WHEN** reading `crm_leads` fails
- **THEN** the handler returns status `retry` with a future `retry_at`

### Requirement: Idempotent conversion ledger
Every send attempt SHALL upsert `ad_conversion_dispatches` on `(organization_id, lead_id, event_name)` with `status` `sent`, `skipped` or `error`, `event_id` `<lead_id>:<event_name>`, value in `value_cents` + `currency`, and handlers SHALL skip with `ja_enviada` when the existing row is `sent`.

#### Scenario: Lead won twice
- **WHEN** a lead already reported as `Purchase` with `status='sent'` is reopened and won again
- **THEN** no second conversion is sent

### Requirement: Stage conversions per platform
`conversaoDeQualificacaoHandler` (Google, rules in `google_ads_conversion_rules`) and `conversaoDeEtapaMetaHandler` (Meta) SHALL send a stage conversion on `lead.stage_changed` only for open stages of the same organization, only for leads attributed to that rule's platform, and never for movements older than the rule's `configured_at`.

#### Scenario: Lead from Google entering a Meta stage rule
- **WHEN** a lead attributed to `google_ads` enters a stage with a Meta rule
- **THEN** the Meta handler skips it with reason `etapa_sem_origem_meta`

### Requirement: Platform transports
The Meta transport SHALL post to `<graph>/<datasetId>/events` with `event_id`, using `action_source=business_messaging` with `ctwa_clid` when a click exists and `action_source=system_generated` with the SHA-256 hashed phone otherwise, and the Google transport SHALL call `customers/{id}:uploadClickConversions` with the `developer-token` header, returning a permanent failure without calling Google when `GOOGLE_ADS_OAUTH_CLIENT_ID`, `GOOGLE_ADS_OAUTH_CLIENT_SECRET` or `GOOGLE_ADS_DEVELOPER_TOKEN` is missing.

#### Scenario: Installation without Google Ads developer token
- **WHEN** a Google-attributed purchase is processed and `GOOGLE_ADS_DEVELOPER_TOKEN` is empty
- **THEN** no request is sent to Google and the attempt is reported as a permanent failure naming the missing variables

#### Scenario: Meta purchase without click and without phone
- **WHEN** a Meta-attributed purchase has an empty click and the contact has no phone
- **THEN** nothing is posted to Meta and the attempt is a permanent failure

### Requirement: Manual conversion retry
`POST /api/v1/leads/{id}/conversion/retry` SHALL require role `admin`, accept `event_name` `Purchase`, `QualifiedLead`, `Etapa:<uuid>` or `MetaEtapa:<uuid>` (400 `validation_failed` otherwise), and call `fn_solicitar_reenvio_conversao` with the active `organization_id`, auditing `ad_conversion.retry_requested`.

#### Scenario: Invalid event name
- **WHEN** an admin posts with `event_name=Lead`
- **THEN** the response is 400 `validation_failed`

### Requirement: Google Ads OAuth connection
`GET /api/v1/plataformas-de-anuncio/google/connect` SHALL require role `admin` and redirect to Google consent with a `state` signed by `INTERNAL_SECRET` carrying the organization; the callback SHALL verify `state` before exchanging the code, refuse a response without refresh token (`?erro=sem_refresh_token`), and store only the encrypted refresh token in `ad_platform_connections.google_refresh_token_encrypted`, always redirecting to `/app/settings/conversoes`.

#### Scenario: Consent without refresh token
- **WHEN** Google returns tokens without `refresh_token`
- **THEN** the browser is redirected to `/app/settings/conversoes?erro=sem_refresh_token` and nothing is stored

### Requirement: Meta ads insights read
`GET /api/v1/ads/meta/accounts` and `GET /api/v1/ads/meta/campaigns` SHALL require role `manager`, read the credential of the active organization from `ad_insights_connections` through the service client, and map failures to `ads_sem_conexao` (422), `ads_token_invalido` (422), `ads_permissao_insuficiente` (422), `ads_limite_de_chamadas` (422) or `upstream_unavailable` (502).

#### Scenario: No connection
- **WHEN** a manager opens the campaigns table without a Meta insights connection
- **THEN** the response is 422 `ads_sem_conexao`
