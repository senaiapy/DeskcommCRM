# outbound-webhooks Specification

## Purpose
Outbound webhooks are HTTP POST deliveries from the CRM to a tenant-configured URL. There is no standalone subscription table: an outbound webhook is the `call_webhook` action of an automation rule, fired by the automation engine when the rule's trigger event is drained from `event_log`, and its delivery log is the `actions_result` of `automation_rule_runs`. Key paths: UI `app/app/webhooks` (RuleEditor/ActionConfigForm, ActivityTab with Resend); executor `lib/automation/actions/call-webhook.ts`; SSRF guards `lib/automation/outbound-url.ts`, `lib/automation/outbound-ip.ts`; secret encryption `lib/webhooks/secrets.ts`; resend `app/api/v1/automation-rules/runs/[runId]/resend` (POST); action schema in `lib/schemas/webhooks.ts`; receiver guide `docs/integracao/webhooks-de-saida.md`.

## Requirements

### Requirement: Webhook action configuration
A `call_webhook` action SHALL be configured inside `automation_rules.actions` with `config.url` (URL, max 2000 chars), optional write-only `config.secret` (max 200, stored as `config.secret_enc`) and optional `config.include_owner`, and an action without a URL SHALL produce a `failed` result with error `missing_url`.

#### Scenario: Missing URL at execution
- **WHEN** the engine executes a `call_webhook` action whose config has no `url`
- **THEN** the action result is `{ type: "call_webhook", status: "failed", error: "missing_url" }` and no HTTP request is made

### Requirement: Anti-SSRF egress guard
Before any request the executor SHALL validate the URL with `assertSafeOutboundUrl` (only `http`/`https`, `https` required when `NODE_ENV=production`, no IPv6 literal, no private host) and `assertDestinoResolvidoSeguro` (resolved IPs must not be private/special), and SHALL send with `redirect: "manual"` so a 3xx is recorded as `redirect_not_followed` and never followed.

#### Scenario: Plain http in production
- **WHEN** a rule posts to `http://example.com/hook` on a production build
- **THEN** the action result is `failed` with error `unsafe_url:https_required` and no request leaves the server

#### Scenario: Hostname resolving to a private IP
- **WHEN** the URL's hostname resolves to `169.254.169.254`
- **THEN** the action result is `failed` with error `unsafe_url:private_ip`

### Requirement: Envelope with public projections only
The request body SHALL be JSON `{ event, occurred_at, happened_at, delivery_id, data }` where `data` merges the event payload with lead and contact objects projected to `LEAD_PUBLIC_FIELDS` and `CONTACT_PUBLIC_FIELDS`, strips `from_user_id`/`to_user_id`/`from_agent_id`/`to_agent_id` from `lead.assigned`, and includes `owner` only when `config.include_owner === true`.

#### Scenario: Lead without owner opt-in
- **WHEN** a `lead.assigned` rule without `include_owner` delivers
- **THEN** the body contains no `owner` key, no `organization_id` and none of the four internal assignment ids

### Requirement: Delivery headers and signing
Every attempt SHALL carry `X-Deskcomm-Event`, `X-Webhook-Delivery`, `X-Webhook-Attempt` and `X-Webhook-Timestamp`, and when a secret is available also `X-Webhook-Signature: t=<ts>,v1=<hex HMAC-SHA256(secret, "<ts>.<delivery>.<body>")>` and the legacy `X-Deskcomm-Signature` (hex HMAC-SHA256 of the body); if `secret_enc` cannot be decrypted the delivery SHALL be sent unsigned.

#### Scenario: Signed delivery
- **WHEN** a rule with a stored secret delivers
- **THEN** the receiver gets `X-Webhook-Signature` whose `v1` equals HMAC-SHA256 over `"<t>.<X-Webhook-Delivery>.<raw body>"` with that secret

### Requirement: Deterministic delivery id
`delivery_id` / `X-Webhook-Delivery` SHALL be a UUID v5 (fixed namespace `ce410f0a-ca5e-4a06-89a0-bfe57222d74f`) of event id, rule id, action index in the full `rule.actions` list and a secret-free fingerprint of that list, so retries and resends of the same delivery reuse the id.

#### Scenario: Resend keeps the id
- **WHEN** a delivery is resent via `POST /api/v1/automation-rules/runs/{runId}/resend` without changes to the rule's actions
- **THEN** the resent request carries the same `X-Webhook-Delivery` as the original

### Requirement: Retry with fixed backoff
The executor SHALL make up to 3 attempts per execution with delays of 1 s and 5 s and a 10 s timeout per attempt, treating network errors and non-2xx statuses as failures (`http_<status>`), and stop at the first 2xx.

#### Scenario: Receiver returns 500 three times
- **WHEN** the receiver answers 500 to every attempt
- **THEN** the action result is `failed` with error `http_500` and `detail.attempts = 3`

### Requirement: Delivery log in automation_rule_runs
Each delivery outcome SHALL be persisted in `automation_rule_runs.actions_result` with `detail.delivery_id`, `detail.attempt` (absolute number of the last attempt) and `detail.response_status`, inside a run whose `status` is `success`, `partial` or `failed`.

#### Scenario: Successful delivery logged
- **WHEN** the receiver answers 200 on the first attempt
- **THEN** the run's `actions_result` contains `{ type: "call_webhook", status: "success", detail: { response_status: 200, attempt: 1, delivery_id } }`

### Requirement: Manual resend continues attempt numbering
The resend endpoint SHALL re-execute the rule's current `call_webhook` actions with the first attempt number set to one more than the highest attempt already recorded for that delivery across the runs of the same rule and event, tag each result with `resent_from_run_id`, and insert a new run returning 201.

#### Scenario: Resend after three failed attempts
- **WHEN** a delivery that already recorded `attempt = 3` is resent
- **THEN** the first resend request carries `X-Webhook-Attempt: 4` and the new run's results include `detail.resent_from_run_id`
