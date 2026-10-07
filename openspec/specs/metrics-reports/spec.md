# metrics-reports Specification

## Purpose
Read-only performance metrics and reports for one organization: attendant performance, friction (atrito), funnel and losses under `/api/v1/metrics/*`, and activity, financial and tag reports under `/api/v1/reports/*`, rendered by the `/app/metrics` ("Desempenho") page and the `/app/analise` hub. Aggregation runs in Postgres functions (`fn_attendant_metrics`, `fn_atrito_metrics`, `fn_activity_report`, `fn_relatorio_financeiro`) or in pure helpers under `lib/metrics/` and `lib/reports/`. The underlying records belong to `leads-deals`, `pipelines-kanban`, `inbox-conversations` and `finance`; AI cost dashboards belong to `ai-credentials-budget`.

## Requirements

### Requirement: Attendant metrics over a bounded window
`GET /api/v1/metrics/attendants` SHALL require `requireRole("agent")`, SHALL default the window to the last 30 days ending now, SHALL return 422 `validation_failed` when `from`/`to` are not ISO datetimes with offset, `owner_user_id` is not a UUID or `from` is not before `to`, and SHALL call `fn_attendant_metrics(p_org, p_from, p_to, p_owner)` on the session client with `p_org` from the active organization.

#### Scenario: Inverted window
- **WHEN** an agent calls `GET /api/v1/metrics/attendants?from=2026-10-02T00:00:00Z&to=2026-10-01T00:00:00Z`
- **THEN** the response is 422 `validation_failed`

#### Scenario: Viewer is refused
- **WHEN** a user with role `viewer` calls the endpoint
- **THEN** `requireRole` refuses the request and no RPC runs

### Requirement: Friction metrics use the organization's abandonment threshold
`GET /api/v1/metrics/atrito` SHALL require `requireRole("agent")`, SHALL read `organizations.settings.atrito.abandono_horas` through the admin client filtered by the active `organization_id` (falling back to `ABANDONO_HORAS_DEFAULT` = 72 when absent or invalid), and SHALL call `fn_atrito_metrics(p_org, p_from, p_to, p_abandono_horas)` on the session client, returning `window`, `escopo`, `regua`, `pares` and raw `componentes`.

#### Scenario: Organization never configured the threshold
- **WHEN** `organizations.settings.atrito` is absent
- **THEN** the response carries `regua.abandono_horas = 72` and `regua.default = 72`

### Requirement: Changing the abandonment threshold is a manager write
`PATCH /api/v1/metrics/atrito` SHALL call `requireSupportWrite` and `requireRole("manager")`, SHALL accept only an integer `abandono_horas` between 1 and 2160 (422 `validation_failed` otherwise), SHALL merge it into `organizations.settings.atrito` without dropping other keys, and SHALL audit `metrics.atrito_regua_changed`.

#### Scenario: Out-of-range threshold
- **WHEN** a manager sends `{ "abandono_horas": 5000 }`
- **THEN** the response is 422 `validation_failed` and `organizations.settings` is unchanged

### Requirement: Funnel report scoped by organization
`GET /api/v1/metrics/funil` SHALL require `requireRole("agent")`, SHALL default to the last 30 days, and SHALL read `crm_lead_activities`, `crm_leads`, `crm_stages` and `crm_pipelines` filtered by `organization_id` of the active organization plus `fn_atrito_metrics`, returning 500 `internal_error` naming the failed source when any read fails.

#### Scenario: One source query fails
- **WHEN** the `crm_stages` read returns an error
- **THEN** the response is 500 `internal_error` and no partial funnel is returned

### Requirement: Loss report grouped by reason, stage and currency
`GET /api/v1/metrics/lost` SHALL require `requireRole("manager")`, SHALL read `crm_leads` with `status = 'lost'` and `closed_at` inside the window through the admin client filtered by the active `organization_id`, and SHALL group counts and `value_cents` per `currency` via `agruparPerdas`, never summing different currencies together.

#### Scenario: Losses in two currencies
- **WHEN** the window has lost leads in `BRL` and `USD`
- **THEN** the report returns one value bucket per currency

### Requirement: Activity report window and timezone
`GET /api/v1/reports/activities` SHALL require `requireRole("viewer")`, SHALL accept `days` between 1 and 90 (default 7) and an IANA `tz` validated by `fusoValido` (default `UTC`), and SHALL call `fn_activity_report` on the session client, returning 422 `validation_failed` for invalid query values.

#### Scenario: Ninety-one days requested
- **WHEN** `GET /api/v1/reports/activities?days=91` is called
- **THEN** the response is 422 `validation_failed`

### Requirement: Financial report capped at 400 days
`GET /api/v1/reports/financeiro` SHALL require `requireRole("viewer")`, SHALL default `de` to the first day of the current month and `ate` to today, SHALL return 422 `validation_failed` when `de` is after `ate` or the span exceeds 400 days, and SHALL call `fn_relatorio_financeiro` with `p_org` from the active organization.

#### Scenario: Span longer than 400 days
- **WHEN** `de=2025-01-01&ate=2026-06-01` is requested
- **THEN** the response is 422 `validation_failed` with "O período não pode passar de 400 dias."

### Requirement: Tag report bounded by days and tag count
`GET /api/v1/reports/tags` SHALL require `requireRole("viewer")`, SHALL return 422 `validation_failed` when the window exceeds 90 days, `de` is after `ate` or more than 100 tags are requested, and SHALL read `conversations` with an explicit `organization_id` filter on the session client.

#### Scenario: Too many tags
- **WHEN** 101 tags are passed in `tags`
- **THEN** the response is 422 `validation_failed`

### Requirement: Aggregation failures surface as internal errors
Every `/api/v1/metrics/*` and `/api/v1/reports/*` GET handler SHALL return 500 `internal_error` with its `requestId` when its Postgres function or table read returns an error, instead of returning an empty payload.

#### Scenario: RPC error
- **WHEN** `fn_relatorio_financeiro` raises an error
- **THEN** `GET /api/v1/reports/financeiro` responds 500 `internal_error`

### Requirement: Comparison across attendants is shown to managers only
The `/app/metrics` page SHALL compute `canCompare` as the active organization role being at least `manager` by `ROLE_RANK`, describing a 30-day view of friction, funnel and per-attendant performance, while agents see their own scope.

#### Scenario: Agent opens Desempenho
- **WHEN** a user with role `agent` opens `/app/metrics`
- **THEN** the page header describes "seu funil e sua performance" and the per-attendant comparison is not offered
