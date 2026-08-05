# Debugging and sandbox

Workflow debugging, sandbox test rigs, cost savings on expensive scenarios.

---

## ✅ Solutions

### DBG-R1. Sandbox test rig for expensive scenarios

- **Context:** a scenario burns tokens (LLM + web search + image generation) on every
  run, and the last mile broke (Telegram, HTTP notification, DB/Notion write,
  document generation). A full run just to check the fix is expensive.
- **Criteria for setting up a sandbox:**
  - a single full run is noticeably expensive in tokens;
  - the broken part is an **output** node (the last mile);
  - a series of fix iterations is expected (more than one edit).
- **Structure:**

```
Webhook (POST /webhook/<name>-test) → Set (data as it would come from the real chain) → copy of the problem node
```

  The Set node builds an item with exactly the structure the problem node sees in
  production (check the Output of the preceding node on a real execution). The problem
  node is an exact copy with the same `credentials`, `parameters`, `additionalFields`.
- **Storage rules:** the sandbox lives in a separate subfolder (`sandbox/<name>/`) with
  its own description and `workflow.json`. Default data reproduces the real failing case.
  Parameters of the node under test are identical to production (`parse_mode`,
  `appendAttribution`, `binaryData`, HTTP headers) — otherwise the sandbox gives a
  false "green".
- **Publishing rules:** **do not delete the sandbox after the fix** — it stays as a
  regression tool. After each testing session it **must be deactivated**
  (`POST /api/v1/workflows/{id}/deactivate`); reactivate before the next session
  (`POST /api/v1/workflows/{id}/activate`). An active public webhook with no
  current need is an unnecessary URL-leak risk.
- **Algorithm on the "test in sandbox" trigger:** find a suitable sandbox (or create a
  new one) → activate → run curl tests → deactivate → apply the fix in production.

Example (curl reads the domain and key from an externalized env file):

```bash
source ./.n8n-api-key   # exports N8N_API_KEY and N8N_BASE_URL

# activate
curl -X POST -H "X-N8N-API-KEY: $N8N_API_KEY" \
  "$N8N_BASE_URL/api/v1/workflows/<sandbox_id>/activate"

# test
curl -X POST "$N8N_BASE_URL/webhook/<sandbox-path>" \
  -H "Content-Type: application/json" \
  -d '{"text":"any test"}'

# deactivate (mandatory afterward)
curl -X POST -H "X-N8N-API-KEY: $N8N_API_KEY" \
  "$N8N_BASE_URL/api/v1/workflows/<sandbox_id>/deactivate"
```

Status: ✅ Confirmed in production.

### DBG-R2. Order of operations for debugging a failed node

1. Open the Execution Log, find the failed node.
2. Check the node's **Input** and **Output**.
3. Check field mapping (expressions).
4. **Pin Data** — reproduce without a real trigger.
5. **Partial Execution** — run starting from the problem node.

Additional practices: a Sticky Note with the expected data; `console.log()` in a Code
Node (visible in the logs); test on real data via Pin Data before production.

### DBG-R3. Code node sandbox: no URLSearchParams/TextEncoder/crypto.subtle — validating initData in plain JS

- **Symptom:** the Code node throws `ReferenceError: URLSearchParams is not defined`
  (also missing: `TextEncoder`, `crypto.subtle`). Web globals are stripped down in the
  n8n sandbox; `Buffer` is available.
- **Context:** validating Telegram `initData` for a Mini App requires double
  HMAC-SHA256 (`secret=HMAC("WebAppData", bot_token)` on raw bytes →
  `HMAC(secret, data_check_string)`).
- **Fix:**
  1. Parse the query string manually: `split('&')` + `decodeURIComponent` (no
     URLSearchParams).
  2. HMAC: try `require('crypto')` first (often blocked by
     `NODE_FUNCTION_ALLOW_BUILTIN`), fall back to a **hand-rolled pure-JS
     SHA-256/HMAC** (on `Uint8Array`+`Buffer`). Verify the SHA-256 algorithm by porting
     it to Python and comparing against `hashlib` (emulate JS bitwise ops `>>>`, `|0`,
     `~x` — matched on all test vectors).
  3. The Code node **cannot read** the bot token from a credential → store it in a
     service table (`<config_table>`) or env var, load it with a Postgres node before
     auth.
  4. `crypto.subtle` (Web Crypto) is also absent in this sandbox — don't rely on it.
