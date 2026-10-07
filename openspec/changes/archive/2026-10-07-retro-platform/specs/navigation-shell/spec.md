## Purpose
How a signed-in person reaches screens. Request routing happens in `proxy.ts` (public-path bypass via `lib/auth/public-paths.ts`, session check, `/login?next=` redirect, `/admin` gate). The authenticated shell is `app/app/layout.tsx` with `app/app/_components/AppShell.tsx`, which redirects by account and organization state. The menu, hubs and the ⌘K palette are generated from one catalog, `NAV_CATALOG` and `NAV_GROUPS` in `lib/navigation/catalogo.ts`, projected by `lib/navigation/registry.ts` (`sidebarGroups`, `hubSections`, `searchable`) and filtered by role, optional installation module, organization capability and the "interface" choice in `lib/navigation/interface.ts`. The interface choice is stored in `organizations.interface_settings` (company-wide, `app/actions/settings/atualizarInterfaceDaEmpresa.ts`) and `user_organizations.interface_settings` (per member, `PATCH /api/v1/team/{user_id}/interface`).

## ADDED Requirements

### Requirement: Unauthenticated page requests go to login
For a non-public, non-API path without a valid Supabase user (`getUser()`), `proxy.ts` SHALL redirect to `/login?next=<original path and query>`.

#### Scenario: Deep link while signed out
- **WHEN** a signed-out browser requests `/app/kanban?pipeline=x`
- **THEN** the response redirects to `/login?next=%2Fapp%2Fkanban%3Fpipeline%3Dx`

### Requirement: Shell redirects by account and organization state
`app/app/layout.tsx` SHALL redirect to `/login` without a user, to `/acesso-revogado` when the user has no active organization because their membership was revoked, to `/account-suspended` when the active organization's `status` is not `active`, and to `/onboarding` when `onboarded_at` is null (except in support sessions).

#### Scenario: Suspended organization
- **WHEN** a member of an organization with `status = 'suspended'` opens `/app/inbox`
- **THEN** the response redirects to `/account-suspended`

#### Scenario: Revoked member
- **WHEN** a user whose only membership was revoked opens `/app`
- **THEN** the response redirects to `/acesso-revogado` instead of offering to create an organization

### Requirement: One catalog for menu, hubs and palette
Every navigable destination SHALL be declared once in `NAV_CATALOG` with `href`, `label`, a group among `atendimento`, `crm`, `ia`, `canais`, `analise`, `organizacao`, and optional `minRole`, `modulo` and `capacidade`, from which the sidebar, the hub pages (`/app/crm`, `/app/ai`, `/app/analise`, `/app/settings`) and the ⌘K search are all derived.

#### Scenario: New destination
- **WHEN** an entry is added to `NAV_CATALOG`
- **THEN** it appears in the sidebar group, its hub and the palette without other lists being edited

### Requirement: Role, module and capability filtering
`permitidos()` SHALL show a destination only when the caller is a platform admin or `ROLE_RANK[role] >= ROLE_RANK[minRole ?? "viewer"]`, its `modulo` (if any) is enabled in the installation, and its `capacidade` (if any) is held by the organization.

#### Scenario: Agent does not see manager screens
- **WHEN** a user with role `agent` loads the shell
- **THEN** destinations declared with `minRole: "manager"` are absent from the menu and palette

#### Scenario: Optional module off
- **WHEN** the installation module `crm_b2b` is disabled
- **THEN** destinations with `modulo: "crm_b2b"` are hidden for every role, including admins

### Requirement: Interface presets with essential doors
The interface setting SHALL accept `preset` `completa` or `simplificada` (the latter limited to inbox, agenda, kanban, contacts, tasks and connections) or an explicit `destinos` list, and SHALL always keep the essential doors `/app/settings/profile` and `/app/settings/security`, plus `/app/team` and `/app/settings/tenant` for admins and platform admins.

#### Scenario: Simplified interface
- **WHEN** an organization sets `interface_settings = {"preset":"simplificada"}`
- **THEN** an agent sees only the simplified destinations plus profile and security

### Requirement: Company and member interface intersect
When both the organization and the member have an explicit destination set, `combinarInterfaces()` SHALL show their intersection, falling back to the essential doors when the intersection is empty; `PATCH /api/v1/team/{user_id}/interface` SHALL require role `admin` and answer 400 `validation_error` when the chosen set leaves the member's role with no permitted non-essential destination.

#### Scenario: Member restricted to nothing usable
- **WHEN** an admin saves for an `agent` a destination list containing only manager-only screens
- **THEN** the response is 400 `validation_error` and `user_organizations.interface_settings` is unchanged

### Requirement: Home is the inbox when visible
`homeDaInterface()` SHALL choose `/app/inbox` when visible, else the first visible non-essential destination, else `/app/settings/profile`.

#### Scenario: Inbox hidden by the interface
- **WHEN** a member's interface `destinos` is exactly `["/app/agenda"]`
- **THEN** the home destination is `/app/agenda`
