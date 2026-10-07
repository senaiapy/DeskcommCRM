## Purpose
The MCP server exposes the CRM as Model Context Protocol tools over Streamable HTTP at `/api/mcp`, authenticated by organization API tokens, plus a session-authenticated catalog of the tools at `/api/v1/mcp/tools` and the client side that lets an agent call one external MCP server registered by the organization (`lib/mcp/servidor-externo`). Token issuance and revocation are covered by `api-tokens`; the generic REST contract and dual auth by `api-rest-contract`; the audit table by `audit-log`; the shared Redis counters by `rate-limit`; the agent turn that consumes the tools by `ai-agents-runtime`.

## ADDED Requirements

### Requirement: MCP endpoint and transport
`POST`, `GET` and `DELETE /api/mcp` SHALL serve the MCP Streamable HTTP transport from a server named `deskcomm-crm` built per request, set `X-Request-Id` on the response, and answer transport or unexpected failures as a JSON-RPC error `-32603` with HTTP 500.

#### Scenario: Successful call carries a request id
- **WHEN** a valid client sends a `tools/list` request
- **THEN** the response lists the registered tools and has an `X-Request-Id` header

### Requirement: Bearer token authentication
The endpoint SHALL accept only `Authorization: Bearer dsk_...`, resolve it by the SHA-256 hash in `api_tokens`, and reject a missing or malformed header and an unknown, revoked or expired token with JSON-RPC `-32001` and HTTP 401, a failed lookup with `-32603`/500, and SHALL refuse with `-32004`/429 once the failed-attempt limiter for that token trips.

#### Scenario: Revoked token
- **WHEN** a request presents a `dsk_` token whose `revoked_at` is set
- **THEN** the response is HTTP 401 with JSON-RPC error code `-32001`

#### Scenario: Wrong prefix
- **WHEN** the bearer value does not start with `dsk_`
- **THEN** the response is HTTP 401 with JSON-RPC error code `-32001`

### Requirement: Tenant and role come from the token
Every tool call SHALL run with `organization_id` equal to `api_tokens.organization_id` of the presented token, a role equal to the highest valid `role:<role>` scope on the token (default `agent`), and an actor of type `ai_agent` when the token has scope `actor:ai_agent`, otherwise `api_token`.

#### Scenario: Token without role scope
- **WHEN** a token has scopes `["mcp:read"]` only
- **THEN** the call runs with role `agent`

### Requirement: Per-tool scope and role enforcement
Each tool SHALL declare `requiresScope` (`mcp:read` or `mcp:write`) and `requiresRole`, and a call whose token lacks the scope or whose role ranks below the minimum SHALL return an MCP tool error (`isError: true`) produced from `-32002`/403 and SHALL still be audited.

#### Scenario: Read-only token calls a write tool
- **WHEN** a token with only `mcp:read` calls a tool that requires `mcp:write`
- **THEN** the tool result has `isError: true` with "Token missing required scope 'mcp:write'" and an `mcp.tool_called` audit row with `success: false` is written

### Requirement: Rate limits per token, organization and writes
Before scope and role checks, each tool call SHALL consume, in order, a 60-calls-per-60-seconds bucket per token, a 600-per-60-seconds bucket per organization and, for tools of category `write`, a 30-per-60-seconds write bucket per token, refusing with JSON-RPC code `-32004` when any bucket is exhausted.

#### Scenario: Token loop
- **WHEN** one token makes its 61st tool call inside the same 60-second window
- **THEN** the call returns a tool error naming the token limit and the organization bucket is not consumed

### Requirement: Suspended organization only answers privacy tools
A token of an organization that is not operating SHALL authenticate, but every tool not flagged `permiteOrgSuspensa` SHALL fail with `-32002`/403 and code `org_suspended`, so only privacy (LGPD) tools answer until reactivation.

#### Scenario: Suspended organization creates a lead
- **WHEN** a token of a suspended organization calls a lead-creating tool
- **THEN** the tool result is an error stating the organization is suspended and no lead is created

### Requirement: Audit of every tool call
Every tool call, successful or not, SHALL insert an `api_audit_log` row with action `mcp.tool_called`, `resource_type='mcp_tool'`, null `actor_user_id` and `resource_id`, `actor_api_token_id` set, and metadata holding `tool_name`, `duration_ms`, `success` and the arguments with `authorization`, `api_key`, `token`, `password` and `cpf` replaced by `[redacted]` and strings over 500 characters truncated.

#### Scenario: Sensitive argument
- **WHEN** a tool is called with an argument named `cpf`
- **THEN** the audit metadata stores `cpf: "[redacted]"`

#### Scenario: Empty result is not success
- **WHEN** a tool declares an empty result reason for its output
- **THEN** the audit row has `success: false` and `desfecho: "sem_resultado"`

### Requirement: Disabled modules and capabilities are not served
The server SHALL NOT register a tool that belongs to an optional module switched off in the installation or to a capability switched off by the organization, so it is absent from `tools/list`.

#### Scenario: Module off
- **WHEN** an optional module is disabled in the installation
- **THEN** its tools do not appear in the MCP `tools/list` response for any token

### Requirement: Idempotency key validation
When the request carries an `Idempotency-Key` header it SHALL be a UUID, otherwise the endpoint SHALL answer JSON-RPC `-32602` with HTTP 400 before creating the server, and a valid key SHALL be passed to tool handlers in the call context.

#### Scenario: Non-UUID key
- **WHEN** a request sends `Idempotency-Key: abc`
- **THEN** the response is HTTP 400 with JSON-RPC error code `-32602`

### Requirement: Tool catalog for the UI
`GET /api/v1/mcp/tools` SHALL require a logged-in user with an active organization (401 `unauthenticated`, 403 `forbidden_tenant`), return the served tools joined with `TOOL_CATALOG` with each `input_schema` as OpenAPI 3.0 JSON Schema, list in `desligadas_pela_organizacao` the tools the organization switched off, and answer 500 `internal_error` when the catalog and handlers are inconsistent.

#### Scenario: Organization switched a capability off
- **WHEN** an organization disables a capability whose module is on
- **THEN** that tool id is missing from `tools` and present in `desligadas_pela_organizacao`

### Requirement: External MCP server secret storage
The key of the organization's external MCP server SHALL be stored encrypted in `organizations.mcp_externo_chave_encrypted`, `mcp_externo_chave_iv` and `mcp_externo_chave_tag` (only `mcp_externo_chave_last4` shown), while `organizations.settings.mcp_externo` keeps only `{endpoint}`, and an endpoint SHALL pass the outbound URL safety check (`conferirEndpointSeguro`, called by the `definirServidorMcpExterno` settings action) before it is saved.

#### Scenario: Saving an external server
- **WHEN** an organization saves an endpoint and a key for its external MCP server
- **THEN** `settings.mcp_externo` holds only the endpoint and the key exists only as ciphertext columns plus its last four characters

#### Scenario: Blank field removes the registry
- **WHEN** the endpoint or the key is submitted blank
- **THEN** the `mcp_externo` entry is removed from `settings` instead of keeping an empty object