- **Status:** ✅ Confirmed (the frontend authenticates successfully in production).

---

## ⚠️ Errors

### DBG-E1. Double reply on a message batch

- **Symptom:** on a batch of messages (several in a row) the bot replies twice.
- **Cause:** the Buffer Upsert sits **after** Merge — it only sees its own data, not
  the whole batch.
- **Fix:** move the Buffer Upsert to **before** each Wait.

### DBG-E2. The Telegram-trigger webhook can't be hit with curl — secret required

- **Symptom:** trying to simulate a Telegram update via
  `curl -X POST <N8N_BASE_URL>/webhook/<webhookId>/webhook -d '{...}'` → **HTTP 403**
  `{"message":"Provided secret is not valid"}`.
- **Cause:** the n8n Telegram Trigger registers its webhook with a secret
  `secret_token`; Telegram sends it in the `X-Telegram-Bot-Api-Secret-Token` header.
  Without the correct secret (which is never exposed) the request is rejected. This is
  not a bug — it's protection against forged updates.
- **Fix:** bots on Telegram Trigger should only be tested with a **live bot** (a real
  message/button press). For debugging the last mile without a full run, use a
  separate sandbox with a plain `Webhook` node (no secret) that curl can hit (DBG-R1).
  The Telegram Trigger itself cannot be simulated with curl.

### DBG-E3. Traefik reports "running" but isn't listening on the port — container was created during a port conflict

- **Symptom:** `docker compose up -d` after a port conflict (e.g. "address already in
  use") → you resolve the conflict → running `up -d` again prints "Container traefik
  Started", status `running`, but `docker port <container>` is empty, and `curl` to
  80/443 fails with `Connection refused`.
- **Cause:** on the first (failed) run, Docker had already **created** the traefik
  container with its ports before it failed on the bind — the container was left in a
  "Created" state without the port mappings actually applied. The subsequent `up -d`
  sees the existing container and just **starts** it as-is, without recreating it — the
  ports themselves aren't reapplied on a plain `start`.
- **Fix:** `docker compose up -d --force-recreate <service>` — recreates the container
  from scratch, so the ports are applied again. A plain `up -d` won't help in this
  situation no matter how many times you repeat it.
- **Diagnosis:** if `docker ps` shows `Up` but `docker port <name>` is empty and
  `ss -tlnp` doesn't see the port — this is it.
- **Status:** ✅ Confirmed (fix applied and verified).

### DBG-E4. Two live n8n instances with the same active workflows — schedule triggers fire independently, regardless of DNS/routing

- **Symptom:** after migrating to a new server (DNS already switched over), a workflow
  with a **cron/Schedule Trigger** suddenly fails with an error on the new server at
  the exact moment it ran successfully on the old one.
- **Cause:** cron/schedule triggers tick **independently inside each n8n instance** —
  they never go through the domain/reverse proxy/DNS at all. If the old server isn't
  stopped after the migration (data restored, workflows active on both) — both
  instances fire on the same tick simultaneously and both try to process the same
  "due" record in external state (a queue, a database, an external API) → a
  race/duplicate/conflict.
- **Difference from webhooks:** webhook triggers are routed through a reverse proxy
  (e.g. Traefik) by domain — there, DNS decides which server gets the request.
  Schedule triggers aren't saved by this.
- **Fix:** on any n8n server migration — **stop (not necessarily delete) n8n/n8n-worker
  on the old server** as soon as the new one is confirmed fully working, rather than
  keeping both alive "just in case." If parallel testing is needed, manually deactivate
  the schedule workflows on one of the two instances for the duration of the check.
- **Status:** ⚠️ Open risk until the old server is fully stopped.
