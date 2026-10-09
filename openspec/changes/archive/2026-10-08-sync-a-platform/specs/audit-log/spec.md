## ADDED Requirements

### Requirement: Paged CSV export of the tenant audit
`GET /api/v1/audit/export` SHALL require `requireRole("manager")` (platform admins read-only), accept the filters of `GET /api/v1/audit` while ignoring `limit`, answer 422 `validation_failed` for an invalid query, read `api_audit_log` of the active organization in pages of 1000 rows ordered by `created_at` then `id` descending until 10,000 rows, an empty page or the exact `count` is reached, and respond 200 `text/csv` with `Content-Disposition: attachment; filename="audit-<date>.csv"` and `X-Request-Id`.

#### Scenario: More than one thousand entries
- **WHEN** an org admin exports a window holding 2,500 audit rows
- **THEN** the CSV has 2,500 data lines after the header instead of stopping at 1,000

#### Scenario: Window larger than the ceiling
- **WHEN** the window holds 12,000 audit rows
- **THEN** the CSV has exactly 10,000 data lines, newest first

#### Scenario: Viewer exports
- **WHEN** a `viewer` calls `GET /api/v1/audit/export`
- **THEN** the response is 403 and no CSV is produced
