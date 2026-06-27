# AI Agents

AI Agent, model selection, memory, RAG, tool workflows.

> Models and prices are current as of writing (mid-2026). Verify against the provider's current
> pricing and availability before choosing.

---

## ✅ Solutions

### AI-R1. Choosing a model for the task

| Task | Model | Rationale |
|---|---|---|
| Agent for a client, daily use | `gpt-5.4-mini` | Cheap, fast, smart enough |
| Complex agent with reasoning | `gpt-5.4` | Price/intelligence balance |
| Maximum quality, analytics | `gpt-5.5` | Flagship |
| Sensitive client data (152-FZ) | Ollama (local) | Data never leaves the server |

**Default rule:** start with `gpt-5.4-mini`, move up to a stronger model only if the quality is
unsatisfactory. (Verify model names against the provider's current lineup — it gets updated.)

### AI-R2. Default stack

- **LLM:** `gpt-5.4-mini` (everyday tasks) / `gpt-5.5` (complex agents) / Ollama
  (sensitive data).
- **Memory:** Postgres Chat Memory (persistent, reliable).
- **RAG search:** Qdrant or Supabase (pgvector) — if vector search is available.
- **Embeddings:** OpenAI `text-embedding-3-small`.
- **Sensitive data:** Ollama locally, not the cloud.

### AI-R3. Parsing the openAi node v2.1 response in JSON mode (text is a dict, not a string)

- **Context:** with `options.textFormat.textOptions.type = "json_object"` the
  `@n8n/n8n-nodes-langchain.openAi` v2.1 node returns `output[0].content[0].text` **as an
  already-parsed dict**, not a JSON string. If a downstream Code node tries `JSON.parse(text)`
  on it, it drops into the fallback with empty fields, and the whole pipeline produces empty
  output all the way to the end.
- **Solution:** in the Code node, support both formats:

```js
const resp = $input.first().json;
const _t = resp.output?.[0]?.content?.[0]?.text;
let parsed = null;
if (_t && typeof _t === 'object') {
  parsed = _t;                              // new path: dict already parsed
} else if (typeof _t === 'string') {
  try { parsed = JSON.parse(_t); } catch(e) { parsed = null; }
}
if (!parsed || typeof parsed !== 'object') {
  parsed = { /* fallback with default fields */ };
}
```

Status: ✅ Confirmed in production.

### AI-R4. A reference-lookup tool for the agent via Postgres full-text (RAG without vectors)

- **Context:** the agent needs on-demand access to a curated reference set — quotes, FAQs,
  policies, email templates, product descriptions, legal norms. Full vector RAG is not always
  available: on some shared hosting the `vector` extension (pgvector) is **not installed**
  (check: `SELECT count(*) FROM pg_available_extensions WHERE name='vector'`). For a corpus of
  tens to hundreds of records you don't even need semantics.
- **Solution:** the reference set = a plain Postgres table + full-text search. A
  `tsv tsvector` column + a GIN index. The agent's tool is a `toolWorkflow` node → sub-workflow
  (`executeWorkflowTrigger` with a `query` input → Postgres `executeQuery`). The agent passes
  keywords via `$fromAI('query', …)`.

```sql
CREATE TABLE ref (id SERIAL PRIMARY KEY, text_ru TEXT, text_en TEXT, theme TEXT, source TEXT, tsv tsvector);
UPDATE ref SET tsv = to_tsvector('russian', coalesce(text_ru,'')||' '||coalesce(theme,''))
                   || to_tsvector('english', coalesce(text_en,''));
CREATE INDEX ref_tsv_idx ON ref USING GIN(tsv);
```

Query in the sub-workflow (`queryReplacement = ={{ $json.query }}`; `$1` can be repeated):

```sql
SELECT text_ru, text_en, source FROM ref
WHERE tsv @@ plainto_tsquery('russian',$1) OR text_ru ILIKE '%'||$1||'%' OR theme ILIKE '%'||$1||'%'
ORDER BY ts_rank(tsv, plainto_tsquery('russian',$1)) DESC LIMIT 5;
```

- **Why this way:** the corpus lives in the DB, the agent pulls the top 5 → the size of the base
  has **no effect** on tokens or response speed (unlike "the whole reference set in the system
  prompt"). Scales to hundreds of rows instantly.
- **Important (quality):** the tool only retrieves — accuracy is the responsibility of the
  content. Vet the reference set against an authoritative source. In the prompt, require the
  agent to use verbatim ONLY what the tool returned and not to make things up if the search
  comes back empty.
- **Evolution:** once pgvector / Qdrant is available, it switches to semantics without changing
  the tool's interface.

Status: ✅ Confirmed in production.

### AI-R5. An agent tool storage with human-readable numbers (memory, reminders, lists)

- **Context:** the agent's tool (`toolWorkflow`) stores the user's personal records (memory,
  reminders, todos) and shows their numbers in the chat. Requirements: numbers without gaps and
  reuse of freed-up ones, in-place editing, minimal re-prompts.
- **Pattern (4 parts):**
  1. **Number = `id`, gap-fill.** `save/set` inserts into the smallest free `id`
     (see postgres.md PG-R3). The displayed number = the real PK, so `delete`/`update` are
     addressed directly by it.
  2. **The `update` operation (in-place edit).** A separate Route output: input `id` + the new
     full `content`, recompute derived fields (for semantic memory — re-embed),
     `UPDATE … WHERE id=… RETURNING id`. NOT delete+insert (otherwise the number moves to the end).
  3. **Display — a plain number.** In the Code formatter, `'  '+r.id+'. '+text`, not `[id N]`.
     In the prompt, call it a "number," not an "id."
  4. **Autonomy via the prompt.** Perform deletion/editing immediately if the record is
     unambiguously identified. Re-prompt ONLY on ambiguity. An `id` into the unknown — first
     `search`/`list`.
- **Deploy:** edit a live `toolWorkflow` via PUT — preserve node ids, add credentials to every
  node, after PUT verify it's active with a retry (see rest-api-deploy.md DEP-E6). Test the SQL
  on a local Postgres before production.

Status: ✅ Confirmed in production.

### AI-R6. Long-term memory of the interlocutor "like a real person" (recall + capture)

- **Context:** the bot must remember facts about the interlocutor from conversation to
  conversation and recall them itself without an explicit "remember" request (example: "I'm
  going to the park with my dog Rex" → later the bot itself knows who Rex is). Separate from
  conversation memory.
- **Pattern (2 independent mechanisms):**
  1. **Recall = "everything into context," without embeddings and without search.** Before the
     agent, a `Load Facts` node (Postgres) pulls ALL of the user's facts and concatenates them
     into one block:
     `SELECT COALESCE(string_agg('- ['||category||'] '||content, E'\n' ORDER BY category,id),'') AS facts_block ... WHERE telegram_id=...`.
     The block is injected into the System Message via the expression
     `{{ $('Load Facts').first().json.facts_block }}`. Only this way does a fact surface on its
     own, without a command to search: semantic search does NOT work here — the agent wouldn't
     think to search for "Rex." Embeddings aren't needed while there are tens to hundreds of
     facts per user.
  2. **Capture = the agent calls the `save/update/delete` tool itself** (based on AI-R5, but
     without embeddings). KEY: a single soft instruction in the middle of the prompt is NOT
     enough — the model focuses on its in-character reply and silently skips saving. You need a
     **hard directive at the very TOP** of the System Message: "a system action — memory, always
     performed before the reply; if the message contains a new/changed fact about the
     interlocutor, you MUST call memory.save/update, otherwise it's an error." After that it
     calls reliably.
- **Important nuance about novelty:** the agent measures "is this a new fact" by the
  **conversation window**, not by the DB. A fact already stated within the window will NOT be
  saved again. Anything discussed BEFORE saving actually started working won't make it into the
  base on its own — enter it manually. To prove that recall comes from memory and not from the
  window, clear the conversation history.
- **telegram_id — not from the model.** In a multi-user bot, `telegram_id` is passed into the
  tool via the expression `$('Normalize').first().json.telegram_id` (not `$fromAI`), otherwise
  the model can mix up users.
- **Not implemented (groundwork):** a separate fact-extraction module on a mini-model (a
  deterministic pass after the reply that writes around the agent). A standalone technique for
  scenarios where missing a fact is costly: lead qualification, collecting form data from a
  conversation, CRM enrichment, support with capturing the details of a request. A deterministic
  pass (always fires, doesn't depend on the model's mood) is more reliable than "the agent
  decides for itself."

Status: ✅ Confirmed in production (the bot recalled a fact after a full clearing of the
conversation history — recall came only from the fact memory).

### AI-R7. The consultant tool writes the result to the DB itself (deterministically) and returns the original answer to the agent

- **Context:** the main agent must save a sub-agent's structured result (program, plan, numbers)
  into the profile. If the agent does this itself via an update tool, the step is probabilistic
  and is **silently skipped** (example: the program was generated, but `program_json` stayed
  NULL).
- **Solution:** the write is done by **the consultant's tool workflow itself** as a side effect,
  not depending on the LLM. After the Agent node: `Persist` (Code: parses the agent's
  `{output}`, builds `_sql`+`_params` by `mode`) → `Save?` (IF `_save`) → `Save Field` (Postgres
  `executeQuery`, `query="={{ $json._sql }}"`, `queryReplacement="={{ $json._params }}"`) →
  `Return` (Code: `return [{ json: $('Agent').first().json }]` — returns to the agent exactly
  its own answer). A single Persist covers several modes (PLAN→`nutrition_json`,
  CAFFEINE→`caffeine_mg/norm`). `telegram_id` comes from the input `profile`.
- **Gotcha (Code node):** in `runOnceForEachItem`, returning `[{ json: { …array… } }]` failed
  with `A 'json' property isn't an object [item 0]`. Fix: **`runOnceForAllItems`** +
  `const out = $input.first().json;`.
- **Where NOT to do this:** large docs (program/nutrition_json) — the consultant writes them,
  and remove "save it yourself" from the main agent's prompt (otherwise double write/race
  condition). Small scalars (weight, age) — still the agent via an update tool.

Status: ✅ Confirmed (isolated PLAN+CAFFEINE test, production).

### AI-R8. A proactive reminder must repeat the user's request VERBATIM

- **Context:** on a direct request, the agent sets a one-off reminder (the request text is stored
  in `note`), and a separate cron workflow later generates the reminder text via a short
  mini-LLM prompt and sends it (the `Build Prompt` node, branch `kind='reminder'`).
- **Gotcha:** a generic mini-LLM prompt ("write a warm message to the user") devolves into a
  **boilerplate question** about wellbeing and does NOT mention the request itself. The user
  doesn't recognize the message as the reminder they ordered — it looks like a random nudge.
  (The tool was called and fired normally — the bug was not "the tool didn't fire," but in the
  generated text + a delay of up to an hour due to the hourly cron.)
- **Solution:** hard-code into the reminder branch's prompt: "This is a reminder for an EXPLICIT
  request. Start with the words 'You asked me to remind you…' and state the gist (`note`)
  VERBATIM. Do NOT turn it into a boilerplate question. Empty gist → a short neutral reminder."
  That is, `note` must end up in the text verbatim, not merely serve as the "mood" for free
  generation.
- **General principle:** when a deferred message is produced by a separate LLM step, the user's
  original intent must be carried into the text verbatim — free generation "on the topic" loses
  recognizability for the recipient.

Status: ✅ Confirmed (short test).

### AI-R9. Deduplicating a bloated system prompt — a single canon instead of repeats (−25% tokens/turn)

- **Context:** a large agent's system prompt goes to the LLM on EVERY call — every extra line is
  paid for on every turn. Over time a mature prompt accumulates duplicates: the same
  prohibition/rule rewritten in 3 sections "for reliability." Example: the prompt ballooned to
  ~50,000 chars.
- **Solution:** reduce repeated blocks to **a single canon** + references/deltas:
  - the triple prohibition on tool/field names → one canon (in the "Hard Contract"), referenced
    by the other sections;
  - three copies of "when to call sub-agent X" → one routing section;
  - three "rewrite the answer in your own voice" → one shared block + short deltas by type;
  - three "degradation on a tool error" → one;
  - ~60 enumerated names for gender detection → a rule + a short example list.
  Result: 49967 → 37613 chars (**−24.7%**) without changing behavior or meaning.
- **Gotchas/limits:** (1) deduplication is a prompt refactor — verify it behaviorally (run
  through scenarios), not just by the diff; (2) do NOT leave a changelog/"before-after" in the
  prompt body — that's tokens again on every turn; keep history in a version log. (3) The canon
  must be in a section the agent will definitely read (top/contract), otherwise "see above"
  references are risky.

Status: ✅ Confirmed (short test).

### AI-R10. Hide a long structured bot output in a Telegram expandable quote (`<blockquote expandable>`) via a sentinel marker + a post-node

- **Context:** the agent sometimes returns a large structured block (a workout program, a
  nutrition plan, a long list) — in chat this is a "wall of text" that's awkward to scroll.
  Telegram can collapse such a block into an expandable quote
  (`<blockquote expandable>…</blockquote>`), but the tag works **only with `parse_mode` HTML or
  MarkdownV2** and the content inside must be validly escaped.
- **Solution (separation of concerns: model ↔ deterministic node):**
  1. **The model wraps the block in a simple sentinel**, rather than writing HTML itself: in the
     System Message — the rule "wrap the full program/plan in `⟪PROGRAM⟫ … ⟪/PROGRAM⟫`, and
     before the marker give 1–2 lines of the gist." The LLM is not forced to generate correct
     HTML (it would mess up the escaping) — only to place a cheap text marker.
  2. **A separate post-node before sending** (`Clean Output`, Code) finds the block by the
     sentinel and converts it into `<blockquote expandable>`, **escaping only the block's
     content** (`&`→`&amp;`, `<`→`&lt;`, `>`→`&gt;`) — leaving the rest of the text untouched.
  3. **Backward compatibility:** no sentinel → the text goes out unchanged. Existing answers
     don't break.
- **Important nuances:**
  - The pipeline's `parse_mode` must already be HTML (see telegram.md TG-R2) — in plain mode the
    tag arrives as raw text.
  - **Do not split with the 4096 splitter inside the block** — breaking a `<blockquote>` in the
    middle ruins the markup. If the block can exceed the limit — split at block boundaries, not
    inside.
  - Pick an "exotic" sentinel (`⟪ ⟫`) so as not to collide with ordinary user/model text.
- **General technique:** build any "model generates → node formats" task with fragile target
  syntax (HTML, MarkdownV2, JSON) this way: the model places a simple marker, a deterministic
  node turns it into the target syntax with correct escaping. More reliable than trusting the LLM
  with escaping.

Status: ✅ Confirmed (renders correctly in Telegram).

### AI-R11. The agent should honestly state the coarse precision of cron delivery, not promise an exact time

- **Context:** on request, the agent sets a one-off reminder, but a separate cron with coarse
  granularity delivers it (**once an hour** — a deliberate design, not a "reminder manager"). The
  user asks "remind me in 10 minutes."
- **Gotcha:** the agent amiably echoes the user — "Sure, I'll remind you in 10 minutes." But the
  cron fires on the next hourly tick → it actually arrives within the hour. Promising an exact
  time is **misleading** (especially for intervals shorter than the cron's step), even though the
  mechanism itself works fine.
- **Solution:** bake honesty about precision into the prompt: don't promise minute-level delivery
  and don't echo "in N minutes" if N is smaller than the cron's step; for a short interval —
  warn "I check roughly once an hour, I won't hit that exact minute, I'll remind you within the
  next hour" and set it anyway (or offer to tie it to a specific time if precision is needed);
  for a specific time/hours — confirm without "to the second." Do NOT change the cron frequency.
- **General principle:** when execution is asynchronous and coarse in time (cron, queue, batch),
  the precision expectation shapes the agent's answer — the wording must match the real delivery
  granularity, not what the user wants. Fix it with expectation-setting, not necessarily
  frequency.

Status: ✅ Accepted for rollout (edited in production; final visual check — by a tester).
Related to AI-R8 (same cron).

### AI-R12. A web chat on top of THE SAME agent — end-to-end Telegram↔web history via a shared sessionKey=telegram_id

- **Context:** a Telegram bot-agent gained a second channel (web/Mini App chat). It needs to be
  **one conversation** continuing across both channels, not two isolated bots with separate
  memory.
- **Solution:** build the new channel as an agent with **three shared components:** (1) **the same
  system prompt** (read from the same `prompts.<name>`), (2) **the same Postgres Chat Memory with
  the same `sessionKey`** = the user's `telegram_id` (not the web session's chat_id!), the same
  window, (3) **the same model**. Then both branches write/read the same `<bot>_chat_history`
  table under one key → the history is end-to-end: reply in Telegram — it's visible on the web,
  and vice versa.
- **Isolation without sharing nodes:** the web workflow itself is **separate** (`<workflow_name>`),
  doesn't touch the main bot (`<workflow_name>` is byte-identical). Shared — only data (the memory
  table, the prompt row, the model credential), not nodes. This protects the main bot from
  regressions when editing the web channel.
- **`chat/history` for the UI — ordering.** To give the frontend **the last N** turns and at the
  same time **in ascending order** (old on top): `SELECT … ORDER BY id DESC LIMIT N`, then
  **reverse the array in Code**. `ASC LIMIT N` returns the start of the history (not the tail);
  `DESC` without reversing — the feed backwards. Filter out tool messages (`type='tool'` and
  assistant tech rows).
- **Boundary (deliberate):** you may NOT connect the main bot's tools (sub-agents/proactive/
  reminders) to the web agent for the sake of isolation — then the web chat is a "pure
  conversation" on shared memory, while the tool-driven scenarios stay with Telegram. Tool parity
  is a separate task.
