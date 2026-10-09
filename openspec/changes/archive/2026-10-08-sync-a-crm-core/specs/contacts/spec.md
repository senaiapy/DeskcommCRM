## MODIFIED Requirements

### Requirement: CPF is never stored in plaintext
Contact create (`POST /api/v1/contacts`), update (`PATCH /api/v1/contacts/{id}`) and CSV import (`POST /api/v1/contacts/import`) SHALL store the CPF only as the pair `cpf_hash` (sha256 hex of the 11 normalized digits) and `cpf_encrypted` (pgcrypto aes256 from the `encrypt_cpf` RPC of migration 0597, key from GUC `app.cpf_key` or `private.app_secrets` row `cpf_key`) produced together by `camposCpfParaGravar` in `lib/contacts/cpf.ts`, writing neither column when encryption is unavailable, and the check `contacts_cpf_consistency` SHALL require both columns to be null or non-null together.

#### Scenario: Hash without ciphertext rejected
- **WHEN** a row is written to `contacts` with `cpf_hash` set and `cpf_encrypted` null
- **THEN** the write violates `contacts_cpf_consistency`

#### Scenario: Encryption unavailable
- **WHEN** a contact with a CPF is created while `encrypt_cpf` raises `CPF_ENCRYPTION_KEY ausente` or does not exist
- **THEN** the response is 201 and the stored contact has `cpf_hash` and `cpf_encrypted` both null

#### Scenario: Contact with CPF saved
- **WHEN** a contact with a valid CPF is created while the CPF key is seeded
- **THEN** the row has both `cpf_hash` and `cpf_encrypted` set and `?search=<the 11 digits>` finds it

#### Scenario: Decrypt request below manager
- **WHEN** a user with role below `manager` calls `GET /api/v1/contacts/{id}` with header `X-Decrypt-Purpose`
- **THEN** the response has `cpf_decrypted = null` and `cpf_decrypt_denied = true`

#### Scenario: Decrypt by a manager is audited
- **WHEN** a `manager` calls `GET /api/v1/contacts/{id}` with header `X-Decrypt-Purpose` for a contact with CPF
- **THEN** `decrypt_cpf` writes an `api_audit_log` row `contact.cpf_decrypted` before returning the plaintext in `cpf_decrypted`

### Requirement: Duplicate detection
`GET /api/v1/contacts/duplicates` SHALL scan at most 2000 live (`is_merged_into IS NULL`), non-anonymized contacts of the active organization, reading them in pages of 1000 ordered by `created_at` then `id` and deciding completeness from the exact `count` (or an empty page), and return duplicate groups with `chave`, `motivos`, `principal_sugerido` and `contatos`.

#### Scenario: Scan truncated
- **WHEN** the organization has more than 2000 live contacts
- **THEN** `meta.varreu_tudo` is false and `meta.contatos_varridos` is 2000

#### Scenario: More than one page scanned fully
- **WHEN** the organization has 1500 live contacts
- **THEN** `meta.varreu_tudo` is true and `meta.contatos_varridos` is 1500

#### Scenario: Unauthenticated
- **WHEN** there is no session
- **THEN** the response is 401 `unauthenticated`
