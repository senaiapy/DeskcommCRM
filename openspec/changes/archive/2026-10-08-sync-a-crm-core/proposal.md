## Why

The crm-core specs were written against upstream commit `477b77678`. HEAD is 491 upstream commits later and the code drifted: the CPF is now really encrypted at rest (migration 0597) and a contact is saved without CPF instead of being refused, the duplicate scan reads past 1,000 rows, an agent's own-mode deal defaults to the agent, an RLS refusal on `crm_leads` is a 403, the hard-delete advice of an archived pipeline changed, a filtered board computes drag positions against the whole stage, and reopening a cancelled task goes back to pending. One existing scenario ("Hash without ciphertext rejected" answering 500 through the API) is no longer true.

## What Changes

- Spec-only: no product code changes.
- `contacts`: "CPF is never stored in plaintext" and "Duplicate detection" rewritten to the current behavior.
- `leads-deals`: two new requirements (owner default in `own` mode; 42501 on `crm_leads` → 403).
- `pipelines-kanban`: the pipeline DELETE requirement gains the archived-pipeline advice; new drag-position requirement.
- `tasks-activities`: new toggle requirement.
- BREAKING: none.

## Capabilities

### New Capabilities
(none)

### Modified Capabilities
- `agenda-scheduling`: MCP scheduling tools return the local wall-clock time (`quando`, `fim_quando`).
- `contacts`
- `leads-deals`
- `pipelines-kanban`
- `tasks-activities`

## Impact

Only `openspec/specs/*` after archive. No API, schema or runtime change.