- **Mirroring** the second channel into the Telegram conversation (so web replies also appear in
  the chat with the bot) — see telegram.md TG-R16; web-channel sign-in via the Login Widget —
  telegram.md TG-R14.

Status: ✅ Confirmed in production (history in ascending order; `chat/send` writes both messages
to the shared memory, the agent's reply arrives).

### AI-R13. A deferred follow-up tool must be able to do it TODAY: remove the hard floor, past time → nearest slot in the window, otherwise next day (mirror the reminder tool)

- **Context:** the agent has a "schedule a follow-up" tool (`schedule_followup`): an `in_days`
  argument + hour, a row into `proactive_queue`, with a cron handling delivery. Logically "let's
  talk this evening / later today" is a follow-up for **today**.
- **Gotcha:** the tool had a hard floor `in_days = Math.max(1, …)` — structurally it COULD NOT
  schedule for today. Any "this evening" request silently moved to tomorrow (`due_at` = tomorrow
  19:00). From the user's side: "promised to talk in the evening — and didn't write today." The
  bug wasn't in the tool call (it fired), but in the lower bound of the range + in the argument
  description, which didn't permit "today."
- **Solution (three layers, all mandatory):**
  1. **Remove the floor:** `in_days = Math.max(0, Math.min(30, Number(t.in_days)||0))` (0 =
     today).
  2. **Compute `due_at` by mirroring the reminder tool, not "dumb +N days":** if the computed
     `due0 <= now()` (in the user's timezone) — shift to **the nearest whole hour within the
     cron's delivery window** (≤ the window's upper bound, e.g. 20:00); if it's already past the
     window — to the **next day** at the requested hour. `in_days≥1` (the future) — unchanged.
     Done as a CTE `adj` in the Insert node's SQL, modeled on the reminder tool.
  3. **The argument description + the prompt must explicitly permit "today":** the tool text /
     `$fromAI` hint — "0 = today (in the evening/later today), 1 = tomorrow, 2–3 = in a couple of
     days"; in the agent's prompt — routing "this evening/later" → `in_days:0` (or
     `set_reminder` for a specific time), "tomorrow/in a couple of days" → `in_days≥1`. Without
     this the model still schedules for tomorrow, even if the backend already can do today.
