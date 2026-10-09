## ADDED Requirements

### Requirement: Per-channel report over a bounded window
`GET /api/v1/metrics/channels` SHALL require `requireRole("agent")`, default the window to the last 30 days ending now, answer 422 `validation_failed` when `from`/`to` are not ISO datetimes with offset, `owner_user_id` is not a UUID, `from` is not before `to`, or the window exceeds 90 days, and SHALL call the `security invoker` function `fn_channel_metrics(p_org, p_from, p_to, p_owner)` (migration `20261007180002_0590_relatorio_por_canal.sql`) on the session client with `p_org` from the active organization, returning one row per `channel_session_id` with conversations, first human response and unanswered conversations, or 500 `internal_error` when the function fails.

#### Scenario: Window too long
- **WHEN** an agent requests a 120-day window
- **THEN** the response is 422 `validation_failed` naming the 90-day maximum

#### Scenario: Agent scope
- **WHEN** an agent without manager role requests the report
- **THEN** the RLS of `conversations` limits the counts to that agent's own conversations

### Requirement: Attendant and channel metrics cut conversations by the window first
`fn_attendant_metrics` and `fn_channel_metrics`, as redefined by migration `20261008021443_0596_recorte_da_janela_nas_metricas.sql`, SHALL evaluate the per-message lateral only for conversations assigned inside `[p_from, p_to)` or with a human outbound message inside it, returning the same numbers as before for the same window.

#### Scenario: One-day window
- **WHEN** the dashboard asks for a one-day window in an organization with years of conversations
- **THEN** only conversations touched in that day are scanned by the lateral over `messages`

### Requirement: Funnel and loss reads are paged and report truncation
`GET /api/v1/metrics/funil` and `GET /api/v1/metrics/lost` SHALL read their row sources in pages of 1000 ordered by time and id, up to 5 pages (5000 rows), counting the total on the first page, and SHALL return `truncado: true` when the window holds more rows than were read.

#### Scenario: More than 5000 losses
- **WHEN** the window has 6000 lost leads
- **THEN** the loss report groups the 5000 most recent and answers `truncado: true`

#### Scenario: Between 1000 and 5000 rows
- **WHEN** the window has 1500 lead activities
- **THEN** the funnel reads all 1500 and answers `truncado: false`
