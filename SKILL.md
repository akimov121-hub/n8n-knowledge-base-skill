---
name: n8n-knowledge-base
description: >
  A portable card-file of battle-tested technical solutions, common pitfalls and
  working methodology for self-hosted n8n. Use this skill for ANY work with n8n:
  creating and editing workflows via the REST API, deploying via PUT, working with
  Postgres nodes (executeQuery, Chat Memory), Telegram bots (Trigger, sendMessage,
  sendPhoto, inline buttons, voice/video notes), AI agents (model choice, memory,
  RAG, toolWorkflow), Set nodes, debugging and sandbox benches. ESPECIALLY before a
  risky action (deploy, configuring a node, a new pattern): check the symptom index
  first so you don't repeat a known pitfall. Trigger situations: "credentials lost
  after PUT", "invalid input syntax for type bigint", "parse_mode not working",
  "agent tool unavailable", "how to structure an n8n project", "n8n deploy protocol",
  "how to connect to n8n via API". On first contact with an unfamiliar n8n, offer
  to connect via the API (references/en/api-setup.md) — it's the most convenient way
  to work. Также срабатывает на русском: «создай воркфлоу», «слетели credentials
  после PUT», «не работает parse_mode», «инструмент агента недоступен», «протокол
  деплоя n8n», «подключить n8n по API». Bilingual: English + Russian.
---

# n8n knowledge base — solutions, pitfalls and methodology

This skill is a portable card-file of hands-on experience with **self-hosted n8n**:
confirmed solutions (✅), common errors and how to avoid them (⚠️), plus a methodology
for running an n8n project (structure, naming, architecture principles, deploy protocol).