- **General principle:** the allowed range of a scheduling tool must include "now/today," and the
  delivery-time computation must account for the fact that the requested moment may have already
  passed (shift to the nearest slot, not blindly to tomorrow). Both the range bounds and **the
  argument description, and the prompt** must permit the near bound — otherwise an "in the
  evening" promise is structurally impossible. Reuse the "past time → nearest slot / next day"
  logic from the reminder tool, don't reinvent it.
- **Cap nuance:** if the proactive has a daily cap (e.g. soft proactive 1/calendar day), a
  same-day follow-up will fire only if there hasn't been a proactive today yet. For guaranteed
  same-day delivery — pair it with `set_reminder` (bypasses the window/cap).

Status: ✅ Confirmed (tests: in_days=0 in the morning → today 19:00; in_days=0 late/at night →
tomorrow; in_days≥1 as before). Mirrors AI-R8/AI-R11 (same cron) and the reminder tool's logic.

---

## ⚠️ Errors

### AI-E1. `pairedItem` breaks on the `@n8n/n8n-nodes-langchain.openAi` v2.1 node

- **Symptom:** in an expression after openAi v2.1 you write `$('Node_name').item.json.field` —
  it returns `undefined`. The downstream node gets empty parameters (in Telegram sendPhoto this
  gives `Bad Request: there is no photo in the request`), even though the ancestor actually
  returned a value.
