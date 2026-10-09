## ADDED Requirements

### Requirement: Root path is a public landing page
`app/page.tsx` SHALL redirect a visitor whose `getUser()` returns a user to `/app`, and SHALL otherwise render, without login and with `dynamic = "force-dynamic"`, a page with the brand name, a link to `/login`, and links to `/legal/privacy` (including the anchor `#dados-do-google`) and `/legal/terms`.

#### Scenario: Signed-out visitor
- **WHEN** a browser without session requests `/`
- **THEN** the response is 200 with links to `/legal/privacy` and `/legal/terms` and no redirect to `/login`

#### Scenario: Signed-in user
- **WHEN** a signed-in user requests `/`
- **THEN** the response redirects to `/app`
