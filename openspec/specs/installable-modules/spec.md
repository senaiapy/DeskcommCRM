# installable-modules Specification

## Purpose
Official optional modules installed once per installation (ADR-0002, D3): a module's tables are created by a
fixed provisioning function `fn_<slug>_provisionar()` shipped in the baseline, not by the baseline for every
tenant. Installation goes through the same receipt ledger (`extension_operations`) and platform-admin gate as
`extensions`, and `modulos_instalados` records what is installed. The first and only catalogued module is
covered by `honorarios-module`; declarative packages are covered by `extensions`. Design note:
`docs/specs/modulo-instalado-onda-2.md`.

## Requirements

### Requirement: Catalog listing for the installation admin
`GET /api/v1/modulos` SHALL require `requireExtensionPlatform` (full-scope platform admin, no support session, `aal2` when required) and return `disponiveis` from `CATALOGO_DE_MODULOS` and `instalados` from `modulos_instalados` (`modulo`, `estado`, `instalado_em`, `reaplicado_em`, `motivo_suspensao`) with `Cache-Control: no-store`.

#### Scenario: Organization admin
- **WHEN** an organization `admin` without platform scope calls `GET /api/v1/modulos`
- **THEN** the response is 403 `forbidden`

#### Scenario: Platform admin
- **WHEN** a full-scope platform admin calls it
- **THEN** the body lists `honorarios` under `disponiveis` and the installed rows under `instalados`

### Requirement: Install a module at installation level
`POST /api/v1/modulos/instalar` SHALL require `requireSupportWrite`, `requireExtensionPlatform`, a UUID `Idempotency-Key` and a body `{modulo}`; a slug absent from `CATALOGO_DE_MODULOS` SHALL fail with 404 `extension_module_unknown` before reaching the database, and a successful first install SHALL return `{operationId, appliedNow: true}` and audit `modulo.instalado`.

#### Scenario: Unknown module
- **WHEN** the body is `{"modulo": "comanda"}`
- **THEN** the response is 404 `extension_module_unknown` and no row is written

#### Scenario: Repeated request
- **WHEN** the same `Idempotency-Key` and module are sent again
- **THEN** the response is `appliedNow: false` with the same `operationId` and no new audit row

### Requirement: The database decides which modules exist
`public.fn_modulo_instalar(p_actor, p_operation, p_modulo)` SHALL be `security definer`, executable only by `service_role`, validate the slug against `^[a-z][a-z0-9_]{1,40}$`, re-check the actor after taking `pg_advisory_xact_lock(255, 1)`, raise `extension_module_unknown` when `to_regprocedure('public.fn_<slug>_provisionar()')` is null and `extension_core_update_in_progress` during a core update, then run the provisioner, upsert `modulos_instalados` with `estado = 'ativo'`, insert a completed `extension_operations` row of kind `module_install` and send `pg_notify('pgrst', 'reload schema')`.

#### Scenario: Key reused with another module
- **WHEN** an operation id already used for `honorarios` is sent with a different module
- **THEN** the function raises `extension_idempotency_conflict`

#### Scenario: Module usable immediately
- **WHEN** the install commits
- **THEN** the PostgREST schema cache is reloaded so the module tables answer without a restart

### Requirement: Installed modules are re-applied on every update
`public.fn_reaplicar_modulos_instalados()`, run by the baseline appendix on each `update.sh`, SHALL re-run every installed module's provisioner, set `estado = 'ativo'` and `reaplicado_em = now()` on success, and on failure set `estado = 'suspenso'` with `motivo_suspensao = sqlerrm` instead of aborting the update; `fn_conferir_modulos_instalados()` SHALL raise when any module is `suspenso`.

#### Scenario: Provisioner missing in the new version
- **WHEN** an update ships without `fn_honorarios_provisionar()`
- **THEN** `modulos_instalados.estado` becomes `suspenso` with the error text in `motivo_suspensao`

### Requirement: Installed-module table is instance-wide and closed
`public.modulos_instalados` SHALL have no `organization_id`, have RLS enabled and all privileges revoked from `anon` and `authenticated`, so it is read and written only through `service_role` and the two functions above.

#### Scenario: Tenant user reads the table
- **WHEN** an authenticated session selects from `modulos_instalados` through the REST API
- **THEN** access is denied
