## Purpose
Interface language for screens, server messages and dates. Portuguese (`pt-BR`) is the source language and the dictionary key; translations live in the `DICIONARIO` map of `lib/i18n/dicionario.ts` (`traduzir(texto, idioma)`), with draft catalogs in `lib/i18n/traducoes/{en,zh-CN}.json`. The language registry and its maturity levels are `REGISTRO_DE_IDIOMAS` in `lib/i18n/registro.ts`; `lib/i18n/idiomas.ts` normalizes codes and parses `Accept-Language`; `lib/i18n/idiomaAnonimo.ts` serves visitors without an account; `lib/i18n/datas.ts` maps a language to date-fns locale and BCP-47 tag; `lib/i18n/IdiomaProvider.tsx` provides it to client components. The user's choice is `auth.users.user_metadata.locale` (written by `app/actions/settings/updateProfile.ts` and `trocarIdioma.ts`), the organization default is `organizations.locale` (written by `updateTenant.ts`), and the resolution chain lives in `lib/auth/server.ts`. Coverage is policed by gates such as `tests/unit/i18n-espanhol-cobre-a-tela.test.ts` and `idioma-da-interface.test.ts`.

## ADDED Requirements

### Requirement: Registry with maturity levels
Only languages whose `nivel` in `REGISTRO_DE_IDIOMAS` is `telas_principais` or `completo` SHALL be offered (`IDIOMAS_VISIVEIS`); today that is `pt-BR` and `es`, while `en` and `zh-CN` stay `em_construcao` and hidden.

#### Scenario: Language selector
- **WHEN** the profile language selector is rendered
- **THEN** it offers `pt-BR` and `es` and not `en` or `zh-CN`

### Requirement: Unknown codes close to the default
`normalizarIdioma()` SHALL return the input only when it is a visible language code and otherwise `IDIOMA_PADRAO = "pt-BR"`.

#### Scenario: Legacy profile value
- **WHEN** a user's `user_metadata.locale` is `en-US`
- **THEN** the interface language resolves to `pt-BR`

### Requirement: Resolution chain person, then organization, then default
For an authenticated user the language SHALL resolve as `user_metadata.locale`, else the support session's locale, else the active organization's `organizations.locale` (chosen by the active-org cookie), else `pt-BR`.

#### Scenario: No personal preference
- **WHEN** a user with no `locale` in metadata belongs to an active organization whose `locale` is `es`
- **THEN** screens and API messages for that user are in Spanish

#### Scenario: Personal preference wins
- **WHEN** the same user sets `locale = "pt-BR"` via the language switch
- **THEN** screens are in Portuguese regardless of the organization default

### Requirement: Visitors follow Accept-Language
For a visitor without an account, `idiomaDoVisitante()` SHALL pick the first visible language that matches the primary subtag of the `Accept-Language` entries ordered by `q`, falling back to `pt-BR`.

#### Scenario: Spanish browser on the login page
- **WHEN** an anonymous request carries `Accept-Language: es-MX,es;q=0.9,en;q=0.8`
- **THEN** the page is rendered in `es`

### Requirement: Portuguese is the key, translation is a lookup
`traduzir(texto, idioma)` SHALL return `texto` unchanged for `pt-BR` and `DICIONARIO[texto][idioma]` otherwise, falling back to the Portuguese text when no entry exists.

#### Scenario: Missing translation
- **WHEN** a Spanish user hits a message with no `DICIONARIO` entry
- **THEN** the Portuguese text is shown instead of a key or an empty string

### Requirement: API messages are translated, codes are not
`/api/v1` handlers SHALL translate `error.message` with the caller's resolved language (`authz.user.idioma`) while keeping `error.code` language-independent.

#### Scenario: Invalid cursor for a Spanish user
- **WHEN** a Spanish-language user calls `GET /api/v1/audit` with a bad cursor
- **THEN** the response is 400 with `error.code = "invalid_cursor"` and a Spanish `error.message`

### Requirement: Language switch is audited
The language switch action `trocarIdioma` SHALL write `user_metadata.locale` and audit `profile.updated` with `metadata.origem = "seletor_do_topo"`; `updateProfile` SHALL store null when the user chooses `auto` (`SEM_PREFERENCIA_DE_IDIOMA`).

#### Scenario: Choosing automatic
- **WHEN** a user saves the profile with locale `auto`
- **THEN** `user_metadata.locale` becomes null and the organization default applies
