## Purpose
Read-only connector to an organization's external PostgreSQL database (installation module `banco_externo`): admins
register connections in `external_db_connections` with an encrypted password, any member browses schemas and tables live
at `/app/integracao-dados`, and the AI agent reads them through the MCP tools `crm_describe_external_data` and
`crm_query_external_data`. The MCP tool runtime and agent tool permissions are covered by the AI capabilities
(`ai-agents`, `mcp-server`); installation module toggles by the platform module catalog.

## ADDED Requirements

### Requirement: Module gate
Every `/api/v1/external-db/*` handler SHALL answer 404 `not_found` when the installation module `banco_externo` is off (`seModuloDesligado`, `abrirAcesso` motivo `modulo_desligado`).

#### Scenario: Module disabled
- **WHEN** the `banco_externo` module is off and an admin posts to `/api/v1/external-db/connections`
- **THEN** the response is 404 `not_found` and nothing is written

### Requirement: Role split between reading and configuring
`GET /api/v1/external-db/connections`, `GET .../connections/{id}`, `GET .../schemas` and `GET .../tables/{schema}/{tabela}` SHALL require role `viewer`, while `POST /connections`, `PATCH` and `DELETE .../connections/{id}` and `POST .../connections/{id}/test` SHALL require role `admin`, all scoped to the active `organization_id`.

#### Scenario: Agent tries to create a connection
- **WHEN** a user with role `agent` posts a new connection
- **THEN** the response is 403 and no `external_db_connections` row is created

### Requirement: Connection validation and storage
`POST /api/v1/external-db/connections` SHALL validate the body strictly (`port` 1-65535 default 5432, `ssl_mode` in disable/prefer/require/verify-ca/verify-full default `require`, `max_rows` 1-5000 default 200, `max_filters` 0-100 default 20, `max_response_bytes` 4096-1048576 default 30000, `customer_key_column` and `customer_key_kind` both or neither), store the password AES-GCM encrypted, and return 201 with only the safe columns.

#### Scenario: Customer key half-filled
- **WHEN** the body has `customer_key_column` but no `customer_key_kind`
- **THEN** the response is 422 `validation_failed`

#### Scenario: Duplicate label
- **WHEN** an admin creates a connection whose `label` already exists in the organization
- **THEN** the response is 409 `external_db_label_em_uso`

### Requirement: Network destination policy
Creating or updating a connection, and every access, SHALL resolve the host (`validarHostDeBanco`) and refuse loopback, link-local/cloud metadata (169.254/16), CGNAT, multicast and reserved addresses, including when any resolved address is forbidden, while allowing RFC1918 LAN ranges, answering 422 `external_db_destino_bloqueado` (or 422 `validation_failed` when DNS fails).

#### Scenario: Cloud metadata address
- **WHEN** an admin registers host `169.254.169.254`
- **THEN** the response is 422 `external_db_destino_bloqueado`

#### Scenario: LAN database
- **WHEN** the host resolves only to `192.168.0.10`
- **THEN** the connection is accepted

### Requirement: Read-only transaction with timeouts
Every query to the external database SHALL run through `consultar` (lib/external-db/conexao.ts) inside `BEGIN READ ONLY` with local `statement_timeout`, `lock_timeout` and `idle_in_transaction_session_timeout`.

#### Scenario: Write attempt reaches the external database
- **WHEN** any statement executed through `consultar` tries to modify data
- **THEN** PostgreSQL rejects it because the transaction is read-only

### Requirement: Server-built SELECT with closed filter vocabulary
`GET /api/v1/external-db/connections/{id}/tables/{schema}/{tabela}` SHALL answer 404 `not_found` when the table or view is not found by live introspection, build the SELECT on the server with quoted identifiers restricted to existing columns and a closed set of filter operators with bound parameters, cap the page at the connection `max_rows` (never above 5000), and audit `external_db_connection.read`.

#### Scenario: Unknown column requested
- **WHEN** the request asks for a column that does not exist in the table
- **THEN** the response is 422 `validation_failed` and no query is sent

#### Scenario: External database unreachable
- **WHEN** the external server times out during the read
- **THEN** the response is 502 `upstream_unavailable`

### Requirement: Access state errors
Opening a connection for reading SHALL answer 404 `not_found` for a connection of another organization or missing, 409 `external_db_desativada` when `enabled` is false, and 500 `external_db_sem_chave` when the installation encryption key is unavailable.

#### Scenario: Disabled connection
- **WHEN** a viewer lists schemas of a connection with `enabled=false`
- **THEN** the response is 409 `external_db_desativada`

### Requirement: Rate limits
Write routes SHALL be limited to 30 changes per 60 s per organization (`external-db:write:<org>`), and read and test routes SHALL also be rate-limited, answering 429 `rate_limited`.

#### Scenario: Burst of edits
- **WHEN** an admin sends a 31st connection change within 60 seconds
- **THEN** the response is 429 `rate_limited`

### Requirement: Delete closes the pool
`DELETE /api/v1/external-db/connections/{id}` SHALL delete the row filtered by `organization_id` and `id`, close the cached pool (`fecharPool`), audit `external_db_connection.deleted` and return `{ id, deleted: true }`.

#### Scenario: Deleting a connection in use
- **WHEN** an admin deletes a connection that has an open pool
- **THEN** the pool is closed and later reads of that id answer 404

### Requirement: AI tool restricted to the conversation's customer
`crm_query_external_data` SHALL, when the connection has `customer_key_column`/`customer_key_kind` and the turn has a contact, add a server-side `in` filter with the contact's phones or e-mails that the model cannot remove, refuse filters without value (`filtro_sem_valor`) and more filters than `max_filters` (`limite_de_filtros`), and label returned rows as untrusted data.

#### Scenario: Model asks for all orders
- **WHEN** the agent calls `crm_query_external_data` on a connection with `customer_key_kind='phone'` during a conversation
- **THEN** only rows whose key column matches the contact's phone are returned
