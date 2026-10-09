## ADDED Requirements

### Requirement: MCP scheduling tools return the local wall-clock time
`crm_list_appointments` SHALL add `quando` and `fim_quando` (a local wall-clock label built by `rotuloLocal` in the appointment's `time_zone`) to each appointment, and `crm_book_appointment`, `crm_find_and_book_appointment`, `crm_reschedule_appointment`, `crm_cancel_appointment` and `crm_confirm_appointment` SHALL return the same two labels on the returned `compromisso`, while keeping the stored instants (`starts_at`, `ends_at`) unchanged.

#### Scenario: The AI reads the time the customer will see
- **WHEN** an appointment stored as `2026-10-08T01:30:00Z` in `America/Sao_Paulo` is booked through MCP
- **THEN** the tool result carries `starts_at` as stored and `quando` with the São Paulo wall-clock time (7 Oct, 22:30), so the agent does not quote the UTC hour
