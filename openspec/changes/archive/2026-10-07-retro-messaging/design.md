## Context

Retro-spec of the messaging layer of DeskcommCRM as of 2026-10-07. Seed material: `CLAUDE.md` (WAHA section), `docs/specs/03-spec-whatsapp-waha.md`, `13-spec-governanca-atendimento.md`, `21-spec-conversa-pessoal-sai-da-operacao.md`, `22-spec-zernio-desvincular-perfil-e-chave-dupla.md`, `docs/webhooks/`. The code is the authority.

## How the evidence was gathered

- Each capability was traced from its webhooks (`app/api/v1/webhooks/{waha,meta,channel,in}/[token]`), API routes, crons (`app/api/v1/cron/*`, scheduled in `docker/scheduler/entrypoint.sh`) and event_log consumers (`lib/event-log/register-handlers.ts`) into `lib/**` and `supabase/baseline.sql`.
- Requirements name the concrete table, constraint, header, env var or error code that was read; claims that could not be found in code were dropped (e.g. a 5-second campaign interval, an outbound subscription table).
- The opt-out scenarios were executed against a scratchpad copy of `lib/opt-out/deteccao.ts` (six phrases, all matching the spec). Everything else is read, not re-tested.
- Specs were written by helper agents per domain and spot-checked (e.g. `call-webhook.ts:31-32`, `webhooks/in/[token]/route.ts:58`).

## Inconsistencies found (docs vs code)

1. **"Campanha 1 msg/5s"** (`CLAUDE.md`) has no constant in code. The campaign cron runs every minute and sends at most one message per number per run, through channel pacing (1200 ms throttle, 800 ms jitter, daily cap) plus the optional `intervalo_segundos`.
2. **"Webhooks: HMAC SHA512"** holds only for WAHA. Meta (`X-Hub-Signature-256`), Zernio, inbound lead capture and outbound `call_webhook` all use HMAC-SHA256. WAHA events without a signature are accepted unless `WAHA_WEBHOOK_REQUIRE_SIGNATURE` is on, and sessions are created with an empty per-session secret, so the default protection is the network (Caddy).
3. **No outbound-webhook subscription system.** There is no subscriptions API, table or delivery worker; outbound webhooks are the `call_webhook` action of automation rules, logged in `automation_rule_runs.actions_result`. `lib/webhooks/assinatura.ts` only names the inbound header.
4. **Routing skips groups only in the last definition** of `fn_request_channel_routing` (migration 0482, baseline appendix); the worker does not check `is_group`.
5. **Constraints redefined in the baseline appendix:** `conversations_status_check` still accepts legacy `pending`/`resolved` while the API enum does not; `messages_type_check`, `campaign_recipients_status_check` (`personal`) and `automation_rule_runs` status (`adiado`) are widened only in the appendix.
6. **Error-code catalog gaps:** `media_expired` (410) is not in `lib/api/errors.ts`; `campanha_conteudo_invalido` is documented as 422 but returned with 409 for a duplicate name.
7. **`webhook_sources.secret`** (plaintext) still appears in the CREATE TABLE although routes use `secret_encrypted` since migration 0041.
8. **`lib/waha/README.md`** lists a `throttle.ts` that does not exist; pacing lives in `lib/agent-engine/pacing/`.
9. **`app/api/v1/phone-numbers`** is a VoIP trunk registry, not the Meta channel.
10. **Reopen** (`PATCH status=open`) is audited as `conversation.released`.

## Declared debts

- `call_webhook` retries in-process with `sleep` (3 attempts, 10 s timeout each); a restart mid-retry loses attempts and the event_log handler can block for about 36 s per action. Two simultaneous resends can reuse an attempt number. The legacy `X-Deskcomm-Signature` header is still sent.
- `automation_rules.run_count` is updated read-then-write and can lose increments under concurrency; rule chaining is limited to depth 1.
- `lib/automation/throttle.ts` reads `channel_session_warmup`, which nothing writes, so its warm-up cap never applies.
- Campaign pages rely on the API's manager check rather than their own.
- `attendant-heartbeat` was removed; `last_heartbeat_at` is informational and presence never switches availability off.
- No general integrator guide for inbound capture exists in `docs/webhooks/`.
- WhatsApp partner providers (`channels/partner`, `graph-partner`) are not covered by any spec yet (candidate `whatsapp-partner-providers`).

## Goals / Non-Goals

- Goal: an accurate, diffable messaging baseline per canonical id.
- Non-goal: fixing any item above.