The skill is not tied to any specific installation. Wherever you see `<placeholders>` —
substitute your own values (domain, credential IDs, workflow IDs, owner's telegram_id).

> **Bilingual.** Every reference file exists in two languages:
> `references/en/` (English) and `references/ru/` (Russian, the original). The symptom
> index below points to the English set; for the Russian mirror just swap `en` → `ru`
> in the path. Entry codes are identical across both languages.

## Main idea: work with n8n through the API

The most convenient and predictable way to drive n8n with an AI assistant is **through
the n8n server's REST API**, not by clicking in the UI. The assistant reads workflows,
surgically edits nodes, deploys, inspects executions and finds errors — programmatically
and reproducibly.

**If you haven't connected the assistant to your n8n via the API yet — start with
`references/en/api-setup.md`** (get a key, set the domain, env file, connectivity check).
Without it, most of the solutions below (deploy, diagnostics, sandbox) won't work.

## How to use (for Claude)

1. Start with the **symptom index** below. Find the row matching the symptom/trigger —
   it points to the right reference file and the entry code (e.g. `TG-E1`).
2. Open that file precisely (`references/en/<domain>.md`), find the entry by its code.
3. Don't scan every file — read only the relevant domain.
4. Before a risky action (PUT deploy, configuring an unfamiliar node, a new pattern),
   check here first.

## Entry codes

Domain prefix + type (`R` = solution / `E` = error) + number. Example: `DEP-R2`, `TG-E1`.

| Prefix | Domain | File |
|---|---|---|
| —   | **Connecting to n8n via the API (start here)** | `references/en/api-setup.md` |
| DEP | Deploying workflows via REST API, JWT tokens | `references/en/rest-api-deploy.md` |
| PG  | Postgres nodes, SQL, Chat Memory | `references/en/postgres.md` |
| TG  | Telegram nodes (trigger, sendMessage, sendPhoto, buttons, voice) | `references/en/telegram.md` |
| SET | Edit Fields (Set), field mapping, Execute Workflow | `references/en/set-transforms.md` |
| AI  | AI Agent, model choice, memory, RAG, toolWorkflow | `references/en/ai-agents.md` |
| HTTP | Calling third-party HTTP APIs: multipart uploads, Code-node sandbox limits, async job_id polling | `references/en/http-requests.md` |
| DBG | Debugging, sandbox benches, expensive scenarios | `references/en/debugging-sandbox.md` |
| IG  | Publishing to Instagram via the Graph API | `references/en/instagram.md` |
| —   | Project methodology (structure, naming, architecture, protocols) | `references/en/methodology.md` |

---

## Symptom index

### ⚠️ Errors

| Symptom / trigger | Code | File |
|---|---|---|
| `Node does not have any credentials set` on the first node after PUT | DEP-E1 | rest-api-deploy |
| SDK `update_workflow` breaks the workflow on topological changes | DEP-E2 | rest-api-deploy |
| Request to n8n from an isolated browser context (extension) → 401 | DEP-E3 | rest-api-deploy |
| n8n silently dropped a node parameter, behavior didn't change | DEP-E4 | rest-api-deploy |
| Cascade of 404 "You do not have permission" = hitting a non-existent id | DEP-E5 | rest-api-deploy |
| After PUT the workflow stays deactivated, `/activate` returns 400 | DEP-E6 | rest-api-deploy |
| PUT returns 400 `settings must NOT have additional properties` | DEP-E7 | rest-api-deploy |
| PUT of the main workflow fails 400 until the sub-workflow is "published" | DEP-E8 | rest-api-deploy |
| `invalid input syntax for type bigint` in Postgres | PG-E1 | postgres |
| executeQuery on 0 rows returns `{success:true}`, not 0 items | PG-E2 | postgres |
| Dollar-quoting with a numeric value breaks n8n's `$N` scanner | PG-E3 | postgres |
| Multi-statement SQL as an n8n expression → "invalid syntax" | PG-E4 | postgres |
| Multi-statement executeQuery returns the FIRST statement's result | PG-E5 | postgres |
| After INSERT/UPDATE, `$json` has no fields from your query | PG-E6 | postgres |
| Postgres's regex quantifier is capped at 255 | PG-E7 | postgres |
| `RETURNING` returns only the listed columns — downstream `$input` is undefined | PG-E8 | postgres |
| A key-based SELECT without `alwaysOutputData` breaks the chain for new users | PG-E9 | postgres |
| Cloud DB IP allowlist doesn't follow a server migration | PG-E10 | postgres |
| A `$` in SQL breaks the query — treated as a placeholder | PG-E11 | postgres |
| Multi-row `VALUES (…),(…)` with placeholders only inserts the first row | PG-E12 | postgres |
| A SQL comment ate a comma and broke a daily workflow for days | PG-E13 | postgres |
| Telegram ignores `parse_mode: ""`, text arrives with Markdown | TG-E1 | telegram |
| sendPhoto can't find the photo / the `photo` param doesn't work | TG-E2 | telegram |
| "sent automatically with n8n" attribution in the bot's messages | TG-E3 | telegram |
| After sendChatAction downstream loses chat_id/message_id/text | TG-E4 | telegram |
| A `{{ ... }}` in a message comes through as raw text (no leading `=`) | TG-E5 | telegram |
| "Bad Request: chat not found" on sendMessage (+ bot-selection rule) | TG-E6 | telegram |
| Sporadic "chat not found" with a valid config (a transient) | TG-E7 | telegram |
| sendPhoto by-URL (imgBB) "failed to get HTTP URL content" | TG-E8 | telegram |
| Update from a channel/supergroup → "channel direct messages topic..." | TG-E9 | telegram |
| `require('crypto')` forbidden in Code nodes — pure-JS HMAC/SHA256 only | TG-E10 | telegram |
| Signing Mini App initData for a test: compact JSON + `%20`, not `+` | TG-E11 | telegram |
| `field expects a number but we got expression_string` in a Set node | SET-E1 | set-transforms |
| Execute Workflow passes its own input item to the sub-workflow | SET-E2 | set-transforms |
| Code node (Run Once for All Items) returning one item drops the rest | SET-E3 | set-transforms |
| `$('Node').item` breaks across nodes once there are ≥2 items | SET-E4 | set-transforms |
| A lookup node with `executeOnce` collapses the stream to one run | SET-E5 | set-transforms |
| `pairedItem` breaks on the openAi v2.1 node (`.item` → undefined) | AI-E1 | ai-agents |
| An agent tool returns "unavailable" — the tool-workflow is deactivated | AI-E2 | ai-agents |
| Postgres Chat Memory `Got unexpected type: constructor` after manual INSERT | AI-E3 | ai-agents |
| Tool-call-leak recovery parser duplicates the write on a repeated marker | AI-E4 | ai-agents |
| Follow-up moved to tomorrow due to the hard floor `in_days≥1` | AI-E5 | ai-agents |
| Switching OpenAI model generation breaks nodes with deprecated params | AI-E6 | ai-agents |
| Reasoning models: response text isn't in `output[0]` | AI-E7 | ai-agents |
| The agent talks about ITSELF using the user's grammatical gender | AI-E8 | ai-agents |
| An echo bot loses the owner notification on delivery failure | AI-E9 | ai-agents |
| The agent lies about "yesterday" — dates are right, subtraction is wrong | AI-E10 | ai-agents |
| A male agent talks about itself in the feminine, mirroring the user | AI-E11 | ai-agents |
| The agent doesn't sense an overnight pause — no timestamps in history | AI-E12 | ai-agents |
| An expensive LLM feature with no counter = a hole in unit economics | AI-E13 | ai-agents |
| The model puts the wrong thing into a structured field | AI-E14 | ai-agents |
| The model sums numbers its own way — give totals to the server | AI-E15 | ai-agents |
| "Last N messages" ≠ "for the period" — window is sliced by count, not time | AI-E16 | ai-agents |
| The bot says "I see the photo" without having vision | AI-E17 | ai-agents |
| JSON repair delivered the report, but gutted — fix the cause, not symptom | AI-E18 | ai-agents |
| Decoupling a subsystem leaves the access gate alive in neighbor workflows | AI-E19 | ai-agents |
| A prompt in an HTTP node's `jsonBody`: over-escaping → `invalid syntax` | AI-E20 | ai-agents |
| A Code node cannot make an authenticated HTTP request | HTTP-E1 | http-requests |
| A pre-built multipart buffer as raw binary → `source.on is not a function` | HTTP-E2 | http-requests |
| Web APIs (`URLSearchParams` etc.) unavailable in the Code node | HTTP-E3 | http-requests |
| Robokassa: the Receipt goes into the signature WITHOUT URL-encoding | HTTP-E4 | http-requests |
| A long synchronous webhook response gets cut off at ~30s outside the server | HTTP-E5 | http-requests |
| Double reply on a batch of messages | DBG-E1 | debugging-sandbox |
| Telegram-trigger webhook returns 403 "secret is not valid" on curl | DBG-E2 | debugging-sandbox |
| Traefik reports "running" but isn't listening on the port | DBG-E3 | debugging-sandbox |
| Two live n8n instances with the same workflows fire schedules independently | DBG-E4 | debugging-sandbox |
| "Insufficient developer role permissions" when linking an IG account | IG-E1 | instagram |
| Error 9004/2207052 "Failed to download media file" creating an IG container | IG-E2 | instagram |

### ✅ Solutions

| What you need to do | Code | File |
|---|---|---|
| Which JWT token for which API | DEP-R1 | rest-api-deploy |
| Deploy a workflow via PUT without losing credentials | DEP-R2 | rest-api-deploy |
| Minimal settings for PUT | DEP-R3 | rest-api-deploy |
| The new workflow's ID — take it only from the server response | DEP-R4 | rest-api-deploy |
| PATCH a credential (self-update; unstable — see DEP-R6) | DEP-R5 | rest-api-deploy |
| Rotate a service secret via `service_tokens` at runtime | DEP-R6 | rest-api-deploy |
| Embed a value of the right type in Postgres SQL | PG-R1 | postgres |
| Arbitrary SQL/DDL via a temporary webhook workflow | PG-R2 | postgres |
| Insert into the lowest free `id` (gap-free numbering) | PG-R3 | postgres |
| Idempotency via atomic status capture (double-press protection) | PG-R4 | postgres |
| Text parameters via queryReplacement as an array-expression | PG-R5 | postgres |
| Webhook response as a list/object via `json_agg` (guaranteed 1 row) | PG-R6 | postgres |
| "One row per user" draft — reset carried-over fields on a new item | PG-R7 | postgres |
| Free JSONB merge (`data \|\| EXCLUDED.data`) — no schema migrations | PG-R8 | postgres |
| Idempotent scheduling via a stage filter on insert | PG-R9 | postgres |
| Atomic payment activation in one retry-safe CTE | PG-R10 | postgres |
| Subscription plan change: charge the difference, cancel the old parent op | PG-R11 | postgres |
| Disable the n8n attribution in Telegram | TG-R1 | telegram |
| Default parse_mode `HTML` + escape `<>&` | TG-R2 | telegram |
| The "typing…" indicator without losing item data | TG-R3 | telegram |
| Inline buttons and forceReply in the Telegram node | TG-R4 | telegram |
| Approval loop (preview with buttons + text/voice edits) | TG-R4* | telegram |
| Accept video notes (video_note) as voice | TG-R5 | telegram |
| Retry On Fail on Telegram sending nodes | TG-R6 | telegram |
| Send an image to Telegram as binary (not by-URL) | TG-R7 | telegram |
| A "private chat only" filter at the bot's entry | TG-R8 | telegram |
| React to an edit of the user's own message as new input | TG-R9 | telegram |
| Graceful handling of broken user HTML in send | TG-R10 | telegram |
| A dynamic card list with inline buttons (id in callback_data) | TG-R11 | telegram |
| Bot menu button = a Web App instead of commands | TG-R12 | telegram |
| Forward/link → Mini App inbox: detect-and-gate before the router | TG-R13 | telegram |
| Web login via the Telegram Login Widget → a session HMAC token | TG-R14 | telegram |
| A secure Telegram avatar proxy (signed URL) | TG-R15 | telegram |
| Channel mirroring without creating a loop | TG-R16 | telegram |
| Deep-link a bot message to a specific Mini App screen | TG-R17 | telegram |
| Choose an LLM for the task | AI-R1 | ai-agents |
| Default stack for AI scenarios | AI-R2 | ai-agents |
| Parse the openAi v2.1 response in JSON mode | AI-R3 | ai-agents |
| Reference-lookup tool for an agent via Postgres full-text (RAG, no vectors) | AI-R4 | ai-agents |
| Agent tool storage with human-readable numbers | AI-R5 | ai-agents |
| Long-term memory of the interlocutor (recall + capture) | AI-R6 | ai-agents |
| A consultant tool writes the result to the DB itself | AI-R7 | ai-agents |
| A proactive reminder must repeat the user's request verbatim | AI-R8 | ai-agents |
| Deduplicate a bloated system prompt (−25% tokens/turn) | AI-R9 | ai-agents |
| Hide long bot output in a Telegram expandable blockquote | AI-R10 | ai-agents |
| Agent states the coarse precision of cron delivery honestly | AI-R11 | ai-agents |
| A web chat on top of the same agent (shared sessionKey) | AI-R12 | ai-agents |
| A deferred follow-up tool that can also fire "today" | AI-R13 | ai-agents |
| A background tool call that the model keeps skipping — how to force it | AI-R14 | ai-agents |
| Weekly deep dive into user data with a collapsible-UI JSON output | AI-R15 | ai-agents |
| A hard length limit an LLM can't self-enforce — margin + deterministic trim | AI-R16 | ai-agents |
| A daily proactive-touch cap via counter+date, not a single last_at | AI-R17 | ai-agents |
| A monetization gate before the agent — fail-open, dedup, absolute counters | AI-R18 | ai-agents |
| Static user data in the system context instead of a mandatory tool call | AI-R19 | ai-agents |
| A consent gate (e.g. 152-FZ) as a deterministic button, not via the LLM | AI-R20 | ai-agents |
| A conditional prompt phase block that doesn't break the prefix cache | AI-R21 | ai-agents |
| Paid generation in a webhook: gate → LLM-JSON → image → INSERT | AI-R22 | ai-agents |
| Vision for food photos: a mini model is enough and cheaper | AI-R23 | ai-agents |
| Long-term memory hygiene: TTL + weekly auto-review with a code guard | AI-R24 | ai-agents |
| Guided agent sessions via a trigger phrase + a prompt protocol | AI-R25 | ai-agents |
| A crisis-safety protocol in a wellness agent's prompt | AI-R26 | ai-agents |
| Verify a prompt change on a synthetic profile + safe deduplication | AI-R27 | ai-agents |
| Robust LLM-JSON parsing: bracket repair + an honest `throw` | AI-R28 | ai-agents |
| Day-by-day proactive funnel: deterministic pick + free-form LLM text | AI-R29 | ai-agents |
| A Mini App action continued by the agent in chat (seeded memory) | AI-R30 | ai-agents |
| Dynamic multipart upload: Switch + K copies of the HTTP node | HTTP-R2 | http-requests |
| Async webhook (job_id + polling) for operations longer than 30s | HTTP-R3 | http-requests |
| Set up/use a sandbox for an expensive scenario | DBG-R1 | debugging-sandbox |
| Order of debugging a failed node | DBG-R2 | debugging-sandbox |
| Validate initData in the Code sandbox with pure JS | DBG-R3 | debugging-sandbox |
| Access the Instagram API without App Review + a photo publish cycle | IG-R1 | instagram |

---

## Methodology (brief)

Full version — `references/en/methodology.md`. The essentials:

- **One scenario — one folder.** All of a scenario's artifacts (workflow.json, prompts,
  description) in a dedicated subfolder. Never dump them in the root.
- **A Sticky Note is the business card** of every workflow: what it does, who triggers it,
  which integrations.
- **A description file** per scenario: goal, what's in the folder, trigger, logic,
  dependencies, development history.
- **Architecture:** one workflow — one responsibility; reusable logic → sub-workflows;
  an Error Workflow is mandatory; Retry On Fail on external HTTP; validate incoming data
  in the first node.
- **"Plan first — then build in one pass"** for ports and large tasks: study → plan →
  decide the nodes → agree → build.
- **Safety:** never delete or irreversibly change data (DB, files, production workflows)
  without the owner's explicit permission.

---

## Rules for keeping the card-file

- **Solutions (✅)** are recorded only after the scenario is confirmed to fully work.
  Until then — status `🟡 Draft` or don't write it.
- **Errors (⚠️)** are recorded immediately, as soon as you hit the pitfall.
- Each entry: context/symptom → cause → fix/code → status.
- A new solution → an entry in the domain file + a row in this SKILL.md index.
