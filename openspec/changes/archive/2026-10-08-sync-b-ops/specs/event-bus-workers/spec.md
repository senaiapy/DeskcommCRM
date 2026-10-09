## ADDED Requirements

### Requirement: The scheduler keeps the cron secret out of the crontab
`docker/scheduler/entrypoint.sh` SHALL write `Authorization: Bearer $INTERNAL_SECRET` to `$CRON_AUTH_DIR/header` (default `/run/deskcomm-cron/header`, directory mode 700, file mode 600) and every generated crontab line SHALL send it with `curl -H @<that file>`, so the secret never appears in the crontab text.

#### Scenario: Crontab inspected
- **WHEN** an operator prints the scheduler's crontab
- **THEN** the lines reference the header file and do not contain the value of `INTERNAL_SECRET`

### Requirement: The message webhook drains only its own follow-up handlers when a worker drains
When `EVENT_LOG_WORKER_DRAINS` is truthy (`1|true|on|yes|sim`; default `true` in `docker-compose.prod.yml` and `docker-compose.local.yml`, empty in `.env.example`), the in-request drain after an inbound message SHALL call `drainEventLog` with `limit: 10` and `escopo` limited to the inbound organization and the follow-up trigger handlers (`FOLLOWUP_GATILHO_RETORNO_HANDLER_KEY`, `FOLLOWUP_GATILHO_LEAD_HANDLER_KEY`), returning each event to `pending` with the scoped keys appended to `consumed_by` (counted as `deixados_ao_worker`, never as a failed attempt) so the worker loop runs the remaining handlers; when the variable is falsy the request drains everything as before.

#### Scenario: Production stack
- **WHEN** a WhatsApp message arrives on an installation running the `worker` service with default settings
- **THEN** the webhook request runs only the follow-up trigger handlers of that organization and the other consumers of `message.received` are run by the worker loop

#### Scenario: Scoped handler fails
- **WHEN** a scoped follow-up handler errors inside the request
- **THEN** the event's `attempts` is not incremented and no backoff is applied
