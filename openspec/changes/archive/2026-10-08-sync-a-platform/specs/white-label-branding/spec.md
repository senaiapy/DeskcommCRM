## ADDED Requirements

### Requirement: Public pages name the resolved installation brand
The public pages `/` (`app/page.tsx`), `/legal/*` (`app/legal/layout.tsx`) and `/signup` (`app/(public)/signup/page.tsx`) SHALL take the product name from `marcaDaSaida(null)` (database above `.env`, never throwing) instead of the `.env`-only `branding()`, while the LGPD PDF keeps carrying no brand.

#### Scenario: Brand saved through the console
- **WHEN** `platform_branding.app_name` is "Acme CRM" while `APP_NAME` is "Old Name"
- **THEN** `/legal/privacy`, `/signup` and the signed-out `/` show "Acme CRM"

#### Scenario: Branding read fails
- **WHEN** reading `platform_branding` fails while a visitor opens `/legal/terms`
- **THEN** the page responds 200 with the fallback brand
