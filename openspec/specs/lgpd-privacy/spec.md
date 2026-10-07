# lgpd-privacy Specification

## Purpose
Data-subject rights handling (Brazilian LGPD and the per-country profile): the `lgpd_requests` ledger, the admin
review routes under `/api/v1/lgpd`, the manual contact anonymization cascade, the export and redaction workers
consumed from `event_log`, the media deletion queue and the SLA watcher cron. Request intake from the store
webhooks is covered by `nuvemshop`; the event drain that runs the workers is covered by `event-bus-workers`;
the append-only trail is covered by `audit-log`; age-based purging is covered by `data-retention`.

## Requirements

### Requirement: Request ledger with business-day deadline
`createLgpdRequest` in `lib/lgpd/repository.ts` SHALL insert a `lgpd_requests` row with `status = 'received'` and `due_at` computed by `computeDueAt` as the N-th business day of the organization's country calendar (7 days for `data_request`, 15 days for `redact` and `store_redact`, the latter with `emergency = true` and `scope = 'tenant'`).

#### Scenario: Data request received from the store
- **WHEN** the customer-data-request webhook creates a request on a Friday
- **THEN** a `lgpd_requests` row exists with `request_type = 'data_request'`, `status = 'received'` and `due_at` equal to the midnight UTC of the 7th business day, skipping weekends and the country's holidays

#### Scenario: Store uninstall redaction
- **WHEN** the store-redact webhook creates a request
- **THEN** the row has `scope = 'tenant'`, `emergency = true` and a 15-business-day `due_at`

### Requirement: Admin-only request listing and detail
`GET /api/v1/lgpd/requests`, `GET /api/v1/lgpd/requests/{id}` and `GET /api/v1/lgpd/requests/{id}/preview` SHALL require `requireRole("admin")` (platform admin read-only allowed) and SHALL filter `lgpd_requests` by the session's active `organization_id`, never by a body or path value.

#### Scenario: Agent is denied
- **WHEN** a user with role `agent` calls `GET /api/v1/lgpd/requests`
- **THEN** the response is 403 and no row is returned

#### Scenario: Request of another tenant
- **WHEN** an admin requests `GET /api/v1/lgpd/requests/{id}` for an id that belongs to another organization
- **THEN** the response is 404 `not_found`

### Requirement: SLA bucket and paginated listing
`GET /api/v1/lgpd/requests` SHALL accept `status`, `type`, `sla_bucket` (`overdue|critical|warning|ok`), `page` and `limit` (max 100), order rows by `due_at` ascending, attach `sla_bucket` computed by `computeSlaBucket` from the civil day of `due_at`, and return `meta.total`, `meta.page`, `meta.limit` and `meta.has_more`; invalid parameters SHALL return 422 `validation_failed`.

#### Scenario: Invalid limit
- **WHEN** the client sends `limit=500`
- **THEN** the response is 422 `validation_failed`

#### Scenario: Bucket filter
- **WHEN** the client sends `sla_bucket=overdue`
- **THEN** only rows whose computed `sla_bucket` is `overdue` are returned in `data`

### Requirement: Signed export link on detail
`GET /api/v1/lgpd/requests/{id}` SHALL return the request, up to 50 `api_audit_log` entries whose `resource_id` or `metadata.request_id` equals the id, and, when `status = 'completed'` and `result.pdf_path` is set, a `signed_pdf_url` valid for 72 hours on the private bucket `lgpd-exports`.

#### Scenario: Completed export
- **WHEN** an admin opens a completed data request
- **THEN** the body contains `signed_pdf_url` pointing to `lgpd-exports` and `audit_trail` ordered by `created_at`

### Requirement: Masked dry-run preview
`GET /api/v1/lgpd/requests/{id}/preview` SHALL return per-category `counts` and at most 10 sample rows per category without writing to the database, masking e-mail and phone with `maskEmail`/`maskPhone`, replacing message bodies with `[masked]` and exposing only `cpf_present`, never the CPF value.

#### Scenario: Preview hides CPF
- **WHEN** an admin previews a request whose contact has a CPF
- **THEN** the response contains `cpf_present: true` and no CPF digits

### Requirement: Manual approval re-emits the canonical event
`POST /api/v1/lgpd/requests/{id}/approve` SHALL require `requireSupportWrite`, `requireRole("admin")`, an `Idempotency-Key` header (422 `missing_idempotency_key` when absent) and `approved_reason` of 10-500 chars; only a request in `received` status SHALL be approved (409 `conflict` otherwise), emitting `lgpd.data_request_received` or `lgpd.redact_received` into `event_log`, setting `status = 'processing'` and auditing `lgpd.manually_approved`.

#### Scenario: Approve a received redaction
- **WHEN** an admin approves a `redact` request in `received` with a valid key
- **THEN** an `event_log` row `lgpd.redact_received` exists, the request is `processing` and the response is 200 `{request_id, status: "processing"}`

#### Scenario: Replay with the same key
- **WHEN** the same `Idempotency-Key` is sent again for the same endpoint
- **THEN** the cached `idempotency_keys.response_body` is returned and no second event is emitted

