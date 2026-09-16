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

### DBG-R4. Testing a scheduled scenario in the live runtime: a webhook stand-in for "send now"

- **Approach:** add a Webhook (GET, random path like `agenda-now-<hex>`) as a second
  entry point into the chain, feeding into the same first working node. Hitting it with
  `curl` produces a real production execution (`mode: webhook`): the Error Workflow
  fires (it doesn't on a manual Execute), the result is visible in Executions, and the
  schedule itself is untouched. Also convenient for iterating on message formatting with
  the owner: "send it again" = one curl call.
- **One-off alternative:** use PUT to move the cron to "now + 3 min", wait for it to
  fire, then restore the original file with the same PUT (a snapshot before the PUT is
  mandatory). Works when a permanent stand-in isn't needed.
- **Risk:** anyone who learns the path could spam the owner. Keep the path only in the
  private description; in the scenario, the stand-in sends to the same recipient as the
  schedule, so nothing goes to a stranger.
- **Status:** ✅ Confirmed.

### DBG-R5. Simulating Telegram Trigger updates without real Telegram: the webhook secret is `<workflowId>_<nodeId>`

- **Task:** test a bot (messages, callback buttons, `/start` with payload) as a
  different user, without a second account and without pinging real people.
- **Fix:** the Telegram Trigger registers its webhook with a `secret_token` and returns
  403 on any POST missing the `X-Telegram-Bot-Api-Secret-Token` header. The secret is
  computed in `GenericFunctions.getSecretToken` as `${workflow.id}_${node.id}` with
  characters outside `[A-Za-z0-9_-]` stripped out. Get the node id from
  `GET /api/v1/workflows/{id}`; the URL is `<N8N_BASE_URL>/webhook/<webhookId>/webhook`.
  From there, send regular Update objects (`message`, `callback_query`) with that
  header. Replies the bot sends to real people still go through for real, while replies
  to fake users fail with `chat not found` (set `onError: continueRegularOutput` on the
  Bot API node).
- **Status:** ✅ Confirmed on a live bot; the technique has been reused for similar
  checks since.

### DBG-R6. A node with no input items doesn't execute — and silently breaks the whole chain downstream

- **Symptom:** notifications didn't go out, no errors: a Code node returned `[]`
  (nothing to analyze), the next node never ran, and neither did every node after it.
  In the execution view they just show up gray.
- **Rule:** if a chain must reach the end regardless of the outcome, the filter node
  should always emit at least one placeholder item (`{ problem_id: 0 }`), and the
  receiving side should know how to skip it (SQL with `WHERE id = 0` → empty →
  `alwaysOutputData`). On Execute Workflow, set `alwaysOutputData: true` and
  `onError: continueRegularOutput`: an empty response or a failing sub-workflow must
  not swallow the parent's notifications; the parent should re-read the outcome from the
  database rather than take it from the sub-workflow's response.
- **Status:** ✅ Confirmed on real data.

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

### DBG-E5. An error handler that logs to a DB goes silent exactly when it's needed most

- **Symptom:** the connection to the cloud DB dropped for a minute, several scenarios
  failed — and not a single notification arrived. Worse, the error handler itself
  failed three times during that minute. The incident went unnoticed and was only
  found because the owner happened to check the executions list.
- **Cause:** the chain was `Error Trigger → Prepare Error → Log Incident (Postgres) →
  Notify Admin (Telegram)`. Logging the incident to the DB happened **before** sending
  the message. As long as the DB is up, this makes no difference. But the most common
  reason the handler fires in the first place is exactly a DB outage — and in that
  case it fails on the very first step, never reaching the notification.
- **Fix:** `onError: continueRegularOutput` on the logging node (+ 5 retries at 5 s
  each). Logging is now optional: if the write fails, the message still goes out. No
  need to reorder the nodes, since the Telegram node reads its data from `Prepare
  Error`, not from the logging node's result.
- **How it was verified:** a copy of the handler with a deliberately failing SQL query
  (`SELECT * FROM no_such_table_xyz`). The logging node returned the error as data,
  `Notify Admin` still ran and the message was delivered (`message_id` was returned).
  Before the fix, the chain broke at that exact point.
- **General rule:** a notifier must not depend on anything that can fail alongside the
  system it's observing. Everything in it other than actually sending the message is
  optional: either move it after the send, or swallow its errors. Otherwise you end up
  with a watcher that goes blind at the exact moment of the outage.
- **Check for any error handler:** mentally disconnect the DB, the network, the
  external API — and see whether the message still gets through. If not, reorder or
  decouple it.
- **Status:** ✅ Verified on a copy.

### DBG-E6. `n8n execute` from inside the container won't run a scenario with a Schedule Trigger, and conflicts with the live instance over the runner port

- **Symptom:** `docker exec <n8n> n8n execute --id=<id>` → `n8n Task Broker's port 5679
  is already in use` (a second n8n process in the same container). With
  `N8N_RUNNERS_BROKER_PORT=5680` it starts, but then fails: `Missing node to start
  execution — workflow must contain an Execute Workflow Trigger` — in 2.x the CLI can
  only start from a Manual or Execute Workflow Trigger; it can't "press" a schedule.
- **What to do instead:** don't waste time on the CLI — test in the live runtime, see
  DBG-R4.
- **Status:** ✅ Confirmed.

### DBG-E7. The Code node truncates the error message down to the part after the colon and appends `[line N]`

- **Symptom:** `throw new Error('Channel check: this message should arrive')` →
  in the Error Trigger, `execution.error.message` comes out as `this message should
  arrive [line 1]`. n8n treats everything before the colon as the error type label and
  drops it.
- **Rule:** don't start `throw` text with a "Label: …" pattern — use a dash or
  parentheses instead; in the error handler, pull in `error.description` if you need
  the rest.
- **Status:** ✅ Confirmed.

### DBG-E8. Under the task runner, a Code node has `require('crypto')`, `$env`, and `process` blocked, and no Web Crypto

- **Symptom:** `require('crypto')` → `Module 'crypto' is disallowed`; `$env.X` →
  `access to env vars denied` (`N8N_BLOCK_ENV_ACCESS_IN_NODE=true`); `process` is
  undefined; `globalThis.crypto.subtle` is `undefined`. `Buffer` is available.
- **Rule:** on this setup, write HMAC/SHA-256 in plain JS (a working implementation
  already exists in the Mini App API auth node and has been reused elsewhere). Pass
  secrets into the node as placeholders via the deploy script, not through `$env`.
  Check which modules are actually available with a temporary webhook scenario rather
  than guessing — it depends on `NODE_FUNCTION_ALLOW_BUILTIN` and the runner mode.
- **Status:** ✅ Confirmed.
