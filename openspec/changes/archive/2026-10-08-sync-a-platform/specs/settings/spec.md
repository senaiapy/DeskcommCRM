## MODIFIED Requirements

### Requirement: Optional features page gated by role
`/app/settings/recursos` SHALL redirect to `/403` for users below `manager` who are not the server owner, list the optional features of `RECURSOS_OPCIONAIS` with installation modules taken from `MODULOS_DA_EMPRESA` (which leaves out `MODULOS_SO_DA_INSTALACAO`, today `cobranca`), and allow toggling a feature only when the user's role reaches the role mapped to that feature's `quemDecide` (installation-level features only for the server owner).

#### Scenario: Agent opens optional features
- **WHEN** an `agent` requests `/app/settings/recursos`
- **THEN** the response redirects to `/403`

#### Scenario: Installation-only module is not listed
- **WHEN** an organization admin opens `/app/settings/recursos`
- **THEN** the page lists no entry for the module `cobranca` ("Cobrança dos seus clientes")

## ADDED Requirements

### Requirement: RGPD art. 15 paragraphs declared by the controller
`PATCH /api/v1/settings/art15` SHALL call `requireSupportWrite()` and `requireRole("admin")`, validate the body with `art15SettingsSchema` (`finalidades` and `destinatarios` up to 2000 chars, `prazo_conservacao` up to 500, 422 `validation_failed` otherwise), store blank values as null under `organizations.settings.art15` with a non-destructive merge of the other `settings` keys, audit `org.art15_updated` with the stored paragraphs, and answer 500 without writing when the current `settings` cannot be read; `/app/settings/tenant` SHALL show the form only when `alineasDoArt15Visiveis(organizations.country)` is true (not Brazil).

#### Scenario: Portuguese organization fills the paragraphs
- **WHEN** an admin of an organization with `country = 'PT'` patches `{ "finalidades": "Gestão de clientes", "destinatarios": "", "prazo_conservacao": "5 anos" }`
- **THEN** `settings.art15` holds `finalidades` and `prazo_conservacao` with `destinatarios = null`, other `settings` keys are unchanged, and an audit `org.art15_updated` exists

#### Scenario: Manager tries to declare
- **WHEN** a `manager` calls `PATCH /api/v1/settings/art15`
- **THEN** the response is 403 and `settings.art15` is unchanged

#### Scenario: Brazilian organization
- **WHEN** an admin of an organization with `country = 'BR'` opens `/app/settings/tenant`
- **THEN** no art. 15 form is rendered
