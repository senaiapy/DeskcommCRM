## ADDED Requirements

### Requirement: The data file is the titular's copy and reaches every country
The export worker SHALL serialize `data.json` from `copiaDoTitular(data, perfil.codigo)` (`lib/lgpd/copia-do-titular.ts`), which removes the team-only sections (`SECOES_DA_EQUIPE`, e.g. `case_chat_messages`), internal database ids, staff names and phones and what the AI typed into tool calls, handoffs and cases, keeps `conversation_notes` (body and date only, without author or attachment) only for country `PT`, and the delivery e-mail SHALL carry a signed link to `data.json` with the same expiry as the PDF for every country, failing the attempt with `signed_url_json_failed` when that link cannot be created.

#### Scenario: Brazilian data subject
- **WHEN** an access request of an organization in `BR` is exported
- **THEN** the e-mail links both `report.pdf` and `data.json`, and `data.json` has no `conversation_notes` section

#### Scenario: Portuguese data subject
- **WHEN** an access request of an organization in `PT` is exported
- **THEN** `data.json` keeps the internal notes about the subject with `body` and `created_at` only