- **Cause:** the openAi v2.1 node doesn't propagate pairedItem metadata. `.item` uses pairing to
  determine "which parent item produced the current one," and without it falls into undefined.
  Especially painful in chains like `imgBB → openAi → sendPhoto`, where the url is taken via a
  cross-node ref.
- **Fix:** in ALL cross-node expressions where at least one openAi v2.1 sits between source and
  consumer — use `.first()` instead of `.item`:
  - ❌ `$('Upload to imgBB').item.json.data.url`
  - ✅ `$('Upload to imgBB').first().json.data.url`
- **Prevention:** by default write `.first()` always in cross-node refs — it's safe in all
  cases, and use `.item` only when you explicitly need the pairing binding.

### AI-E2. An agent tool (`toolWorkflow`) silently returns "unavailable" if the sub-workflow is deactivated

- **Symptom:** the agent in the chat replies "the tool is currently unavailable." Meanwhile the
  main bot's execution is status **success** (not error), so it isn't immediately visible in the
  logs. The problem tool node's runData holds:
  `{"response": "There was an error: \"Workflow is not active and cannot be executed.\""}`.
  The tool workflows themselves have **no separate executions** (the call is inline within the
  parent).
- **Cause:** the sub-workflow tool (trigger `executeWorkflowTrigger`) was switched to
  `active=false`. When called via the AI Agent, such a workflow returns the error as a string in
  `response`, and the agent interprets it as "the tool is unavailable." The parent execution
  stays success — the error doesn't surface in the Error Workflow.
