# extensions Specification

## Purpose
Declarative extension packages (profile `declarative`, host API 2): catalog admission, install/update, revert and
removal by the installation administrator, and per-organization activation, configuration and card opening, all
recorded as idempotent receipts in `extension_operations`. Packages carry no code, SQL, tables or screens; they
name capabilities from a closed list that the host maps to existing work screens. The law lives in
`docs/doctrine/extensoes.md` and the contract in `docs/specs/extensoes-declarativas-v1.md`. Official modules with
their own tables are covered by `installable-modules`; platform-admin identity is covered by
`platform-admin-console` and `mfa-policy`; the receipt audit rows are covered by `audit-log`.

## Requirements

### Requirement: Installation-level operations require a full-scope platform admin
`POST /api/v1/extensions/catalogs`, `POST /api/v1/extensions/install`, `POST /api/v1/extensions/{id}/revert`, `POST /api/v1/extensions/{id}/remove` and `POST /api/v1/extensions/operations/{id}/cancel` SHALL call `requireSupportWrite` and `requireExtensionPlatform`, which accepts only a user with a non-revoked `platform_admins` row of `scope = 'full'`, outside a support session, and with an `aal2` session when `mfa_required` is set or an MFA factor is owed.

#### Scenario: Organization admin tries to install
- **WHEN** an organization `admin` who is not a platform admin calls `POST /api/v1/extensions/install`
- **THEN** the response is 403 `forbidden`

#### Scenario: Platform admin without second factor
- **WHEN** a full-scope platform admin with `mfa_required = true` calls it from an `aal1` session
- **THEN** the response is 403 `mfa_required`

#### Scenario: Unauthenticated
- **WHEN** the call carries no session
- **THEN** the response is 401 `unauthenticated`

### Requirement: Every operation is keyed by a UUID Idempotency-Key
Mutating extension routes SHALL require an `Idempotency-Key` header that is a UUID (422 `validation_failed` otherwise) and use it as the `extension_operations.id`; repeating the same request with the same key SHALL return the existing receipt with `applied_now = false`, and the same key with a different request fingerprint SHALL fail with 409 `extension_idempotency_conflict`.

#### Scenario: Missing key
- **WHEN** `POST /api/v1/extensions/install` is sent without `Idempotency-Key`
- **THEN** the response is 422 `validation_failed`

#### Scenario: Same key, different version
- **WHEN** a key already used to install version 1.0.0 is reused to install 1.1.0
- **THEN** the response is 409 `extension_idempotency_conflict`

### Requirement: Strict, bounded request bodies
Extension request bodies SHALL be `application/json` without content encoding (415 `validation_failed` otherwise) and read by `readExtensionBody` with a byte cap enforced on the bytes actually read (4096 bytes for operation bodies, parsed by `parseStrictJson` with depth 6, 64 nodes and 12 properties per object; 512 KiB for a catalog document), answering 413 `extension_payload_too_large` above the cap even when `Content-Length` is absent or false.

#### Scenario: Oversized catalog
- **WHEN** a catalog document of 600 KiB is posted to `/api/v1/extensions/catalogs`
- **THEN** the response is 413 `extension_payload_too_large`

### Requirement: Install downloads by digest and validates the manifest
`installExtension` SHALL prepare the operation through `fn_extensions_prepare_install` with the `expected_installation_revision` shown on screen, download the package from the admitted catalog origin at `/packages/{sha256}.json` (https only, except `EXTENSIONS_LOCAL_CATALOG_ORIGIN`), reject a body whose SHA-256 differs from the catalog entry with `extension_digest_mismatch`, require `profile = 'declarative'` and a `host_api` range that contains 2, and publish only through `fn_extensions_finish_install`.

#### Scenario: Tampered package
- **WHEN** the downloaded bytes do not hash to the catalog `sha256`
- **THEN** the operation is failed through `fn_extensions_fail_install` with `extension_digest_mismatch` and audit `extension.install_failed`

#### Scenario: Incompatible host API
- **WHEN** the manifest declares `host_api` `{min: 3, max: 3}`
- **THEN** the operation fails with `extension_incompatible`

#### Scenario: Successful install
- **WHEN** a compatible package is installed for the first time
- **THEN** the receipt is completed and audit `extension.installed` records the version and digest

