## ADDED Requirements

### Requirement: The installer accepts a Supabase URL only when GoTrue answers
`v_supabase_url` in `hostgator-setup-kit/install.sh` SHALL call `<url>/auth/v1/verify` and accept the address only when the answer is HTTP 400 with a JSON `"msg"` body, otherwise printing which address answered with which HTTP code and asking to check address and port.

#### Scenario: Coolify panel on the given port
- **WHEN** the operator types the address of another panel that answers HTTP 200
- **THEN** the validation fails saying the address answered but is not Supabase

### Requirement: The installer refuses to guess among several Traefik networks
`rede_do_traefik` SHALL use the only network of the host Traefik container when there is one, the `coolify` network when it is among several, and otherwise return empty so that `install.sh` stops listing the networks found and asking for `TRAEFIK_NETWORK` in `.env`; a declared `TRAEFIK_NETWORK` SHALL keep precedence.

#### Scenario: Traefik on two unrelated networks
- **WHEN** Traefik is attached to `aaa-simulado` and `proxy` and `TRAEFIK_NETWORK` is not set
- **THEN** the installer stops naming both networks instead of picking `aaa-simulado`

### Requirement: Install and update seed the encryption keys into the database
`ensure_encryption_key` in `hostgator-setup-kit/_common.sh`, called by `install.sh` and `update.sh`, SHALL reuse or generate `NUVEMSHOP_OAUTH_ENCRYPTION_KEY` (hex 32 bytes) and `CPF_ENCRYPTION_KEY` (base64 32 bytes), append a generated value to `.env`, and upsert them into `private.app_secrets` as `nuvemshop_oauth_key` and `cpf_key`, only warning when the upsert fails.

#### Scenario: Update of an installation without a CPF key
- **WHEN** `update.sh` runs on a `.env` without `CPF_ENCRYPTION_KEY`
- **THEN** a key is generated, appended to `.env` and stored in `private.app_secrets` under `cpf_key`

### Requirement: The installer removes its temporary e-mail notice file
`install.sh` SHALL delete the `mktemp` file held in `PENDENCIA_EMAIL` through an `EXIT` trap on success, on failure and when it stops early.

#### Scenario: Installation aborted midway
- **WHEN** `install.sh` exits with an error after the e-mail step
- **THEN** no `tmp.*` file created by that run is left in `/tmp`

### Requirement: The access e-mail warning matches the Supabase kind
`aviso_do_site_url` SHALL print nothing when `SINGLE_SERVER=1` (the kit's own Supabase), print the Supabase dashboard URL-configuration warning with the `SUPABASE_ACCESS_TOKEN` hint when `NEXT_PUBLIC_SUPABASE_URL` is a `*.supabase.co` address or empty, and otherwise print the self-managed `SITE_URL` / `ADDITIONAL_REDIRECT_URLS` instructions without any `sbp_` token request.

#### Scenario: Self-managed Supabase
- **WHEN** `update.sh` runs without `SUPABASE_ACCESS_TOKEN` against `https://supa.example.com`
- **THEN** the warning asks to check `SITE_URL` and `ADDITIONAL_REDIRECT_URLS` and does not mention a cloud token