#### Scenario: Already processing
- **WHEN** the request status is `processing`
- **THEN** the response is 409 `conflict`

### Requirement: Manual anonymization with resumable cascade
`POST /api/v1/lgpd/anonymize` SHALL require `requireSupportWrite`, a body `{contact_id, justification(10-1000)}` and `requireRole("admin")` in the contact's organization (read through RLS), call `fn_lgpd_anonymize_contact` exactly once per contact, complete the lead and activity redaction through `completarRedacaoDoContato`, emit `contact.anonymized`, and return `action` = `anonymized`, `resumed` or `already_anonymized`.

#### Scenario: First anonymization
- **WHEN** an admin anonymizes a non-anonymized contact
- **THEN** the response `action` is `anonymized` and an audit row `lgpd.anonymize_executed` lists the tables actually redacted

#### Scenario: Resume after interruption
- **WHEN** the contact is already `is_anonymized` but a lead title is still unredacted
- **THEN** the residue is redacted, `action` is `resumed` and the audit action is `lgpd.anonymize_catchup`

#### Scenario: Concurrent update
- **WHEN** `fn_lgpd_anonymize_contact` raises SQLSTATE `40001`
- **THEN** the response is 409 `state_conflict`

### Requirement: Anonymization is irreversible
`PATCH /api/v1/contacts/{id}` SHALL reject any update to a contact whose `is_anonymized` is true with 403 `lgpd_anonymization_irreversible`.

#### Scenario: Editing an anonymized contact
- **WHEN** an admin sends a PATCH with a new `name` for an anonymized contact
- **THEN** the response is 403 `lgpd_anonymization_irreversible` and the row is unchanged

### Requirement: Export worker builds and delivers the report
The handler `lgpd-export-worker.v1` SHALL consume `lgpd.data_request_received`, skip requests already `completed` or of another type, fail the request with `error_message = 'max_attempts_exceeded'` after 3 attempts, upload `{org}/{request}/data.json` and `{org}/{request}/report.pdf` (rendered with `@react-pdf/renderer`) to bucket `lgpd-exports`, e-mail a signed link valid for `LGPD_EXPORT_EXPIRES_HOURS` (default 72) and mark the request `completed` with `result.sha256` and `result.delivered_to_hash`.

#### Scenario: Successful delivery
- **WHEN** the export completes and the e-mail is sent
- **THEN** the request is `completed`, `lgpd.export_generated` and `lgpd.export_delivered` are emitted, and the recipient is stored only as a SHA-256 hash

#### Scenario: E-mail send failure
- **WHEN** the e-mail provider rejects the send
- **THEN** the request returns to `status = 'received'` with `error_message = 'email_send_failed'` and the handler result is `error`

### Requirement: Redaction worker by scope
The handler `lgpd-redact-worker.v1` SHALL consume `lgpd.redact_received`; for `scope = 'contact'` it SHALL resolve the contact (by `contact_id` or `external_customer_id`), run `fn_lgpd_cascade_redact_contact` through `cascadeRedactContact`, enqueue media paths in `storage_redaction_queue` and mark the request `completed` with `cascaded_to`; for `scope = 'tenant'` it SHALL redact contacts in checkpointed batches and set `organizations.status = 'redacted'`.

#### Scenario: Contact redacted
- **WHEN** a contact-scope redaction runs for an existing contact
- **THEN** the request is `completed`, `lgpd.redact_applied` is emitted and the audit action is `lgpd.redact_completed`

#### Scenario: Already anonymized
- **WHEN** the contact was anonymized earlier
- **THEN** the request is `completed` with `result.already_anonymized = true` and audit `lgpd.redact_skipped_already_anonymized`

### Requirement: Media deletion queue drain
`GET /api/v1/cron/storage-redaction` SHALL authorize with `autorizaCron` (403 `forbidden` otherwise) and drain up to `limit` (default 50, max 200) `storage_redaction_queue` rows in `pending`, moving each to `deleted`, to `skipped` with `error_message = 'object_not_found'` when the object is already gone, or to `failed` after 3 attempts.

#### Scenario: Object already removed
- **WHEN** storage reports the object as not found
- **THEN** the queue row becomes `skipped` with `error_message = 'object_not_found'`

### Requirement: SLA watcher alarms the DPO
`GET /api/v1/cron/lgpd-sla-watcher` SHALL authorize with `autorizaCron`, scan up to 500 `lgpd_requests` not in `completed`/`failed` whose `received_at` is older than 5 days (`data_request`) or 10 days (`redact`, `store_redact`), and call `triggerSlaAlarm`, which de-duplicates through `request_payload.last_alarm_at` for 24 hours and writes audit `lgpd.sla_alarm_triggered`.

#### Scenario: Alarm de-duplicated
- **WHEN** the watcher runs twice within 24 hours for the same overdue request
- **THEN** the second run counts it in `deduped` and sends no new alarm

#### Scenario: Missing cron secret
- **WHEN** the call has no `Authorization: Bearer` or `x-cron-secret` header
- **THEN** the response is 403 `forbidden`
