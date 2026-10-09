## ADDED Requirements

### Requirement: External data tools disappear when the module is off
The MCP catalog entries `crm_describe_external_data` and `crm_query_external_data` SHALL declare `modulo: "banco_externo"`, so the MCP server and the agent do not receive them while the installation module `banco_externo` is off, and a call that still reaches the tool with the module off SHALL answer `modulo_desligado` telling the operator to turn the module on in Admin › Sistema.

#### Scenario: Module switched off
- **WHEN** the `banco_externo` module is off and an MCP client lists tools
- **THEN** neither external data tool is in the `tools/list` response

### Requirement: Table names resolve case-insensitively with an exact-name preference
When the requested table is not found as written, `crm_query_external_data` SHALL search the external catalog by lower-cased name, prefer the exact name in schema `public`, then any match in `public`, then the exact name in another schema, then the first match, and SHALL use the resolved real name for the column check and the query.

#### Scenario: ORM table with capital letter
- **WHEN** the agent asks for table `pedido` and the external database has only `public."Pedido"`
- **THEN** the query reads `public."Pedido"` instead of answering `tabela_nao_encontrada`