- **Fix:** reactivate the tool workflow: `POST /api/v1/workflows/{id}/activate`. For
  `executeWorkflowTrigger`, activation is safe — it doesn't run on its own.
- **Diagnostics:** don't trust the parent's `success` status. Open the main bot's execution with
  `includeData=true`, in `runData` check the `response` of each tool node for the substring
  `There was an error`. In parallel, `GET /api/v1/workflows/{id}` for each tool →
  `active`.
- **Prevention:** keep tool workflows always active. **Important:** after
  `POST /api/v1/workflows` a new tool workflow is created inactive — `/activate` it immediately.

### AI-E3. Postgres Chat Memory fails with `Got unexpected type: constructor` on a manual INSERT into the history

- **Symptom:** after a third-party workflow (a proactive sender) wrote a message into the chat
  history table, **EVERY subsequent user message crashes the main bot** — the Memory node throws
  `Got unexpected type: constructor` (visible in the memory node's runData and in the Error
  Workflow). Before the first manual INSERT the bot worked.
- **Cause:** the record was made in the langchain **Serializable `toJSON()`** format —
  `{"lc":1,"type":"constructor","id":["langchain_core","messages","AIMessage"],"kwargs":{"content":…}}`.
  But the n8n Postgres Chat Memory loader (`mapStoredMessageToChatMessage`) understands only
  **StoredMessage**: `{"type":"ai"|"human"|"tool","content":"…",…}` (content at the TOP level,
  without `kwargs`). On `type:"constructor"` it throws an exception, and the whole agent round
  fails.
