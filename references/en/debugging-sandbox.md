# Debugging and sandbox

Workflow debugging, sandbox benches, saving money on expensive scenarios.

---

## ✅ Solutions

### DBG-R1. Sandbox bench for expensive scenarios

- **Context:** the scenario burns tokens (LLM + web search + image generation) on every
  run, but the last mile broke (Telegram, HTTP notification, write to DB/Notion,
  document generation). A full run just to verify the fix is expensive.
- **Criteria for setting up a sandbox:**
  - a single full run is noticeably expensive in tokens;
  - the broken part is an **output** node (the last mile);
  - a series of fix iterations is expected (more than one edit).
- **Structure:**

```
Webhook (POST /webhook/<name>-test) → Set (data as from the real chain) → copy of the problem node
```

  The Set builds an item with exactly the structure the problem node sees in production (look it
  up in the Output of the previous node on a real execution). The problem node is an exact copy
  with the same `credentials`, `parameters`, `additionalFields`.
- **Storage rules:** the sandbox lives in a separate subfolder (`sandbox/<name>/`) with its own
  description and `workflow.json`. The default data reproduces the real failed case. The
  parameters of the node under test are identical to production (`parse_mode`,
  `appendAttribution`, `binaryData`, HTTP headers) — otherwise the sandbox gives a false "green".
- **Publishing rules:** do **not** delete the sandbox after the fix — it lives on as a tool for
  regressions. After each test session you **must deactivate** it
  (`POST /api/v1/workflows/{id}/deactivate`); before the next one, activate it back
  (`POST /api/v1/workflows/{id}/activate`). An active public webhook that isn't needed is an
  unnecessary risk of URL leakage.
- **Algorithm for the "check it in the sandbox" trigger:** find a suitable sandbox (or set up a
  new one) → activate → run tests with curl → deactivate → apply the fix in production.

Example (curl reads the domain and key from an extracted env file):

```bash
source ./.n8n-api-key   # exports N8N_API_KEY and N8N_BASE_URL

# activate
curl -X POST -H "X-N8N-API-KEY: $N8N_API_KEY" \
  "$N8N_BASE_URL/api/v1/workflows/<sandbox_id>/activate"

# test
curl -X POST "$N8N_BASE_URL/webhook/<sandbox-path>" \
  -H "Content-Type: application/json" \
  -d '{"text":"any test"}'

# deactivate (mandatory afterwards)
curl -X POST -H "X-N8N-API-KEY: $N8N_API_KEY" \
  "$N8N_BASE_URL/api/v1/workflows/<sandbox_id>/deactivate"
```

Status: ✅ Confirmed in production.

### DBG-R2. Order of debugging a failed node

1. Open the Execution Log, find the failed node.
2. Look at the **Input** and **Output** of that node.
3. Check the field mapping (expressions).
4. **Pin Data** — reproduce without a real trigger.
5. **Partial Execution** — run starting from the problem node.

Additional practices: a Sticky Note with the expected data; `console.log()` in the Code Node
(visible in the logs); test on real data via Pin Data before going to production.

### DBG-R3. Code node sandbox: no URLSearchParams/TextEncoder/crypto.subtle — validating initData with pure JS

- **Symptom:** the Code node fails with `ReferenceError: URLSearchParams is not defined` (and there is also no `TextEncoder`, `crypto.subtle`). Web globals are stripped down in the n8n sandbox; `Buffer` is available.
- **Context:** validating the Telegram `initData` for a Mini App = a double HMAC-SHA256 (`secret=HMAC("WebAppData", bot_token)` over raw bytes → `HMAC(secret, data_check_string)`).
- **Solution:**
  1. Parse the query manually: `split('&')` + `decodeURIComponent` (without URLSearchParams).
  2. HMAC: try `require('crypto')` (often blocked by `NODE_FUNCTION_ALLOW_BUILTIN`), fallback — a **built-in pure-JS SHA-256/HMAC** (over `Uint8Array`+`Buffer`). Cross-check the SHA-256 algorithm against a Python port using `hashlib` (emulate the JS bitwise operations `>>>`, `|0`, `~x` — matched on all vectors).
  3. The Code node **cannot read** the bot token from a credential → store it in a service table (`<config_table>`) or env, and load it with a Postgres node before auth.
  4. `crypto.subtle` (Web Crypto) is also absent in this sandbox — don't count on it.
- **Status:** ✅ Confirmed (the frontend authenticates in production).

---

## ⚠️ Errors

### DBG-E1. Double reply on a batch of messages

- **Symptom:** on a batch of messages (several in a row) the bot replies twice.
- **Cause:** the Buffer Upsert sits **after** the Merge — it sees only its own data, not the whole batch.
- **Fix:** move the Buffer Upsert **before** each Wait.

### DBG-E2. A Telegram-trigger webhook can't be hit with curl — secret

- **Symptom:** an attempt to simulate a Telegram update via
  `curl -X POST <N8N_BASE_URL>/webhook/<webhookId>/webhook -d '{...}'` → **HTTP 403**
  `{"message":"Provided secret is not valid"}`.
- **Cause:** the n8n Telegram Trigger registers the webhook with a secret `secret_token`; Telegram
  sends it in the `X-Telegram-Bot-Api-Secret-Token` header. Without the correct secret (which is
  not exposed externally) the request is rejected. This is not a bug — it's protection against
  spoofed updates.
- **Fix:** test bots built on the Telegram Trigger **only with a live bot** (a real
  message/button). To debug the last mile without a full run — use a separate sandbox with an
  ordinary `Webhook` node (no secret) that curl hits (DBG-R1). The Telegram Trigger itself
  cannot be simulated with curl.
