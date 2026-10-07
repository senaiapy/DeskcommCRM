# white-label-branding Specification

## Purpose
Runtime white-label: one published Docker image serves every brand. The installation brand lives in the singleton table `public.platform_branding` (`app_name`, `logo_url`, `accent_hex`, `show_powered_by`, `seeded_from_env`, `fallback_at`, `fallback_reason`), read and seeded by `lib/branding/instalacao.ts`; the organization brand lives in `organizations.settings.branding`, written through RPC `fn_definir_marca_da_organizacao` (`app/actions/settings/updateMarcaDaOrganizacao.ts`) and read by `lib/branding/organizacao.ts`; layers are combined by the pure resolver `lib/branding/resolve.ts`, rendered from `app/layout.tsx`, and flattened for DOM-less outputs (e-mail, MFA issuer, icons) by `marcaDaSaida()` in `lib/branding/saida.ts`. Platform admins edit the installation brand at `/admin/marca` (`app/actions/settings/updateBranding.ts`, `updateCustomBrandingCss.ts`); org admins at `/app/settings/marca`; logos go through `POST|DELETE /api/v1/marca/logo`. `APP_NAME`, `APP_LOGO_URL`, `APP_ACCENT_HEX` are only the seed and rollback floor. Leakage of the product brand is policed by `tests/unit/branding.test.ts`.

## Requirements

### Requirement: Database above environment
The resolved brand SHALL take `organizations.settings.branding` over `platform_branding` over the `APP_NAME`/`APP_LOGO_URL`/`APP_ACCENT_HEX` seed over the product default (`DEFAULT_APP_NAME = "DeskcommCRM"`), and `platform_branding` SHALL be seeded from those env vars (`seeded_from_env = true`) only when the row is missing or incomplete.

#### Scenario: Operator edits the brand in the console
- **WHEN** `platform_branding.app_name` is "Acme CRM" while `APP_NAME` is "Old Name"
- **THEN** every screen and e-mail shows "Acme CRM"

#### Scenario: Image rollback onto newer schema
- **WHEN** the image is rolled back to a version that does not know `platform_branding`
- **THEN** the brand degrades to the `APP_NAME` value instead of the product default

### Requirement: Single-row installation table
`public.platform_branding` SHALL hold at most one row, enforced by the check `platform_branding_singleton (id = 1)`.

#### Scenario: Second row
- **WHEN** an INSERT with `id = 2` is attempted
- **THEN** Postgres rejects it with the singleton check violation

### Requirement: Resolvers never throw
`marcaDaInstalacao()` SHALL return null on a read error and `marcaDaSaida()` SHALL catch any error and return the product default, so that `branding()` in `app/layout.tsx` never turns a brand failure into a 500; failures SHALL be recorded in `platform_branding.fallback_at`/`fallback_reason`.

#### Scenario: Database unreachable during render
- **WHEN** reading `platform_branding` fails
- **THEN** pages render with the fallback brand and respond 200

### Requirement: Process cache with write invalidation
The installation brand SHALL be memoized per process for 30 seconds (`TTL_MS = 30_000`) and invalidated immediately by `invalidarMarcaDaInstalacao()` on every write path.

#### Scenario: Admin saves a new accent color
- **WHEN** a platform admin saves a new `accent_hex`
- **THEN** the next render in the same process uses the new color without waiting for the TTL

### Requirement: Who may change each layer
Installation branding SHALL require `requirePlatformAdminEscrita()` (full scope, MFA satisfied) and audit `platform_branding.updated`; organization branding SHALL require `podeAdministrarEmpresa()`, refuse support sessions and unsatisfied MFA (`forbidden_role`, `mfa_required`), and audit `org.branding_updated`.

#### Scenario: Manager edits organization brand
- **WHEN** a `manager` without company-admin rights submits the organization brand form
- **THEN** the action returns `forbidden_role` and `organizations.settings.branding` is unchanged

### Requirement: Logo upload safety
`POST /api/v1/marca/logo` SHALL call `requireSupportWrite()`, require `escopo` ∈ {`instalacao`, `organizacao`} (422 `validation_failed` otherwise), refuse files larger than 512 KiB with 413 `payload_too_large`, refuse SVG content with 415 `logo_svg_recusado`, accept only PNG or JPG detected from the bytes (415 `unsupported_media_type` otherwise), and throttle repeated changes with 429 `rate_limited`.

#### Scenario: SVG renamed to .png
- **WHEN** an admin uploads SVG bytes with `Content-Type: image/png`
- **THEN** the response is 415 `logo_svg_recusado`

### Requirement: The LGPD PDF carries no brand
The LGPD export PDF (`lib/lgpd/pdf-renderer.tsx`) SHALL name the data controller `organizations.legal_name` and the resolved DPO and SHALL NOT render the reseller or product brand.

#### Scenario: White-labelled reseller installation
- **WHEN** a data subject export PDF is generated in an installation branded "Acme CRM"
- **THEN** the PDF footer names the organization's legal name and not "Acme CRM"