- **Fix:** a manual INSERT into the history table — only StoredMessage:
  `{"type":"ai","content":<text>,"tool_calls":[],"additional_kwargs":{},"response_metadata":{},"invalid_tool_calls":[]}`.
  Repair already-written broken rows with an UPDATE:
  `SET message=jsonb_build_object('type','ai','content',message->'kwargs'->>'content','tool_calls','[]'::jsonb,'additional_kwargs',COALESCE(message->'kwargs'->'additional_kwargs','{}'::jsonb),'response_metadata',COALESCE(message->'kwargs'->'response_metadata','{}'::jsonb),'invalid_tool_calls','[]'::jsonb) WHERE message->>'type'='constructor'`
  (the content is preserved).
- **Prevention:** (1) when you manually write a row that ANOTHER node reads (chat memory, a
  queue) — first read a real row written by the consumer itself and replicate its schema 1:1,
  don't guess. (2) An E2E test of the proactive must cover the FULL cycle with the consumer:
  send a message AND then reply as the user, making sure memory loads; "the message went out" is
  not enough. (3) Don't copy inherited code (a format from a previous version) without verifying
  — inherited ≠ correct.

Status: ⚠️ Confirmed in production, fixed (the write format + repairing broken rows with an
UPDATE).

### AI-E4. The recovery parser for tool-call leaks duplicates the write on a repeated marker

