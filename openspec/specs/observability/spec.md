# observability Specification

## Purpose
Error telemetry, structured logs and request correlation in DeskcommCRM: Sentry initialisation for server, edge and browser with an opt-out DSN, restricted data collection and URL/PII scrubbing hooks, the zero-dependency JSON logger in `lib/logger.ts`, and the `X-Request-Id` header added by `proxy.ts` and the API wrappers. Audit trail rows are covered by `audit-log`; the dependency probe by `health`; the LGPD SLA alarm that also reports to Sentry by `lgpd-privacy`.

## Requirements

### Requirement: Sentry DSN resolution with an explicit off switch
`resolveSentryDsn` in `lib/sentry/dsn.ts` SHALL disable Sentry when `SENTRY_DSN` is `off`, `false` or `0` (case-insensitive for `off`), SHALL fall back to the built-in community DSN `DEFAULT_SENTRY_DSN` when `SENTRY_DSN` is empty, and SHALL otherwise use the configured value.

#### Scenario: Operator opts out
- **WHEN** the server starts with `SENTRY_DSN=off`
- **THEN** `Sentry.init` receives no DSN and the boot log says telemetry is disabled

#### Scenario: Fresh install without the variable
- **WHEN** `SENTRY_DSN` is empty
- **THEN** errors go to the community DSN and the boot log says anonymised error reports are active and how to disable them

### Requirement: Community DSN receives errors only
When the resolved DSN is the community DSN, `sentry.server.config.ts`, `sentry.edge.config.ts` and `instrumentation-client.ts` SHALL set `tracesSampleRate` to 0, the client SHALL set `replaysSessionSampleRate` to 0 and drop the `BrowserSession` and `BrowserTracing` integrations, while a self-configured DSN SHALL get `tracesSampleRate` 1.

#### Scenario: Community DSN in the browser
- **WHEN** the browser initialises Sentry with the community DSN
- **THEN** no performance traces and no session replays are sampled, and only error replays remain

### Requirement: Runtime-specific initialisation and request error capture
`instrumentation.ts` SHALL import `sentry.server.config` when `NEXT_RUNTIME` is `nodejs` and `sentry.edge.config` when it is `edge`, and SHALL export `onRequestError = Sentry.captureRequestError`; `next.config.ts` SHALL wrap the config with `withSentryConfig` using `tunnelRoute: "/monitoring"`.

#### Scenario: Unhandled error in a route handler
- **WHEN** a Node route handler throws
- **THEN** Next calls `onRequestError` and the error is captured by Sentry with the configured privacy options

### Requirement: Restricted data collection
`opcoesDePrivacidade` in `lib/sentry/privacidade.ts` SHALL configure `dataCollection` with `userInfo: false`, `cookies: false`, request and response headers matching `forwarded`, `-ip`, `remote-`, `via` or `-user` denied, `httpBodies: []`, AI inputs and outputs off, `databaseQueryData: false`, `queues: false` and `stackFrameVariables: false`, and all three Sentry configs SHALL spread it.

#### Scenario: Request with a client IP header
- **WHEN** a captured request carries `x-forwarded-for`
- **THEN** that header is not sent to Sentry

### Requirement: Scrubbing hooks on events, spans and breadcrumbs
`sentryScrubHooks` in `lib/sentry/scrub.ts` SHALL provide `beforeSend` (scrubbing event URLs, `message` and every `exception.values[].value` with `scrubUrl`), `beforeSendSpan` (scrubbing span `name` and attributes) and `beforeBreadcrumb` (scrubbing `message` and `data.url`, `data.from`, `data.to`), and these hooks SHALL be part of `opcoesDePrivacidade`.

#### Scenario: Error message contains a tokenised URL
- **WHEN** an exception message includes a URL whose path carries a credential
- **THEN** the event sent to Sentry has that path segment replaced by `[TOKEN]`

### Requirement: Personal data patterns are masked
`scrubMessage` SHALL replace e-mail addresses with `[EMAIL]`, CPF-shaped numbers with `[CPF]`, phone numbers with `[PHONE]` and `apikey_` tokens with `[CHAVE]`, and `isSensitiveHeader` SHALL match header names containing `authorization`, `cookie`, `api-key`, `token`, `secret`, `password`, `credential`, `forwarded`, `-ip`, `remote-` or exactly `via`.

#### Scenario: Log text with an e-mail
- **WHEN** `scrubMessage("contato joao@example.com")` runs
- **THEN** the result is `contato [EMAIL]`

### Requirement: Structured JSON logger
`lib/logger.ts` SHALL write one JSON line per call with `level`, `msg`, `ts` (ISO-8601) and the context fields spread at the top level, using `console.log` for `info`, `console.warn` for `warn`, `console.error` for `error`, and emitting `debug` only when `NODE_ENV` is `development`.

#### Scenario: Debug in production
- **WHEN** `logger.debug(...)` is called with `NODE_ENV=production`
- **THEN** nothing is written

### Requirement: Every API wrapper response carries X-Request-Id
`ok()`, `fail()` and `noContent()` in `lib/api/wrappers.ts` SHALL set the `X-Request-Id` response header to the caller's `requestId` or, when none is given, to a fresh `randomUUID()`.

#### Scenario: Handler without an explicit request id
- **WHEN** a handler returns `fail("not_found", ..., 404)` without `requestId`
- **THEN** the response still has an `X-Request-Id` header with a UUID

### Requirement: The request proxy propagates or mints a request id
`proxy.ts` SHALL set the `x-request-id` response header to the incoming `x-request-id` request header when present, or to `crypto.randomUUID()` otherwise, for every path matched by its `matcher` (all paths except `_next/static`, `_next/image`, `favicon.ico`, `robots.txt`, `sitemap.xml` and static asset extensions).

#### Scenario: Caller supplies a correlation id
- **WHEN** a page request arrives with `x-request-id: abc-123`
- **THEN** the proxy response carries `x-request-id: abc-123`

### Requirement: Agent jobs record queue wait and wall time
`recordRunMetrics` SHALL insert into `metrics`, for every finished agent job with `claim_acquired_at`, `run_queue_wait_ms` (claim time minus `created_at`, never negative) and `run_wall_ms` (database clock from the claim to the close), even when the job made no LLM call, besides the token and cost metrics written only when it made calls.

#### Scenario: Job with no model call
- **WHEN** an agent job finishes without any `llm_calls` row
- **THEN** `metrics` still receives `run_queue_wait_ms` and `run_wall_ms` for it