### Requirement: Capabilities are a closed list mapped by the host
A package SHALL only declare permissions from `EXTENSION_PERMISSIONS` (`navigation.tasks|inbox|kanban|contacts|agenda|radar`, `theme.apply`) and card capabilities from `EXTENSION_CAPABILITIES`; `POST /api/v1/extensions/{id}/open` SHALL resolve the destination with `destinoDaCapacidade` from host constants, never from package data, and SHALL return 409 `extension_card_unavailable` when the card is absent from the installed version.

#### Scenario: Opening a card
- **WHEN** a viewer opens card `x` whose capability is `tasks.open` at the current revision
- **THEN** the response is 200 `{href}` with the host route for tasks

#### Scenario: Stale tab
- **WHEN** `expected_revision` differs from the current guide revision
- **THEN** the response is 409 `extension_revision_conflict`

### Requirement: Organization context guard
Organization-scoped extension routes (`GET /api/v1/extensions`, `GET /api/v1/extensions/{id}`, `PUT /api/v1/extensions/{id}/configuration`, `POST /api/v1/extensions/{id}/open`, `GET /api/v1/extensions/operations/{id}`) SHALL require an `X-Expected-Organization-Id` UUID header (400 `validation_failed` when missing) that matches the active organization (409 `extension_context_changed` otherwise).

#### Scenario: Organization switched in another tab
- **WHEN** the header carries organization A while the session's active organization is B
- **THEN** the response is 409 `extension_context_changed`

### Requirement: Organization admin activates and configures
`PUT /api/v1/extensions/{id}/configuration` SHALL require `requireRole("admin")` in the active organization, refuse support sessions with 403 `forbidden`, take `{expected_revision, enabled, configuration}` and write through `fn_extensions_configure`; listing and reading the guide SHALL require only `requireRole("viewer")`.

#### Scenario: Manager tries to enable
- **WHEN** a `manager` sends the configuration PUT
- **THEN** the response is 403

### Requirement: Revert and remove are logical and revision-checked
`POST /api/v1/extensions/{id}/revert` SHALL restore the previous version for all organizations without downloading, and `POST /api/v1/extensions/{id}/remove` SHALL disable the extension in every organization without deleting rows; both SHALL require `expected_installation_revision` and fail with 409 `extension_version_changed` when `extension_installations.revision` no longer matches.

#### Scenario: Remove keeps data
- **WHEN** the platform admin removes an installed extension
- **THEN** `extension_installations.removed_at` is set, every `organization_extensions` row is kept with `enabled = false`, and the revision is incremented

#### Scenario: Stale revision
- **WHEN** the revert is sent with an `expected_installation_revision` older than the current one
- **THEN** the response is 409 `extension_version_changed`

### Requirement: Unknown failures surface as unconfirmed
`extensionFailure` SHALL map `ExtensionServiceError` to its own code and status, `ExtensionError` to 503 for `extension_download_failed`, 413 for `extension_payload_too_large` and 422 otherwise, Zod errors to 422 `validation_failed`, and any other error to 503 `upstream_unavailable` asking the operator to check the receipt history before retrying.

#### Scenario: Database connection drops after publish
- **WHEN** `fn_extensions_finish_install` fails with an unknown error
- **THEN** the response is 503 `upstream_unavailable`

### Requirement: Finishing an install compares only the announced package keys
`fn_extensions_finish_install` (redefined by migration `20261002170000_0511_catalogo_oficial_instala.sql`) SHALL raise `extension_artifact_mismatch` when the manifest's `publisher`, `name`, `version`, `license`, `host_api`, `display` or `permissions` differ from the admitted catalog entry, when the digest or byte length differ, or when the manifest has a key outside its thirteen allowed keys, and SHALL ignore catalog-only showcase keys such as `publisher_label`, `homepage`, `repository`, `tags` and `published_at`.

#### Scenario: Official catalog entry with showcase fields
- **WHEN** an installation admin installs an official-catalog package whose entry carries `homepage` and `tags`
- **THEN** the install completes instead of failing with `extension_artifact_mismatch`

#### Scenario: Version announced differently
- **WHEN** the package manifest `version` differs from the catalog entry
- **THEN** the finish step raises `extension_artifact_mismatch`