- **Context:** gpt-5.4 sometimes "leaks" a tool call as text — printing `to=functions.<tool>` +
  JSON arguments straight into the answer instead of a proper tool call (often with garbage
  tokens). Defense: a recovery node (Code) scans the agent's `output` for `to=functions.X`,
  parses the JSON and applies the write ITSELF (INSERT/UPDATE) — the tool itself never executed.
- **Symptom:** one action got written **twice** in a single round (two identical reminders in
  `proactive_queue`).
- **Cause:** in the leaked text the marker `to=functions.X` appeared SEVERAL times, but the JSON
  argument block — only once. The parser, for each marker, takes the nearest following `{…}` →
  both markers grabbed THE SAME JSON → two identical INSERTs.
- **Fix:** dedupe identical statements within one recovery round:
  `const uniq=[...new Set(stmts)]; if(!uniq.length) return []; return [{json:{sql:uniq.join('\n')}}]`.
  Safe: different legitimate writes differ in text and are preserved, idempotent repeats are
  collapsed.
- **Prevention:** design any parser of non-deterministic LLM output (a tool-call leak, free
  text) to be resilient to repeats/garbage — dedupe results or bind the JSON to a specific
  marker by position.

Status: ⚠️ Confirmed in production, fixed (dedupe in Parse Writes).

### AI-E5. The follow-up moved to tomorrow due to the hard floor `in_days≥1` — the "in the evening" promise wasn't kept

- **Context:** the agent agreed with the user "let's talk this evening," called the deferred
  follow-up tool (`schedule_followup`).
- **Symptom:** the message arrived not this evening, but the next day at 19:00. From the user's
  side — "promised and didn't write today."
- **Cause:** the tool's Build node had `in_days = Math.max(1, …)` — the lower bound of the range
  = 1 (tomorrow). Structurally the tool couldn't schedule for today; any "this evening" request
  rounded up to tomorrow. The tool itself fired correctly and wrote to `proactive_queue` — the
  bug was in the argument's range, not in the mechanics.
- **Fix:** see AI-R13 — drop the floor to 0, shift past time to the nearest window slot
  (otherwise the next day), the argument description and prompt must explicitly permit "today."
- **Prevention:** check the lower bound of any scheduling tool — does it allow "now/today"; a
  hard `Math.max(1, …)` on time units is a red flag if the scenario expects firing in the same
  period.

Status: ⚠️ Confirmed, fixed (see AI-R13).
