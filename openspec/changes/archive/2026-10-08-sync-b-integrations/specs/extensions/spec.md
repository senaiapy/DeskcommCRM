## ADDED Requirements

### Requirement: Finishing an install compares only the announced package keys
`fn_extensions_finish_install` (redefined by migration `20261002170000_0511_catalogo_oficial_instala.sql`) SHALL raise `extension_artifact_mismatch` when the manifest's `publisher`, `name`, `version`, `license`, `host_api`, `display` or `permissions` differ from the admitted catalog entry, when the digest or byte length differ, or when the manifest has a key outside its thirteen allowed keys, and SHALL ignore catalog-only showcase keys such as `publisher_label`, `homepage`, `repository`, `tags` and `published_at`.

#### Scenario: Official catalog entry with showcase fields
- **WHEN** an installation admin installs an official-catalog package whose entry carries `homepage` and `tags`
- **THEN** the install completes instead of failing with `extension_artifact_mismatch`

#### Scenario: Version announced differently
- **WHEN** the package manifest `version` differs from the catalog entry
- **THEN** the finish step raises `extension_artifact_mismatch`
