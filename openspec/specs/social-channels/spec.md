# social-channels Specification

## Purpose
Social-network inbox channels (Instagram and Facebook Messenger) brokered by the partner provider labelled "Zernio" (`SOCIAL_PROVIDER = 'zernio_social'`). An admin links a provider profile and API key (stored encrypted, one per organization, in `channel_integrations`), authorizes networks via OAuth, and connects an account's inbox through `GET/POST /api/v1/channels/social`; each connected account becomes a `channel_sessions` row with `provider = 'zernio_social'`, a `webhook_path_token` and an encrypted webhook secret. The provider delivers events to the generic `POST /api/v1/webhooks/channel/[token]` (signature `x-zernio-signature`, `lib/channels/inbound.ts`, `lib/channels/zernio/webhook.ts`), which are parsed by `lib/channels/social/parser.ts` and ingested by `lib/channels/social/ingest.ts` through the shared Zernio ingest. Network catalog: `lib/channels/social/catalog.ts`; storage helpers: `lib/channels/social/store.ts`. Seed doc: `docs/specs/22-spec-zernio-desvincular-perfil-e-chave-dupla.md`.

## Requirements

### Requirement: Admin-only social connection management
`GET /api/v1/channels/social` SHALL require role `admin`, and `POST /api/v1/channels/social` SHALL require `requireSupportWrite`, role `admin` and satisfied MFA (403 `mfa_required`), accept only the actions `profiles`, `configure`, `authorize`, `inbox`, `health`, `disconnect` and `unlink` (400 `validation_error` for any other body), and rate-limit to 20 requests per 60 s per organization (429 `rate_limited` with `Retry-After: 60`).

#### Scenario: Unknown action
- **WHEN** an admin posts `{ "action": "delete_everything" }`
- **THEN** the response is 400 `validation_error`

#### Scenario: Burst of connection attempts
- **WHEN** an organization sends a 21st POST within 60 seconds
- **THEN** the response is 429 `rate_limited` with header `Retry-After: 60`

### Requirement: One encrypted provider profile per organization
`configure` SHALL verify that the `profile_id` belongs to the API key at the provider (422 otherwise), refuse with 409 when the organization is already linked to a different profile, refuse with 422 when the key cannot be encrypted, and upsert `channel_integrations (organization_id primary key, profile_id, credential_encrypted)`, a table revoked from `anon`/`authenticated` and granted only to `service_role`.

#### Scenario: Switching to another profile
- **WHEN** an organization linked to profile A configures profile B
- **THEN** the response is 409 (`social_unavailable`) and `channel_integrations` keeps profile A

### Requirement: Inbox only for supported networks
`inbox` SHALL connect an account only when it is active in the linked profile and its platform has inbox support (`instagram` or `facebook` in `SOCIAL_NETWORKS`), returning 422 otherwise, and SHALL create/update a `channel_sessions` row with `provider = 'zernio_social'`, `webhook_path_token`, `webhook_secret_encrypted`, and register the webhook `/api/v1/webhooks/channel/{token}` at the provider.

#### Scenario: LinkedIn account
- **WHEN** an admin requests `inbox` for a LinkedIn account
- **THEN** the response is 422 and no `channel_sessions` row is created

#### Scenario: Already connected account
- **WHEN** the account already has a `WORKING` session with a registered webhook
- **THEN** the response body has `already_connected: true` with the existing `channel_id`

### Requirement: Unlink requires no active social channels
`unlink` SHALL refuse with 409 while any non-archived `zernio_social` session exists, return 404 when no profile is linked, and otherwise delete the organization's `channel_integrations` row and resolve connection-health alerts of its social sessions, auditing `channel.social_desvinculado`.

#### Scenario: Unlink with an active channel
- **WHEN** an admin unlinks while an Instagram channel is still active
- **THEN** the response is 409 and the `channel_integrations` row remains

### Requirement: Webhook signature and routing
`POST /api/v1/webhooks/channel/[token]` SHALL return 404 `not_found` for unknown tokens, 200 `ignored` (`canal_arquivado`) for archived sessions, and 401 `unauthorized` (`bad_signature`) unless `x-zernio-signature` equals the hex HMAC-SHA256 of the raw body with the session's decrypted secret (at least 16 chars), compared in constant time.

#### Scenario: Unsigned social event
- **WHEN** an event arrives for a social session without `x-zernio-signature`
- **THEN** the response is 401 with reason `bad_signature`

### Requirement: Events are scoped to the connected account
Because provider webhooks are account-wide, a signed social event whose account does not match the session's `zernio_account_id` SHALL be answered 200 with `status: "ignored"` and SHALL NOT create messages.

#### Scenario: Event for another profile's account
- **WHEN** a signed event references an account different from the session's `zernio_account_id`
- **THEN** the response is 200 with `reason: "evento_de_outra_conta"` and no `messages` row is written

### Requirement: Idempotent social message ingest
Social inbound messages SHALL be stored with `external_id = 'social:{account_id}:{platform_message_id}'` under the `(organization_id, external_id)` unique constraint, treating `23505` as a duplicate, and SHALL pass through the shared post-ingest effects (`aplicarEfeitosPosEntrada`).

#### Scenario: Provider redelivers a DM
- **WHEN** the same Instagram DM is delivered twice
- **THEN** exactly one `messages` row exists for it

### Requirement: Audit of social configuration
Successful `POST /api/v1/channels/social` mutations SHALL write an audit row with action `channel.social_disconnected` (disconnect), `channel.social_desvinculado` (unlink) or `channel.social_configured` (configure, authorize, inbox; `profiles` and `health` are read-only and not audited), with `resourceType = 'social_connections'` and the acting user.

#### Scenario: Disconnect is audited
- **WHEN** an admin disconnects an account
- **THEN** an audit row `channel.social_disconnected` with `metadata.account_id` is written
