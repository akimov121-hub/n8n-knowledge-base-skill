# AI Agents

AI Agent, model selection, memory, RAG, tool workflows.

> Models and prices are accurate as of writing (mid-2026). Check current pricing and
> provider availability before choosing.

---

## ✅ Solutions

### AI-R1. Choosing a model for the task

| Task | Model | Reasoning |
|---|---|---|
| Client-facing agent, daily use | `gpt-5.4-mini` | Cheap, fast, smart enough |
| Complex agent with reasoning | `gpt-5.4` | Price/intelligence balance |
| Maximum quality, analytics | `gpt-5.5` | Flagship |
| Sensitive client data (Russian Federal Law 152-FZ) | Ollama (local) | Data never leaves the server |

**Default rule:** start with `gpt-5.4-mini`, upgrade only if quality isn't good enough.
(Check model names against the provider's current lineup — it gets updated.)

### AI-R2. Default stack

- **LLM:** `gpt-5.4-mini` (everyday tasks) / `gpt-5.5` (complex agents) / Ollama
  (sensitive data).
- **Memory:** Postgres Chat Memory (persistent, reliable).
- **RAG search:** Qdrant or Supabase (pgvector) — if vector search is available.
- **Embeddings:** OpenAI `text-embedding-3-small`.
- **Sensitive data:** Ollama locally, not the cloud.

### AI-R3. Parsing the openAi v2.1 node's response in JSON mode (text as a dict, not a string)

- **Context:** with `options.textFormat.textOptions.type = "json_object"`, the
  `@n8n/n8n-nodes-langchain.openAi` v2.1 node returns `output[0].content[0].text` **already
  parsed as a dict**, not as a JSON string. If a Code node downstream tries
  `JSON.parse(text)` — it falls into the fallback with empty fields, and the whole pipeline
  ends up empty all the way through.
- **Solution:** support both formats in the Code node:

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

### AI-R4. Reference-lookup tool for the agent via Postgres full-text search (RAG without vectors)

- **Context:** the agent needs on-demand access to a curated reference set — quotes, FAQ,
  regulations, letter templates, product descriptions, legal norms. Full vector RAG isn't
  always available: on some shared hosting the `vector` extension (pgvector) is **not
  installed** (check with: `SELECT count(*) FROM pg_available_extensions WHERE name='vector'`).
  For a corpus of tens to hundreds of records, semantic search isn't even necessary.
- **Solution:** the reference set is a plain Postgres table + full-text search. A
  `tsv tsvector` column + a GIN index. The agent's tool is a `toolWorkflow` node → a
  sub-workflow (`executeWorkflowTrigger` with a `query` input → Postgres `executeQuery`).
  The agent passes keywords via `$fromAI('query', …)`.

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

- **Why this way:** the corpus lives in the DB, the agent fetches the top 5 → the size of
  the reference base **does not affect** token count or response speed (unlike "the whole
  reference set in the system prompt"). Scales to hundreds of rows instantly.
- **Important (quality):** the tool only retrieves — accuracy is the responsibility of the
  data itself. Vet the reference set against an authoritative source. Require in the prompt
  that the agent use ONLY what the tool returned verbatim, and not make things up if the
  search comes back empty.
- **Evolution:** once pgvector / Qdrant becomes available — switch to semantic search
  without changing the tool's interface.

Status: ✅ Confirmed in production.

### AI-R5. Agent tool storage with human-readable numbers (memory, reminders, lists)

- **Context:** an agent tool (`toolWorkflow`) stores a user's personal records (memory,
  reminders, todos) and shows their numbers in chat. Requirements: gap-free numbering with
  reuse of freed-up numbers, in-place editing, minimal follow-up questions.
- **Pattern (4 parts):**
  1. **Number = `id`, gap-fill.** `save/set` inserts into the smallest free `id`
     (see postgres.md PG-R3). The displayed number is the real PK, so `delete`/`update`
     address it directly.
  2. **`update` operation (in-place edit).** A separate Route output: input `id` + a new
     full `content`, recompute derived fields (for semantic memory — re-embed),
     `UPDATE … WHERE id=… RETURNING id`. NOT delete+insert (otherwise the number moves to
     the end).
  3. **Display — plain number.** In the Code formatter: `'  '+r.id+'. '+text`, not `[id N]`.
     In the prompt — call it a "number", not an "id".
  4. **Autonomy per the prompt.** Perform delete/edit immediately if the record is
     unambiguously identified. Ask for clarification ONLY when ambiguous. An unknown `id` —
     `search`/`list` first.
- **Deployment:** editing a live `toolWorkflow` via PUT — preserve node ids, add
  credentials to every node, after PUT check activation with a retry (see rest-api-deploy.md
  DEP-E6). Test SQL on local Postgres before production.

Status: ✅ Confirmed in production.

### AI-R6. Long-term memory about the interlocutor "like a real person" (recall + capture)

- **Context:** the bot must remember facts about the person it's talking to across
  conversations and recall them on its own, without an explicit "remember" request
  (example: "going to the park with my dog Rex" → later the bot already knows who Rex is).
  Separate from conversation memory.
- **Pattern (2 independent mechanisms):**
  1. **Recall = "everything into context", no embeddings, no search.** Before the agent, a
     `Load Facts` node (Postgres) pulls ALL of the user's facts and concatenates them into
     one block:
     `SELECT COALESCE(string_agg('- ['||category||'] '||content, E'\n' ORDER BY category,id),'') AS facts_block ... WHERE telegram_id=...`.
     The block is inserted into the System Message via the expression
     `{{ $('Load Facts').first().json.facts_block }}`. Only this way does a fact surface on
     its own, without a search command: semantic search does NOT work here — the agent
     won't think to search for "Rex". No embeddings needed while facts per user number in
     the tens to hundreds.
  2. **Capture = the agent itself calls the `save/update/delete` tool** (based on AI-R5, but
     without embeddings). KEY POINT: one soft instruction in the middle of the prompt is NOT
     enough — the model focuses on answering in character and silently skips saving. A
     **hard directive at the very TOP** of the System Message is needed: "system action —
     memory, ALWAYS executed before the reply; if the message contains a new/changed fact
     about the interlocutor — you MUST call memory.save/update, otherwise it's an error."
     After that it calls it reliably.
- **Important nuance about novelty:** the agent judges "is the fact new" by the **dialog
  window**, not by the DB. A fact already stated within the window will NOT be saved again.
  Anything discussed BEFORE saving actually started working won't make it into the database
  on its own — enter it manually. To prove recall comes from memory and not from the
  window — clear the dialog history.
- **telegram_id — not from the model.** In a multi-user bot, `telegram_id` is passed into
  the tool via the expression `$('Normalize').first().json.telegram_id` (not `$fromAI`),
  otherwise the model can mix up users.
- **Not implemented (backlog):** a separate fact-extraction module on a mini model
  (a deterministic pass after the reply, writing outside the agent). A standalone technique
  for scenarios where missing a fact is costly: lead qualification, gathering profile data
  from correspondence, CRM enrichment, support with detail tracking. A deterministic pass
  (always fires, doesn't depend on the model's mood) is more reliable than "the agent
  decides for itself."

Status: ✅ Confirmed in production (the bot recalled a fact after fully clearing dialog
history — recall came only from fact memory).

### AI-R7. A consultant tool writes the result to the DB itself (deterministically), returning the raw answer to the agent

- **Context:** the main agent needs to save a sub-agent's structured result (a program, a
  plan, numbers) into the profile. If the agent itself does this via an update tool, the
  step is probabilistic and gets **silently skipped** (example: a program was generated,
  but `program_json` stayed NULL).
- **Solution:** the write is performed by **the consultant's own tool-workflow** as a side
  effect, independent of the LLM. After the Agent node: `Persist` (Code: parses the agent's
  `{output}`, builds `_sql`+`_params` based on `mode`) → `Save?` (IF `_save`) → `Save Field`
  (Postgres `executeQuery`, `query="={{ $json._sql }}"`,
  `queryReplacement="={{ $json._params }}"`) → `Return` (Code:
  `return [{ json: $('Agent').first().json }]` — returns exactly the agent's answer to the
  agent). One Persist covers several modes (PLAN→`nutrition_json`, CAFFEINE→`caffeine_mg/norm`).
  `telegram_id` — from the input `profile`.
- **Gotcha (Code node):** in `runOnceForEachItem`, returning `[{ json: { …array… } }]` failed
  with `A 'json' property isn't an object [item 0]`. Fix: **`runOnceForAllItems`** +
  `const out = $input.first().json;`.
- **Where NOT to do this:** large docs (program/nutrition_json) — the consultant writes
  them, and "save it yourself" must be removed from the main agent's prompt (otherwise
  double writes/race conditions). Small scalars (weight, age) — still handled by the agent
  via the update tool.

Status: ✅ Confirmed (isolated test PLAN+CAFFEINE, production).

### AI-R8. A proactive reminder must repeat the user's request VERBATIM

- **Context:** the agent sets a one-time reminder on direct request (the request text lives
  in `note`), and a separate cron workflow later generates the reminder text via a short
  mini-LLM prompt and sends it (`Build Prompt` node, `kind='reminder'` branch).
- **Gotcha:** a generic mini-LLM prompt ("write a warm message to the user") drifts into a
  **routine check-in question** about how the person is doing and does NOT mention the
  actual request. The user doesn't recognize the message as the reminder they ordered — it
  looks like a random nudge. (The tool WAS called and fired as intended — the bug wasn't
  "the tool didn't fire" but the generation text + up-to-an-hour delay from the hourly cron.)
- **Solution:** hard-code into the reminder-branch prompt: "This is a reminder for an
  EXPLICIT request. Start with the words 'You asked me to remind you…' and state the gist
  (`note`) VERBATIM. DO NOT turn this into a routine check-in question. If the gist is
  empty → a short neutral reminder." I.e., `note` must land in the text verbatim, not just
  serve as "mood" for free generation.
- **General principle:** when a deferred message is produced by a separate LLM step, the
  user's original intent must be carried through into the text verbatim — free generation
  around a "topic" loses recognizability for the recipient.

Status: ✅ Confirmed (short test).

### AI-R9. Deduplicating a bloated system prompt — a single canon instead of repeats (−25% tokens/turn)

- **Context:** a large agent's system prompt goes to the LLM on EVERY call — every extra
  line is paid for on every turn. A mature prompt accumulates duplicates over time: the same
  rule/prohibition gets rewritten in 3 sections "just to be safe." Example: the prompt had
  bloated to ~50,000 characters.
- **Solution:** collapse repeated blocks into **one canon** + references/deltas:
  - a tripled ban on tool/field names → one canon (in the "Hard Contract"), referenced by
    the other sections;
  - three copies of "when to call sub-agent X" → one routing section;
  - three "rewrite the reply in your own voice" → one shared block + short deltas by type;
  - three "degrade gracefully on tool error" → one;
  - ~60 listed names for gender detection → a rule + a short example list.
  Result: 49,967 → 37,613 characters (**−24.7%**) with no change to behavior or meaning.
- **Gotchas/limits:** (1) dedup is a prompt refactor — verify behaviorally (run scenarios),
  not just by diffing; (2) do NOT leave a changelog/"before-after" in the prompt body —
  that's tokens on every turn again; keep history in the version log. (3) the canon must
  live in a section the agent will actually read (top/contract), otherwise "see above"
  references are at risk.

Status: ✅ Confirmed (short test).

### AI-R10. Hide long structured bot output inside a Telegram expandable quote (`<blockquote expandable>`) via a sentinel marker + post-processing node

- **Context:** the agent sometimes returns a large structured block (a workout program, a
  meal plan, a long list) — in chat this becomes a "wall of text" that's awkward to scroll
  through. Telegram can collapse such a block into an expandable quote
  (`<blockquote expandable>…</blockquote>`), but the tag only works **with `parse_mode`
  HTML or MarkdownV2**, and the content inside must be properly escaped.
- **Solution (separating responsibility between the model and a deterministic node):**
  1. **The model wraps the block with a plain sentinel** instead of writing HTML itself: in
     the System Message — a rule that says "wrap the full program/plan in
     `⟪PROGRAM⟫ … ⟪/PROGRAM⟫`, and give 1–2 lines of summary before the marker." The LLM
     isn't asked to generate correct HTML (it would mess up the escaping) — only to place a
     cheap text marker.
  2. **A separate post-processing node before sending** (`Clean Output`, Code) finds the
     block by its sentinel and converts it into `<blockquote expandable>`, **escaping only
     the block's contents** (`&`→`&amp;`, `<`→`&lt;`, `>`→`&gt;`) — leaving the rest of the
     text untouched.
  3. **Backward compatibility:** no sentinel → text goes out unchanged. Existing replies
     aren't broken.
- **Important nuances:**
  - the pipeline's `parse_mode` must already be HTML (see telegram.md TG-R2) — the tag in
    plain mode arrives as raw text.
  - **Don't let the 4096-char splitter cut inside the block** — breaking a
    `<blockquote>` in the middle breaks the markup. If a block might exceed the limit — cut
    on block boundaries, not inside.
  - Choose an "exotic" sentinel (`⟪ ⟫`) to avoid colliding with ordinary user/model text.
- **General technique:** build any "model generates → node formats" task with a
  fragile target syntax (HTML, MarkdownV2, JSON) like this: the model places a simple
  marker, a deterministic node turns it into the target syntax with proper escaping. More
  reliable than trusting the LLM with escaping.

Status: ✅ Confirmed (renders correctly in Telegram).

### AI-R11. The agent must honestly state the coarseness of cron delivery timing, not promise an exact time

- **Context:** the agent sets a one-time reminder on request, but a separate cron delivers
  it with coarse granularity (**once an hour** — a deliberate design, not a "reminder
  manager"). The user asks "remind me in 10 minutes."
- **Gotcha:** the agent friendly repeats the user's phrasing — "Okay, I'll remind you in 10
  minutes." But the cron fires on the nearest hourly tick → it actually arrives within the
  hour. Promising an exact time **is misleading** (especially for intervals shorter than the
  cron step), even though the mechanism itself works fine.
- **Solution:** hard-code honesty about timing accuracy into the prompt: don't promise
  delivery to the minute and don't repeat "in N minutes" if N is less than the cron step; for
  a short interval — warn "I check roughly once an hour, I won't hit that exact minute, I'll
  remind you within the next hour" and still set it (or suggest tying it to a specific time
  if precision is needed); for a specific time/hour — confirm without "down to the second."
  Do NOT change the cron frequency for this.
- **General principle:** when execution is asynchronous and coarse in timing (cron, queue,
  batch), the accuracy expectation should shape the agent's wording — the phrasing must
  match the actual delivery granularity, not what the user wants to hear. Fix it with
  expectation-setting, not necessarily with frequency.

Status: ✅ Accepted for rollout (change live in production; final visual check pending from a tester).
Related to AI-R8 (same cron).

### AI-R12. Web chat on top of the SAME agent — cross-channel Telegram↔web history via a shared sessionKey=telegram_id

- **Context:** a Telegram bot agent gained a second channel (a web/Mini App chat). It needs
  to be **one conversation**, continuing across both channels, not two isolated bots with
  separate memory.
- **Solution:** build the new channel as an agent with **three shared components**: (1)
  **the same system prompt** (read from the same `prompts.<name>`), (2) **the same Postgres
  Chat Memory with the same `sessionKey`** = the user's `telegram_id` (not the web session's
  chat_id!), the same window, (3) **the same model**. Then both branches write/read one
  `<bot>_chat_history` table under one key → the history is unified: reply in Telegram — it
  shows up on the web, and vice versa.
- **Isolation without sharing nodes:** the web workflow itself is **separate**
  (`<workflow_name>`), doesn't touch the main bot (`<workflow_name>` stays byte-identical).
  What's shared is only the data (the memory table, the prompt row, the model credential),
  not the nodes. This protects the main bot from regressions when the web channel is edited.
- **`chat/history` ordering for the UI.** To give the frontend the **latest N** messages
  while keeping them **in ascending order** (oldest on top): `SELECT … ORDER BY id DESC
  LIMIT N`, then **reverse the array in Code**. `ASC LIMIT N` returns the beginning of
  history (not the tail); `DESC` without reversing gives the feed backwards. Filter out tool
  messages (`type='tool'` and assistant technical strings).
- **A deliberate boundary:** the main bot's tools (sub-agents/proactive/reminders) can be
  left OUT of the web agent for the sake of isolation — then the web chat is a "pure
  conversation" on shared memory, and tool-driven scenarios remain Telegram-only.
  Tool-parity is a separate task.
- **Mirroring** the second channel back into the Telegram conversation (so web replies show
  up in the chat with the bot too) — see telegram.md TG-R16; the web channel's login via the
  Login Widget — telegram.md TG-R14.

Status: ✅ Confirmed in production (ascending history order; `chat/send` writes both
messages into shared memory, the agent's reply arrives).

### AI-R13. The delayed follow-up tool must be able to fire TODAY: drop the hard floor, past time → nearest slot in the window, else next day (mirror the reminder tool)

- **Context:** the agent has a "schedule a follow-up" tool (`schedule_followup`): argument
  `in_days` + hour, a row in `proactive_queue`, delivery handled by a cron. Logically, "let's
  check in tonight / later today" is a follow-up for **today**.
- **Gotcha:** the tool had a hard floor `in_days = Math.max(1, …)` — it structurally could
  NOT schedule for today. Any "tonight" request silently slid to tomorrow (`due_at` =
  tomorrow 19:00). From the user's side: "promised to check in tonight — and didn't write
  today." The bug wasn't in the tool call (it fired fine) but in the lower bound of the
  range + in the argument description, which didn't allow "today."
- **Solution (three layers, all required):**
  1. **Remove the floor:** `in_days = Math.max(0, Math.min(30, Number(t.in_days)||0))`
     (0 = today).
  2. **Compute `due_at` mirroring the reminder tool, not "dumb +N days":** if the computed
     `due0 <= now()` (in the user's timezone) — shift to the **nearest whole hour within the
     cron's delivery window** (≤ the window's upper bound, e.g., 20:00); if it's already
     past the window — to the **next day** at the requested hour. `in_days≥1` (future) —
     unchanged. Implemented as a CTE `adj` in the Insert node's SQL, following the reminder
     tool's pattern.
  3. **The argument description + prompt must explicitly allow "today":** the tool
     description/`$fromAI` hint — "0 = today (tonight/later today), 1 = tomorrow, 2–3 = in a
     couple of days"; in the agent prompt — routing "tonight/later" → `in_days:0` (or
     `set_reminder` for a specific time), "tomorrow/in a couple of days" → `in_days≥1`.
     Without this the model still schedules for tomorrow even if the backend already
     supports today.
- **General principle:** the valid range of a scheduling tool must include "now/today," and
  the delivery-time computation must account for the requested moment possibly having
  already passed (shift to the nearest slot, not blindly to tomorrow). Both the range bounds
  AND the argument description AND the prompt must allow the near boundary — otherwise a
  "tonight" promise is structurally impossible. Reuse the "past time → nearest slot / next
  day" logic from the reminder tool, don't reinvent it.
- **Cap nuance:** if the proactive channel has a daily cap (e.g., soft proactive 1/calendar
  day), a same-day follow-up only fires if there hasn't already been a proactive message
  today. For guaranteed same-day delivery — combine with `set_reminder` (bypasses the
  window/cap).

Status: ✅ Confirmed (tests: in_days=0 in the morning → today 19:00; in_days=0 late/at night
→ tomorrow; in_days≥1 as before). Mirrors AI-R8/AI-R11 (same cron) and the reminder tool's
logic.

### AI-R14. "Proactive" tool calling: the phrasing "reply in words first, then call" suppresses the call — need "call it in the same turn, it's a background tool" + an example + "when in doubt, set it"

- **Context:** a product-coach agent must, in suitable situations, call a background tool
  on its own, without the user asking (`schedule_followup` — schedule a "how did it go"
  check-in), while giving the user a normal text reply.
- **Gotcha 1 — ordering:** the instruction "first reply in words, THEN quietly call" doesn't
  work. For a tools-enabled agent, the final text always comes LAST (ReAct loop:
  think→tool→…→final answer); "reply before the tool" is physically impossible, and that
  phrasing makes the call **optional** — the model gives its answer and drops the tool call.
  Symptom: on an explicit trigger ("let's see how the day goes"), the tool isn't called even
  though the prompt describes it as mandatory.
- **Gotcha 2 — softness:** "call it when appropriate… don't overdo it" is read by the model
  as "by default, don't call it." That's not enough for proactive (not directly requested)
  behavior.
- **Solution (prompt wording):**
  1. **Remove the false ordering:** not "reply first, then call," but "**call it in the SAME
     turn; this is a BACKGROUND tool — the user doesn't see the call; the order of the call
     and the text doesn't matter, what matters is not forgetting to call it**."
  2. **Tool = obligation, not an option** + a list of explicit trigger phrases verbatim
     ("let's see how the day goes," "we'll see," "we'll figure it out as we go") + a "**when
     in doubt — set it**" rule (better to overshoot than stay silent).
  3. **A concrete input→call example:**
     `"let's see how the day goes" → schedule_followup(in_days:0, when:evening, note:"…")`.
  4. For **anti-duplication and cancellation**, give the agent visibility into what's
     already scheduled: load open `proactive_queue` records into context every turn
     (`Load Pending` node → into `packet`) + a separate cancellation tool
     (`cancel_followup`, UPDATE done=true by id). Then the model neither piles up duplicates
     nor forgets to clear a touch-point once the user has reported back on the topic.
- **Verification:** before the fix, the tool wasn't called on an explicit trigger; after —
  it reliably fires both on setting (`schedule_followup`) and on cancellation via the user's
  report (`cancel_followup`), with no repeated questions from the evening cron tick.

Status: ✅ Confirmed (setting and clearing verified in production).

### AI-R15. Weekly deep dive into user data — a dedicated prompt + calibration by data volume + strict JSON for a collapsible UI

- **Task:** not "a recap of metrics for the period" but an analytical breakdown —
  connections between metrics, causes, dynamics, personalized recommendations. The source of
  the product-coach's "wow" feature.
- **Solution (pattern):**
  1. **A dedicated prompt** (a row in `prompts`, read by the workflow), not the main agent
     prompt — inherits the persona/voice/philosophy from the main prompt but adds analysis
     METHODOLOGY: explicit requirements to look for correlations (X→next-day Y), the main
     lever, dynamics + period-over-period comparison, anomalies, hidden risk, a "quiet win,"
     and ONE key non-obvious insight backed by numbers.
  2. **Rich input:** not only diary metrics but also the user's **correspondence** (history,
     filtered by `type<>'tool'`, last N messages, stripped of service headers) — gives the
     "why" behind the numbers; + profile/goal/facts + the previous period for comparison.
  3. **Calibration by data volume:** pass the number of days with data (`days_with_data`)
     into the prompt and tier the response: little (≤1) — an honest snapshot with no
     analysis + a call to fill in more data; 2–3 — a light breakdown, do NOT invent
     correlations/risks; ≥4 — full depth. Enforce strictly: "honesty over volume, omit empty
     sections." Otherwise the model fabricates connections out of thin air.
  4. **Strict JSON per schema** (headline / key_insight{title,text} / sections{...} /
     correlations / hidden_risk / quiet_win / overall / recommendations[{what,why,how}]) —
     parsed into the DB and rendered as collapsible cards. `key_insight.title` — a short
     LABEL (3–6 words), otherwise the model writes a whole sentence there and the "title"
     balloons.
  5. **Localization in the text:** explicitly require Russian metric names (not the English
     keys mood/energy) — otherwise the breakdown is peppered with English keys from the
     input data.
- **Generation gate:** only with ≥2 non-empty entries (otherwise there's nothing to analyze),
  a consent toggle, no report yet for this period (UNIQUE + NOT EXISTS). Once a week → no
  need to skimp on the model (large model, large prompt).

Status: ✅ Confirmed (breakdown quality on real data, graceful degradation on sparse data).
Related: AI-R6 (interlocutor memory).

### AI-R16. Hard length limit: an LLM can't count characters → margin in the target + deterministic trimming

- **Symptom:** you ask the model for "summary ≤950 characters," and it regularly overshoots
  (1000–1100). Telegram's caption limit is ≤1024, our working ceiling is 950 → the post
  fails to send / the button gets blocked.
- **Cause:** an LLM **cannot count characters precisely** — it treats "≤950" as "roughly
  that much," aiming for ~the limit and often overshooting.
- **Solution (two measures):**
  1. **A margin in the prompt's target:** ask for less than the limit — "aim for 820–880
     characters, STRICTLY ≤950, better shorter than right at the edge." The model lands
     under the threshold.
  2. **A deterministic guarantee after the model** (Code node), not trusting the LLM's
     count: if the result exceeds the limit — trim at a **sentence boundary**:

```js
let text=(p.post||'').trim();
if(text.length>950){const cut=text.slice(0,950);const m=cut.match(/[\s\S]*[.!?…](?=\s|$)/);text=(m?m[0]:cut).trim();}
```

     Guarantees ≤950 without cutting mid-word (only the trailing sentence gets trimmed,
     which is rare given the margin in the target).
- **General rule:** for ANY hard numeric constraint that an LLM produces (length, item
  count), don't rely on the prompt alone — add a deterministic check/trim in code. The
  prompt reduces the frequency, the code guarantees it.

Status: ✅ Confirmed.

### AI-R17. A cap on "N proactive touches per day" — a counter+date in the profile, not a single last_at; + spreading them out over time

- **Context:** proactive messages (product-coach) need to be capped per day. Originally it
  was 1/day via `last_proactive_at::date < today` — but that's hard-coded to 1 and doesn't
  scale to "up to 2."
- **Gotcha:** a single `last_proactive_at` timestamp doesn't store a counter — you can't
  build "2 per day" on it.
- **Solution:** add `proactive_day date` + `proactive_count int` to the profile. Selection
  gate: `(proactive_day IS DISTINCT FROM today OR COALESCE(proactive_count,0) < N)`. On
  sending (non-reminder), update: `proactive_count = CASE WHEN proactive_day=today THEN
  count+1 ELSE 1 END, proactive_day=today`. Compute the date in the user's timezone
  (`(now() AT TIME ZONE timezone)::date`).
- **Spreading out over time (important):** when fan-processing all candidates in one tick
  without an interval, two touches can land in consecutive hourly ticks back to back. Add
  `AND last_proactive_at < now() - interval '4 hours'` — then the N touches are spread out
  instead of arriving in a batch. Dedup by telegram_id in Build Prompt guarantees ≤1 message
  to a user per tick.
- **Don't cap explicit reminders** — they're a separate branch that doesn't touch the
  counter.

Status: ✅ Confirmed (migration done, gate enforced).

### AI-R18. Monetization gate before the agent: fail-open, after dedup, absolute counters produced by the gate

- **Task:** trial/plan limits for an LLM bot (trial N messages/M days, monthly limits,
  channel gates for voice/photo by plan, an 80% warning).
- **Pattern:** the chain `Plan Load (SELECT of counters, alwaysOutputData) → Plan Gate
  (Code: decides block/warn + ABSOLUTE new counter values) → IF → Plan Count (UPSERT) → IF
  Warn → Send Warn` is inserted **after dedup** of incoming messages (Telegram retries don't
  double-count) and **before** the buffer/agent.
- **Key decisions:** (1) **fail-open** — the entire Code gate is wrapped in try/catch, on an
  internal error the message passes through: the gate must not crash the bot;
  (2) the GATE computes the counters and passes absolute values — UPSERT just writes them
  (no increment race conditions in SQL, the monthly reset is a `plan_month` comparison);
  (3) UPSERT creates a new user's profile row with plan='trial', and `ON CONFLICT` does NOT
  touch `plan` or `trial_started_at` (COALESCE); (4) the `tester` plan (unlimited) for
  insiders — counters bypassed; (5) block messages — warm in tone, mention pricing, kept in
  one place (Code).

Status: ✅ Confirmed (both environments; logic covered by a unit test — 16 cases).

### AI-R19. Static user data goes into the system context, not into a mandatory tool call

- **Symptom/trigger:** the prompt required "mandatory first step — read_profile_tool" →
  nearly every agent turn started with an extra full context pass (~26k tokens) for data
  that was already known BEFORE the model was even called.
- **Solution:** the context-building node (in both the main bot and the web chat, nodes
  `Merge for Agent`/`Chat Prep`) appends a `[USER PROFILE]: {json}` block to systemMessage;
  heavy fields (full programs) and internal ones (plan counters) are excluded from the
  block — the tool remains for those. Prompt: the mandatory step was removed, the tool is
  now "after your own edits in the same turn, or for heavy fields." Place the block at the
  TAIL of system (keeps the static prefix cache intact; the profile changes rarely).
- **Effect:** on regular turns the tool isn't called (verified via runData); bonus — the
  tool-less agent (web chat) now sees the profile for the first time.
- **Observation:** on a DIRECT question ("tell me my data"), the model still plays it safe
  and calls the tool (cost = same as before, not worse). Pushing further via the prompt is
  optional, at the owner's discretion.
- **Pattern verification:** execution runData — the block is visible in inputOverride, and
  no tool call appears in the nodes.

Status: ✅ Confirmed.

### AI-R20. 152-FZ consent in a Telegram bot: a deterministic gate + an inline button, not via the LLM

- **Task:** explicit consent for processing personal data (including special categories)
  BEFORE any data collection, with the date/version recorded.
- **Pattern:** (1) Trigger updates += `callback_query`; branch `Is Callback?` FIRST (before
  access gates) → UPSERT `consent_at/consent_version` (COALESCE — pressing again doesn't
  overwrite the date) → answerCallbackQuery → greeting; (2) the `Consent Load → IF` gate sits
  AFTER dedup and BEFORE the limit counter (consent doesn't burn trial usage) — without
  consent, a fixed text goes out with an "I agree" button and a link to the policy, the
  message content isn't processed; (3) the consent text and its recording are done by
  deterministic nodes only (legal reproducibility), the LLM isn't involved; (4) the greeting
  after the button is conditional (`RETURNING name`): a new user gets onboarding, an
  existing one gets "continuing, please repeat your message"; (5) in web/miniapp — the same
  `consent_at` via a branch in build-SQL, the app shows an overlay.

Status: ✅ Confirmed.

### AI-R21. Conditional phase block in the prompt (trial/segment) without breaking the prefix cache

- **Task:** guide a segment of users (trial) along a "track" with extra instructions,
  without bloating the prompt for everyone else and without killing OpenAI's cache.
- **Pattern:** the block instruction is stored as a separate `prompts` row (editable without
  touching the workflow); the context-building node inserts it CONDITIONALLY
  (`plan==='trial'`) into system **between the static prompt and the profile** — ordering
  "from rare to frequent": [static prompt | phase block (changes once a day — only {DAY}) |
  profile (changes more often)]. For all other segments, system stays byte-for-byte
  unchanged. Milestone days are "a soft guideline, don't derail a live conversation"; hard
  phase events (day-7 offer) are NOT handled via the LLM but via a deterministic INSERT into
  the proactive queue when the phase starts (idempotent by note).

Status: ✅ Confirmed.

### AI-R22. Paid content generation in a webhook: gate → LLM-JSON → gpt-image → INSERT (quota isn't burned on failure)

- **Task:** a "pay to generate" feature (LLM + image) with plan-based limits, synchronous in
  the webhook router, without risking quota loss on a failed attempt.
- **Pattern:** node order = order of the money: (1) context in one SELECT (profile minus
  heavy fields + weekly/daily counters + the last N titles to avoid repeats); (2) a Code
  gate on plan limits (deny returns `ok:true, data:{denied,text}` so the client can show a
  soft toast instead of an error); (3) HTTP → chat/completions (`response_format:
  json_object`, a schema with all fields in the system message, including `theme` from a
  fixed list); (4) Code schema validation — `throw` HERE is cheap: no INSERT has happened
  yet, quota is intact; (5) HTTP → images/generations `output_format: "webp",
  output_compression: 80` — instead of a ~1.5MB PNG you get ~110KB base64, storable directly
  in a Postgres column; (6) INSERT with a queryReplacement array → RETURNING for the client
  response (include the photo in the create response but NOT in list — serve it separately
  via get).
- **Bot sending the raster image:** the client sends a JPEG data-URL → Code: strip the
  prefix + `prepareBinaryData` → Telegram `sendPhoto` multipart (`formBinaryData`).
- **Cost:** mini-LLM ≈ pennies + a medium 1024² image generator ≈ $0.04–0.05 per generation;
  limits (1–2/week by plan, unlimited with a daily ceiling for the top tier) are enforced by
  the SERVER, the client only displays them.

Status: ✅ Confirmed (live E2E, multiple generations).

### AI-R23. Vision for food: a mini model is no worse than the flagship and cheaper; recognition via HTTP + max_completion_tokens

- **Symptom/task:** recognizing food photos (macros from a plate). Which model to choose and
  how to call it.
- **Benchmark (not guesswork):** on real food photos with known macros (the reference —
  self-generated recipes) compared a cheap mini model against the previous flagship model.
  On calories, mini is no worse (−2…−13% off the reference for both), but it's **more
  detailed**, breaking the plate down into individual items with grams, where the flagship
  model lumps things into one dish. Mini is many times cheaper → mini chosen. The
  recognition cost per active user dropped by a large factor.
- **Call gotchas:**
  - the langchain `openAi` node (resource:image, analyze) sends `max_tokens` → fails on
    newer models with `'max_tokens' is not supported, use max_completion_tokens`. Solution:
    a **direct HTTP** `POST /v1/chat/completions` with `max_completion_tokens`, the image as
    an `image_url` data URL.
  - From a Telegram binary: a separate Code node `getBinaryDataBuffer(0,'data')` →
    `'data:'+mime+';base64,'+buf.toString('base64')`.
- **Food prompt — 3 explicit modes (otherwise numbers from screenshots get lost):** (1) photo
  of REAL food → `[FOOD]{items,total,note}`; (2) SCREENSHOT with data (a food diary,
  activity, a document) → a detailed transcription of ALL numbers on screen, in Russian
  (otherwise the model would just write "this is a screenshot" and the numbers never
  reached the agent — caught on real user screenshots); (3) anything else — an ordinary
  description. `[FOOD]` only in mode 1.
- **The recognition plan gate = same as the main bot** (some plans allow it, others don't);
  the client resizes photos to 1024/JPEG before sending.

Status: ✅ Confirmed.

### AI-R24. Long-term memory hygiene: TTL + weekly auto-review with a code guard

- **Symptom:** the memory agent accumulates "facts forever" with no distinction between
  permanent and temporary — stale promises and one-off events clutter the context and land
  in the wrong category.
- **Solution (3 layers):**
  1. **Write-time prompt:** strict categories (family = permanent facts + pets only;
     date-bound promises → the reminder tool, not memory; one-off incidents — don't write
     them; before saving, check for an existing entry → update, not a duplicate).
  2. **TTL:** an `expires_at` column; the memory tool accepts `ttl_days` (only for temporary
     circumstances); a nightly cleanup removes expired ones.
  3. **Weekly review** (Sunday cron + a manual webhook): a mini model looks at each user's
     dated facts → JSON `{delete[],update[]}`. **CRITICAL — the code guard is stricter than
     the prompt:** the applying Code node only allows delete for "events"/"context"
     categories, update only when the text actually changed (not a no-op); family/values/
     habits/work are untouchable at the SQL level. A dry run caught the mini model trying to
     delete a useful habit — without the code guard, the review itself would have destroyed
     memory.
- **Test before going live:** run the review in dry-run mode (only return the plan, don't
  apply it) on real data for several users — see what the LLM wants to delete.

Status: ✅ Confirmed (production run: a duplicate was cleaned up, useful data untouched).

### AI-R25. Guided agent sessions: launched via a trigger phrase from the UI + a protocol in the prompt + a proactive schedule

- **Task:** structured practices (CBT thought reframing, grounding, an evening wind-down)
  and periodic "sessions" with the agent — without a dedicated session backend.
- **Solution in three parts:**
  1. **Launch via a trigger phrase:** a button in the app sends a canonical phrase into the
     chat ("…let's work through an anxious thought (practice)"). Web client — auto-sends
     into its own chat; a Telegram Mini App can't send AS the user → the phrase goes to the
     clipboard + the mini-app closes into the dialog (fallback: show the phrase as text).
  2. **Protocol in the prompt, not in code:** for each practice — numbered steps and
     facilitation rules ("one step per message, wait for a reply, 5–10 minutes, no
     lecturing/diagnoses, close with a warm takeaway"). The LLM tracks session state itself
     via the chat history — no separate state machine needed.
  3. **A periodic session — via the existing proactive queue:** a weekly cron does an
     INSERT into the proactive queue (a kind marker, deterministic time) gated by
     plan/opt-in/consent and idempotent per week (NOT EXISTS … kind AND due_at >= start of
     week); the standard proactive workflow delivers it, the prompt describes the scenario
     by kind.

Status: ✅ Confirmed (inspired by a review of a third-party wellness app, Wysa).

### AI-R30. An action in the Mini App continued by the agent in chat (seeded into chat-memory)

- **Task:** a button in the Telegram Mini App should launch a conversational scenario (a
  practice) in the NATIVE chat with the bot. A Mini App (launched from the menu) cannot send
  a message as the user — only copy-paste (confusing) or close.
- **Solution:** an action in the mini-app's router (`practice/start`): (1) the bot sends the
  scenario opener into the user's chat via `sendMessage` (token from config), (2) that same
  message is **inserted into the agent's chat-memory** as an AI turn —
  `INSERT INTO <bot>_chat_history(session_id, message) VALUES (tid,
  '{"type":"ai","content":"…","tool_calls":[],"additional_kwargs":{},"response_metadata":{}}'::jsonb)`.
  The mini-app closes; the user sees the scenario already started in chat, replies — the
  agent picks up the next turn WITH CONTEXT (the opener is already in its memory).
- **Key point:** `memoryPostgresChat` (LangChain) reads the last N by `session_id`; a manual
  INSERT into the same table in the same format = seamless continuity without tokens and
  without fake Telegram updates. Verify the message format against a real live row (in our
  case `content` sits directly in the message, not under `data`).
- **Web version** drives the scenario through the built-in chat (regular `chat/send`) — no
  action needed.

Status: ✅ Confirmed (verified on-device, Telegram-only).

### AI-R26. A crisis protocol in a wellness agent's prompt (safety)

- **Task:** an AI habit coach may encounter a user in acute distress (thoughts of
  self-harm, etc.). It must neither ignore this nor play doctor.
- **Solution:** a "CRISIS AND SAFETY" block with priority over everything else: (1)
  recognize direct/indirect red flags; (2) a warm, serious response without panic or
  moralizing; (3) explicitly name free government hotlines (in Russia: EMERCOM
  8-495-989-50-50, national 8-800-2000-122, in immediate danger 112); (4) check for
  immediate danger and suggest calling/reaching out to someone close; (5) do NOT diagnose,
  do NOT minimize, do NOT redirect into workouts/nutrition talk. A caveat: "this is a rare
  scenario — don't hunt for a crisis in ordinary tiredness," to guard against false
  positives. Duplicate the contacts in the UI too (an in-app card with tel: links).
- **Testing a safety feature via an isolated LLM call** with a synthetic (fictional) signal,
  not on a real user. Result: all the numbers + the safety check were produced, with no
  diagnoses.

Status: ✅ Confirmed (tested in isolation).

### AI-R27. Verifying a prompt change on a SYNTHETIC profile + safe prompt deduplication

- **Task:** change a large agent system prompt without risk (behavior regression) and
  without writing to the DB / touching a real user.
- **Isolated behavior test (0 records written, 0 real users touched):**
  - A temporary workflow: Webhook → HTTP to the model → Code(extract). `system` = the NEW
    prompt + a synthetic profile block (`{"name":"Testina","gender":"F",...}` — explicitly
    fake, not a real person). `user` = a provocation message targeting the behavior under
    test. Read the response, delete the workflow.
  - **You must not** mint a real tester's session and send it into their live conversation
    — that's impersonation within an external system (the classifier rightly blocks it).
    Synthetic data is the safe substitute.
  - Example checks: "send it to my email" → an honest refusal + an alternative (the honesty
    block intact); logging food for a woman → refers to itself in the masculine grammatical
    gender (the gender rule intact).
- **Prompt dedup without losing quality:** a large prompt often accumulates DUPLICATE
  semantic blocks (e.g., "what I can/can't do" stated twice). Merging saves tokens on EVERY
  message with no behavior change — but only as a **union**: keep one block, fold the unique
  points of the second into it, drop nothing. Check the length delta and run the isolated
  test.

Status: ✅ Confirmed (synthetic test run passed).

### AI-R28. Robust LLM-JSON parsing in the write tool node: bracket repair + honest `throw` (not a silent `{}`)

- **Symptom:** a user dictated/photographed a meal, the bot replied "Logged your food for
  today…," but in the Mini App the "Nutrition per plan" screen showed empty buttons 1–5
  (not the "% of target" pill), the Nutrition tab was empty. The bot's execution was
  `success`.
- **Cause:** the write node parsed the agent's `metrics` argument as
  `try{JSON.parse(s)}catch{metrics={}}`. The model returned `food_log` with broken JSON — an
  array opened with `[` and closed with `}` instead of `]` (a missing `]`). `JSON.parse`
  threw → the catch **silently zeroed out ALL metrics for that call** → the write never
  happened, only a SELECT recent ran → the tool returned success → the agent concluded it
  had logged the food and **lied to the user**. A silent data loss under a success status.
- **Solution (two layers):**
  1. **Repair the JSON before handing it off** — `repairJson()`: a character-by-character
     pass with a bracket stack (accounting for strings/escapes), fixes mismatched/missing
     `]`/`}` (a closer of the wrong type → replace it with the correct one for the top of
     the stack; pad unclosed brackets at the end), strips a ``` code-fence wrapper. The
     missing `]` in `food_log` gets fixed.
  2. **Don't lie on failure** — if it still doesn't parse after repair: `console.log(raw)` +
     `throw` (the agent gets an error → honestly says "couldn't log it, please try again"),
     NOT a silent `metrics={}`. The valid path stays byte-for-byte the same; an empty
     `metrics` (read-only) — no throw.
- **Verification:** tested on real broken JSON from a user (several dishes get repaired, the
  entry exists), on valid JSON (unbroken), on garbage (throw), on empty (SELECT only).
- **General principle:** parse any LLM-supplied JSON argument in a tool node with repair +
  an honest error on failure; NEVER substitute `{}`/`[]` disguised as success — otherwise
  the agent reports success while there's no data. Related: AI-E4 (parser robustness against
  garbage), AI-R7 (deterministic write bypassing the LLM), AI-R22 (throw before the write —
  quota stays intact).

Status: ✅ Confirmed.

### AI-R29. Day-by-day proactive funnel: deterministic selection + free-form text from the LLM

- **Task:** guide a user through a paid trial (or onboarding) with touchpoints, rather than
  hoping the model will schedule its own follow-ups. Incident: trial guidance "stopped" —
  the agent messaged once, the user replied, the agent never scheduled a new follow-up, and
  the "silent for ≥3 days" threshold hadn't kicked in yet → silence during the most
  conversion-critical week. There was no systematic funnel at all — guidance rode entirely
  on the model's own judgment.
- **Pattern (who decides what):**
  - **SQL picks the touch day, not the LLM.** A branch in candidate selection:
    `plan='trial'`, day = `extract(day from now() - trial_started_at)`, `dn IN (1,3,5,6)`.
    Deterministic, reproducible, visible in the DB.
  - **The day's goal lives in the prompt, the text comes from the model.** Each day has its
    own goal (first step → engage with an untouched section → show progress → gently
    mention the subscription). Template blasts read as spam; a goal + profile/diary context
    produces live text.
  - **Protection against repeats — a log in the same queue:** after sending,
    `INSERT INTO proactive_queue (kind='trial', note='day:N', done=true)`, and in
    selection, `NOT EXISTS(... kind='trial' AND note='day:N')`. No separate table needed,
    idempotent regardless of how many scheduler ticks fire.
  - **Branch priority via `prio` + `UNION ALL`:** an explicit user reminder (0) → the funnel
    (1) → follow-up (2) → silence (3), plus dedup "one message per tick per user."
  - **The silence threshold depends on the stage:** 1 day on trial, 3 days once paid —
    `CASE WHEN p.plan='trial' THEN interval '1 day' ELSE interval '3 days' END`. A uniform
    threshold either loses trial users or annoys subscribers.
- **A mandatory window:** send only during daytime hours per the user's `timezone` + no more
  than once every N hours, otherwise the funnel piles up on top of regular touchpoints.

Status: 🟡 Draft (rolled out, selection verified in a dry run; confirmation pending on the
first live touches).

---

## ⚠️ Errors

### AI-E1. `pairedItem` breaks on the `@n8n/n8n-nodes-langchain.openAi` v2.1 node

- **Symptom:** in an expression after openAi v2.1, writing `$('Node_Name').item.json.field`
  returns `undefined`. The node downstream gets empty parameters (in Telegram sendPhoto this
  produces `Bad Request: there is no photo in the request`), even though the upstream node
  actually returned a value.
- **Cause:** the openAi v2.1 node doesn't propagate pairedItem metadata. `.item` relies on
  pairing to determine "which parent item produced the current one," and without it, it
  falls into undefined. This is especially painful in chains like
  `imgBB → openAi → sendPhoto`, where the URL is fetched via a cross-node reference.
- **Fix:** in ALL cross-node expressions where at least one openAi v2.1 node sits between
  the source and the consumer — use `.first()` instead of `.item`:
  - ❌ `$('Upload to imgBB').item.json.data.url`
  - ✅ `$('Upload to imgBB').first().json.data.url`
- **Prevention:** default to always writing `.first()` in cross-node refs — it's safe in
  every case; use `.item` only when pairing-based binding is explicitly needed.

### AI-E2. An agent tool (`toolWorkflow`) silently returns "unavailable" if the sub-workflow is deactivated

- **Symptom:** the agent replies "the tool is currently unavailable" in chat. Meanwhile the
  main bot's execution status is **success** (not error), so it's not obvious from the logs
  at a glance. In the runData of the problematic tool node you'll find:
  `{"response": "There was an error: \"Workflow is not active and cannot be executed.\""}`.
  The tool sub-workflows themselves **have no separate executions** of their own (called
  inline within the parent).
- **Cause:** the tool sub-workflow (with an `executeWorkflowTrigger`) was switched to
  `active=false`. When invoked via the AI Agent, such a workflow returns an error as a
  string inside `response`, and the agent interprets it as "the tool is unavailable." The
  parent execution stays success — the error never surfaces in the Error Workflow.
- **Fix:** reactivate the tool workflow: `POST /api/v1/workflows/{id}/activate`. For
  `executeWorkflowTrigger`, activation is safe — it doesn't trigger on its own.
- **Diagnosis:** don't trust the parent's `success` status. Open the main bot's execution
  with `includeData=true`, check `runData` for each tool node's `response` for the substring
  `There was an error`. In parallel, `GET /api/v1/workflows/{id}` for each tool → check
  `active`.
- **Prevention:** keep tool workflows always active. **Important:** after
  `POST /api/v1/workflows` a new tool workflow is created inactive — activate it
  immediately.

### AI-E3. Postgres Chat Memory crashes with `Got unexpected type: constructor` on a manual INSERT into history

- **Symptom:** after a third-party workflow (a proactive sender) wrote a message into the
  chat history table, **EVERY subsequent user message crashes the main bot** — the Memory
  node throws `Got unexpected type: constructor` (visible in the memory node's runData and
  in the Error Workflow). Before the first manual INSERT, the bot worked fine.
- **Cause:** the record was written in LangChain's **Serializable `toJSON()`** format —
  `{"lc":1,"type":"constructor","id":["langchain_core","messages","AIMessage"],"kwargs":{"content":…}}`.
  But n8n's Postgres Chat Memory loader (`mapStoredMessageToChatMessage`) only understands
  **StoredMessage**: `{"type":"ai"|"human"|"tool","content":"…",…}` (content at the TOP
  level, no `kwargs`). It throws on `type:"constructor"`, crashing the whole agent turn.
- **Fix:** manual INSERTs into the history table must use StoredMessage only:
  `{"type":"ai","content":<text>,"tool_calls":[],"additional_kwargs":{},"response_metadata":{},"invalid_tool_calls":[]}`.
  Repair already-written broken rows with an UPDATE:
  `SET message=jsonb_build_object('type','ai','content',message->'kwargs'->>'content','tool_calls','[]'::jsonb,'additional_kwargs',COALESCE(message->'kwargs'->'additional_kwargs','{}'::jsonb),'response_metadata',COALESCE(message->'kwargs'->'response_metadata','{}'::jsonb),'invalid_tool_calls','[]'::jsonb) WHERE message->>'type'='constructor'`
  (the content is preserved).
- **Prevention:** (1) when manually writing a row that ANOTHER node reads (chat memory, a
  queue) — first read a real row written by the actual consumer, and mirror its schema
  1:1, don't guess. (2) an E2E test for a proactive channel must cover the FULL cycle
  including the consumer: send a message AND then reply as the user, confirming memory
  loads; "the message was sent" is not sufficient. (3) don't copy inherited code (a format
  from a previous version) without verifying it — inherited ≠ correct.

Status: ⚠️ Confirmed in production, fixed (write format + repairing broken rows via UPDATE).

### AI-E4. The tool-call-leak recovery parser duplicates a write on a repeated marker

- **Context:** gpt-5.4 sometimes "leaks" a tool call as text — printing
  `to=functions.<tool>` + JSON arguments directly into the reply instead of a proper tool
  call (often with garbage tokens mixed in). Defense: a recovery node (Code) scans the
  agent's `output` for `to=functions.X`, parses the JSON, and applies the write itself
  (INSERT/UPDATE) — the actual tool never executed.
- **Symptom:** one action got written **twice** from a single turn (two identical reminders
  in `proactive_queue`).
- **Cause:** the leaked text contained the `to=functions.X` marker SEVERAL times, but only
  one JSON argument block. The parser grabs the nearest following `{…}` for each marker →
  both markers grabbed the SAME JSON → two identical INSERTs.
- **Fix:** dedup identical statements within one recovery pass:
  `const uniq=[...new Set(stmts)]; if(!uniq.length) return []; return [{json:{sql:uniq.join('\n')}}]`.
  Safe: distinct legitimate writes differ in text and are preserved, idempotent repeats
  collapse into one.
- **Prevention:** design any parser of non-deterministic LLM output (a leaked tool call,
  free text) to be robust against repeats/garbage — dedup the results or tie the JSON to a
  specific marker by position.

Status: ⚠️ Confirmed in production, fixed (dedup in Parse Writes).

### AI-E5. A follow-up slid to tomorrow due to the hard floor `in_days≥1` — the "tonight" promise wasn't kept

- **Context:** the agent agreed with the user "let's talk tonight," called the delayed
  follow-up tool (`schedule_followup`).
- **Symptom:** the message arrived not tonight but the next day at 19:00. From the user's
  side — "promised and didn't write today."
- **Cause:** the tool's Build node had `in_days = Math.max(1, …)` — the range's lower bound
  was 1 (tomorrow). The tool structurally could not schedule for today; any "tonight"
  request got rounded up to tomorrow. The tool fired correctly and wrote to
  `proactive_queue` — the bug was in the argument range, not the mechanics.
- **Fix:** see AI-R13 — remove the floor down to 0, shift a past time to the nearest slot in
  the window (otherwise the next day), the argument description and prompt must explicitly
  allow "today."
- **Prevention:** check the lower bound of any scheduling tool — does it allow
  "now/today"; a hard `Math.max(1, …)` on time units is a red flag if the scenario expects
  same-period firing.

Status: ⚠️ Confirmed, fixed (see AI-R13).

### AI-E6. Switching OpenAI models to a new generation breaks nodes with deprecated parameters — a bare-request smoke test does NOT catch this

- **Symptom:** after a bulk model swap to a new generation, the bot crashes on the first
  live message: `Unsupported parameter: 'temperature' is not supported with this model`;
  next in line was `max_tokens` ("Use 'max_completion_tokens'").
- **Cause:** newer model generations don't accept sampling parameters (`temperature`,
  `top_p`, penalties) or the old `max_tokens`. A "bare" smoke ping (model+messages only)
  passes fine — but production nodes carry options left over from the old model. The
  parameters live in DIFFERENT node types: `lmChatOpenAi` (options.temperature) and `openAi`
  (options.temperature + options.maxTokens) — scan ALL node types by the model name
  appearing in parameters, not just lmChat.
- **Fix:** when switching models — (1) go through EVERY node in every workflow where the
  model name appears and strip temperature/topP/penalties/maxTokens; (2) smoke-test WITH
  the production nodes' parameters, not a bare request; (3) check executions of all affected
  bots for the period after the switch.

Status: ⚠️ Gotcha (fix: nodes cleaned up, all PUTs ok).

### AI-E7. Reasoning models break the response parser: the text is NOT in `output[0]` but in the item with `type='message'`

- **Symptom:** the app "hung" during generation. The n8n execution was **success in
  17–21s**, but the parser node returned `{header:"",post:""}` → an **empty post** got saved
  to the DB → the frontend polls for the result until it times out. No error anywhere: both
  n8n and the model reported "success."
- **Cause (the TRIGGER was switching to a reasoning model, not "it just broke"):** the
  openAi v2.1 node (Responses API) on reasoning models returns `output` as a **list of 2+
  items**: `output[0].type='reasoning'`, the text is in `output[1].type='message'` →
  `content[0].type='output_text'`. The parser hard-coded
  `resp.output?.[0]?.content?.[0]?.text` → `undefined` → an empty object → empty fields.
  **How it was found:** a sibling node on a mini model (`reasoning_tokens: 0`) worked fine,
  while the node on a reasoning model (`reasoning_tokens: 355`) broke. I.e., exactly the
  nodes that had been switched to a reasoning model broke (the change was made **manually
  in the UI**, bypassing the config generator → it still showed the old model = drift).
  **Marker:** `usage.output_tokens_details.reasoning_tokens > 0` → the model is in reasoning
  mode → the response envelope is different.
- **Solution — look for the message item, not a fixed index:**

```js
var _o = resp.output || [];
var _m = _o.find(x => x && x.type === 'message') || _o[_o.length-1] || {};
var _c = _m.content || [];
var _i = _c.find(x => x && (x.type === 'output_text' || x.text !== undefined)) || _c[0] || {};
var _t = _i.text;   // a string, or already an object (json_mode)
```

  Works on non-reasoning models too (there message is the only element). Apply in both Code
  nodes and Set expressions.
- **Scope (important):** the pattern is copy-pasted across all AI branches — it affected
  more than a dozen nodes across several production workflows (preprocessor, editor,
  co-writer, text edits, art director/image_prompt, caption generation, time parser). **On
  this kind of breakage, check ALL parsers at once** (`grep 'output?.[0]'` across the
  export), not just the one you happened to notice.
- **Prevention (the root cause of the "silent hang"):** an empty AI result is a **failure**,
  not "still computing." The frontend/consumer must distinguish: if the backend responded
  but the text is empty — surface an error to the user immediately, no polling until
  timeout.
- **Diagnosis:** "hung" + success executions at ~20s → look not at timeouts but at **what
  the parser actually saved**; then inspect the structure of `output` from a live response
  (`executions/{id}?includeData=true`).
- **Rule going forward:** switching a model generation/type = **a change in response shape**,
  not just quality. After moving to a reasoning model — re-check ALL parsers on that branch
  (`grep 'output?.[0]'` across the export) and run one live call per AI branch. Editing the
  model in the UI, bypassing the config generator, creates drift. Related entry — AI-E6
  (a generation switch breaking nodes via deprecated parameters).

Status: ✅ Fixed and verified live.

### AI-E8. The agent talks about ITSELF using the user's grammatical gender

- **Symptom:** a male bot persona writes to a woman about its own actions using feminine
  verb forms in Russian: "Name, **logged (fem.)** your food today" (should be "logged
  (masc.)"). Caught by a tester.
- **Cause:** the persona's gender is never explicitly stated anywhere in the prompt, while
  the profile field is described as "`gender` — M/F. For calculations and **addressing the
  user in the correct grammatical gender**." The LLM interprets this broadly: it adjusts
  ALL speech to match the interlocutor's gender, including verbs about itself. Russian
  first-person grammatical gender isn't "style" — it's grammar the model borrows from the
  interlocutor's context.
- **Fix:** an explicit hard rule in the persona block + a clear split from the profile
  field:
  - "You are male. You speak about YOURSELF ONLY in the masculine gender, always,
    regardless of the interlocutor's gender: logged, checked, glad. Never carry the user's
    gender over onto yourself."
  - "`gender` affects ONLY words about the USER: you (fem.) logged, you (fem.) completed."
  - Add a concrete counter-example from the actual bug ("Name, logged (fem.)…" → "Name,
    logged (masc.) your food…") — the model responds better to these than to abstractions.
- **Verification without intruding on someone else's dialog:** a temporary workflow with the
  same system prompt + a SYNTHETIC female profile block + a single LLM call on the
  production model. Don't mint a real user's token and don't write into their live
  conversation (that's impersonation).
- **Lesson:** any role/grammatical persona property (gender, age, "I") must be stated
  explicitly — what's "obvious from the character's name" isn't binding for the model.

Status: ✅ Fixed and verified.

### AI-E19. Decoupling a subsystem — the access gate keeps living on in NEIGHBORING workflows (silently disables features)

- **Symptom:** after decoupling a product from the old access system, everything works —
  login, payment, chat. But a separate feature (proactive messages) silently stops firing
  for some users, with no errors anywhere.
- **Cause:** only the obvious `Check Access` nodes in three workflows were updated, but the
  condition `EXISTS (SELECT 1 FROM bot_access WHERE ... AND active)` remained in the
  proactive workflow — there it's buried inside a large candidate-selection SQL query, not
  in a node with a telling name. As long as rows in `bot_access` keep getting created by
  inertia, nothing breaks; once they stop, the feature dies without a single error.
- **Fix:** after decoupling/replacing a gate, grep for the old table name **across all
  workflows**, not by node names: `grep -l 'bot_access'` over the dump, or an SQL query
  against `workflow_entity.nodes::text LIKE '%bot_access%'`. Bring every match in line with
  a single access rule.
- **Rule:** the access gate must live in ONE place (a DB view/function or a single
  sub-workflow). Duplicating the condition across workflows guarantees drift the next time
  the access model changes.

Status: ⚠️ Gotcha (found and removed; see AI-R29).

### AI-E9. An echo bot loses the owner notification when message delivery to the user fails

- **Symptom:** a support bot (plain Webhook+HTTP, Mirror Echo pattern) — executions fail on
  the `Send Reply` node with `Bad Request: chat not found`. This fails the ENTIRE scenario,
  and the owner notification (`Notify Anton`), which sits AFTER sending the reply to the
  user, **never fires — the bug report is lost**.
- **Cause:** 1) "chat not found" with a valid private `chat_id` means the user hasn't
  pressed `/start` with the bot yet (or blocked/deleted it) — a routine production
  situation, the bot can't initiate a conversation. 2) The Telegram HTTP nodes have no
  `onError` → any delivery failure crashes the whole chain; the owner notification, tied to
  the success of the previous node, dies along with it.
- **Fix:**
  - Set `onError: continueRegularOutput` on ALL outbound sends (`sendMessage` to the user,
    the start message), so a failed delivery doesn't crash the scenario.
  - **The owner-notification branch must not depend on the success of the reply to the
    user.** Either `onError=continue` on `Send Reply` (so the chain continues to Notify), or
    fan them out in parallel from a shared node. Notify should read data from the parser
    node (`$('Parse Reply')`), not from the send step — that way it always gets through.
  - The root cause of "chat not found" can only be fixed by the user: pressing `/start`. But
    the bug notification should get through regardless.

Status: ⚠️ Caught in live executions and fixed.

### AI-E20. A prompt embedded in an HTTP node's `jsonBody`: over-escaping crashes the node with `invalid syntax`

- **Symptom:** the system prompt was swapped inside an HTTP node (a direct model call,
  `jsonBody` = `={{ JSON.stringify({...}) }}`) — the workflow saved without errors, the API
  returned 200, but on the first actual run the node fails with `invalid syntax`, and the
  bot goes silent for users.
- **Cause:** the prompt lives inside a JS string in double quotes. In an API dump, quotes
  show up as `\"`, making it easy to "automatically" escape them again (`\\"`), producing a
  broken expression. The error isn't caught by n8n's validation or by saving — only by
  actually running it.
- **Fix:** escape EXACTLY once: `"` → `\"`, a newline → `\n`. Also check in advance that the
  prompt text contains no `{{` / `}}` — n8n will interpret them as its own expression syntax.
- **Mandatory step:** after editing a prompt — a **live run**, not just a `GET` check (the
  text may look correct in the dump while the expression is actually broken). For webhook
  bots with a secret: POST to the webhook with the right header and check
  `status='success'` in `execution_entity`, otherwise the edit "went through" while the bot
  is actually dead.
- **Safety net:** before editing, save the original `jsonBody` to a file — a rollback then
  takes a single PUT.

Status: ⚠️ Gotcha (hit it and fixed it).

### AI-E10. The agent lies about "yesterday": the dates are in the data, but the model subtracts them wrong

- **Symptom:** the agent states EXACT numbers (calories, sleep, mood match the DB to the
  digit) but attaches them to the wrong day: "after yesterday's lack of sleep" — even though
  yesterday's sleep was 7 hours and the bad night was actually the day before. The user
  catches a fact that never happened and stops trusting the whole breakdown, including the
  correct numbers.
- **Cause:** the context includes the current time (`[YYYY-MM-DD HH:mm]` in the message
  header) and diary entries with a `date` field in ISO format. The model computes "today
  minus the entry date" ITSELF — and gets it wrong at this step, especially when the sample
  contains several similar consecutive days. The data isn't hallucinated — it's a real day,
  just the wrong neighbor.
- **Fix — via the prompt, no pipeline changes:** a rule stating "before saying
  'yesterday/the day before/on the weekend' — subtract the entry date from the current date
  and check the word; if the day matters for the meaning (a bad night, a slip, a record) —
  name the date or weekday: 'on Saturday, 07/25'; if there's no entry for the needed day —
  say so, don't substitute a neighboring one."
- **Why this works:** requiring a named date makes the error visible to both sides and
  forces the model to perform the subtraction explicitly rather than "by eye." Confirmed:
  after the fix, all numbers and days in the reply matched the DB, and "yesterday's" lack of
  sleep was correctly renamed to "recent."
- **Alternative, if the prompt fix isn't enough:** feed ready-made labels into the context
  (`the day before yesterday (Sat, 07/25)`) instead of bare dates — then there's nothing to
  compute. More expensive (requires editing context assembly everywhere), so start with the
  prompt.
- **How to catch it:** export the agent's replies and grep for relative words
  (`yesterday|day before|the other day` — Russian equivalents), then cross-check the
  mentioned numbers against the table for the corresponding dates. An off-by-one-day error
  shows up immediately.

Status: ⚠️ Gotcha → ✅ fix confirmed.

### AI-E11. A male agent talks about itself in the feminine gender, mirroring the interlocutor

- **Symptom:** in a conversation with a woman, the agent suddenly talks about its OWN
  actions using feminine grammatical forms (Russian): "Name, logged (fem.) your food today"
  (instead of the masculine form). The persona breaks, looking like an engine glitch.
- **Cause:** the model agrees Russian verb gender with the nearest animate referent in
  context — which turns out to be the woman, whose name and gender are mentioned nearby.
  It happens roughly once per ~150 replies, so it's nearly impossible to catch in manual
  testing.
- **Fix:** an explicit rule in the prompt: "regardless of the interlocutor's gender, talk
  about your own actions in the masculine gender: logged, calculated, updated, understood;
  don't mirror your own gender onto the interlocutor's gender. Address the user in THEIR
  gender (the profile's gender field)."
- **How to catch it:** grep the agent's replies for feminine first-person forms (Russian:
  `записала|посчитала|обновила|поняла|отметила`) — for a male agent, any match is a defect.

Status: ⚠️ Gotcha (rule added).

### AI-E12. The agent doesn't sense that a conversation paused overnight: dialog history has no timestamps

- **Symptom:** in the evening the two discussed plans "for tomorrow," the person writes in
  the morning — and the agent carries on as if it's still yesterday's reply: "your wind-down
  for today is basically done, leave work until morning," "you'll tackle the next bug
  tomorrow." Except that morning IS already that "tomorrow." From the outside it reads as
  inattentiveness: the person is living in the new day, while the counterpart is still
  finishing yesterday evening.
- **Cause:** Postgres Chat Memory stores only role and text (`StoredMessage`), WITHOUT
  timestamps. To the model, the entire history looks like one continuous conversation
  happening "just now." The current time only appears in the header of the new message —
  and by itself that doesn't solve the problem, because the model doesn't cross-reference it
  against the history unless asked to.
- **Fix via the prompt:** a rule stating "the dialog history carries no timestamps; the only
  reference point is the timestamp in the header of the current message; hours, a night, or
  days may have passed since the previous reply. If a new day has started (by date or time
  of day) — plans 'for tomorrow' from the earlier conversation have become plans for today;
  don't suggest waiting until morning when morning has already arrived."
- **Alternative (more expensive):** write the time into the saved message text itself, or
  inject `[date time]` into the history when assembling context — then the model has
  nothing left to guess. Requires editing the memory pipeline, so start with the prompt.
- **Related gotcha:** AI-E10 — there the model mis-subtracts diary dates; here it fails to
  notice the gap between turns. Both stem from the LLM having no innate sense of time:
  anything not fed explicitly gets filled in by guesswork.

Status: ⚠️ Gotcha (rule added).

### AI-E13. An expensive LLM feature without a counter = a hole in the unit economics

- **Symptom:** food-photo recognition was allowed on both the trial and the premium plan,
  but the limit **was only enforced for premium** (100/month). A trial user could snap as
  many photos as they wanted over 7 days.
- **Cost of the issue:** a vision request ≈ $0.0023 (a 1024px image, detail high, ~1215
  input tokens). Trial profit is +$0.29 — meaning 60 photos in the trial week wiped it out
  entirely, and 100 pushed it into the red.
- **Second half of the problem:** a neighboring route "estimate macros from a name" had no
  gate at all — neither plan nor counter. If a session token leaked, it could be hammered in
  a loop.
- **Solution:** (1) a limit on every plan, not just the top one:
  `LIMITS = { trial:5, basic:10, advanced:30, premium:100, tester:null, super:null }`, an
  unrecognized plan is treated as the strictest; (2) on the cheap-but-open route — a daily
  technical ceiling (30/day), not shown in the UI: plenty for a real human, an automated
  loop hits the wall immediately.
- **Rule:** when adding an LLM feature, answer two questions right away — how much does one
  call cost, and what stops it from being called a thousand times. If the answer to the
  second is "nothing" — a counter is mandatory, even for a cheap call.
- **Separately:** cross-check the model used in the workflow against the model used in the
  economics calculation — a mismatch can inflate or deflate the estimated cost several times
  over.

Status: ✅ Fixed (both API workflows).

### AI-E14. The model puts the wrong thing into a structured field

- **Symptom:** the user asked to adjust one specific workout. The trainer sub-agent returned
  `PROGRAM` mode, where the `exercises` field (a list of exercises) ended up containing
  instructional phrases: "Keep the 1 km easy run," "Remove 3 rounds of squats." The day's
  title came out as "Before the run," and the numeric duration field didn't match the
  described content.
- **Cause:** the prompt described WHAT to put in the field but didn't forbid putting
  something else there, and didn't require cross-checking the numeric field against the
  content. On top of that, the "assemble a program" scenario and the "adjust one workout"
  scenario weren't distinguished — the model packed the edits into the program structure.
- **Fix in the prompt:** (1) an explicit ban on instructional verbs in the list, with
  wrong/right examples; (2) the day-title format is a muscle group or split day, not a
  moment in time; (3) a requirement to sum warm-up + main work + cooldown and cross-check
  against the duration number before returning; (4) a separate section "when NOT to use
  PROGRAM mode" — single-workout edits are returned as plain dialog text.
- **General rule:** for every structured field in a prompt, it helps to give a "wrong /
  right" pair based on a real miss, and to explain WHAT the user will actually see in the
  UI. An abstract "list of exercises" gets interpreted more broadly by the model than
  intended.

Status: ✅ Applied, pending verification on live dialogs.

### AI-E15. The model sums numbers its own way: give the total to the server, leave the addends to the model

- **Symptom:** a user's "Fluids" metric was under-reported and "sometimes worked, sometimes
  didn't." Digging into executions: the agent sent a breakdown
  `{water:3000, coffee:400, other:250}` and a total `water: 3000`, and later another
  breakdown `{water:3000, coffee:600}` with the total again `3000`. So the breakdown was
  tracked correctly, but only water made it into the total.
- **Cause:** the prompt said "`water` — ANY liquid consumed," but the word `water` carries
  its own everyday meaning, and the model repeatedly defaults to that meaning instead of the
  instruction's definition. On top of that, it was asked to do arithmetic (sum the
  breakdown) — an operation whose errors show up in neither the logs nor the `success`
  status.
- **Separately:** sometimes a drink was skipped entirely — "two drip coffees plus a small
  energy drink" produced `{caffeine:320}` with zero ml logged, while the agent's reply
  cheerfully reported having logged both drinks. The model's own account of its actions is
  not proof the action happened.
- **Fix:** take the total away from the model entirely. It only maintains the breakdown by
  type (`liquid_mix`), and the SQL in the write node computes the sum. Manual entries from
  the app aren't overwritten: they live as the difference between the saved total and the
  sum of the breakdown.
- **Migration gotcha:** old days have a total but no breakdown. Blindly doing "total =
  manual + breakdown sum" double-counts water. Split into three branches: a breakdown
  already exists → manual = total − breakdown; no breakdown, but the model sent a "water"
  key → manual = 0 (its breakdown IS the whole day); no breakdown and no water in what was
  sent → manual = the entire old total, drinks get added on top.
- **How to test changes like this:** generate the SQL locally, run it against production
  data inside a one-off `BEGIN … ROLLBACK` workflow across several scenarios (an old day, an
  old day with no water sent, an app tap, a clean day, a manual add), compare against
  expected numbers, delete the workflow. Cheaper than catching a regression a week later
  from a complaint.
- **Rule:** give the model facts and addends, give code the arithmetic and totals. Any
  number the model sums itself will eventually drift silently from reality.

Status: ⚠️ error confirmed in executions; fix applied, SQL verified across 5 scenarios in a
rolled-back transaction, 🟡 pending confirmation on live dialogs.

### AI-E16. "the last N messages" ≠ "for the period": the context window is sliced by time, not by count

- **Symptom 1 (weekly report).** For a user who barely writes in chat, the weekly report
  ended up built from events three weeks old — the model placed a quit-smoking attempt and a
  relapse on the wrong days of the week. For an active user, the same report looked
  flawless.
- **Cause.** The conversation was pulled as `ORDER BY id DESC LIMIT 50` — with no date
  boundary. For an active user this is exactly the current week; for a quiet one, over a
  month. Worse, the prompt-assembly code used a regex to **strip the date label** out of
  every message, while the block was still labeled "CONVERSATION FOR THE WEEK." The model
  did exactly what it was told.
- **A separate detail:** most of the messages in that window were unanswered proactive pings
  from the bot itself ("haven't talked in a while"). For a quiet user, the chat channel
  carries almost no signal but crowds out everything else.
- **Symptom 2 (support bot).** The bot asked the same three diagnostic questions eight times
  in a row, even though the person had answered all of them. Only the current message was
  sent to the model — there was no history at all.
- **Fix.** Always bound the window by a time period: a report window = the report's week,
  the support window = 12 messages and no older than 3 days. Do NOT strip out timestamps:
  without them the model can't tell Tuesday from last month. If there are no messages for
  the period — tell the model that explicitly ("no chat messages, build from the diary")
  instead of feeding it a different period under the right label.
- **How to catch it.** Count, per user: total messages vs. how many fall inside the window.
  A ratio like "87 total / 0 for the week" is the diagnosis right there.
- **Rule.** A block header in the prompt is a promise. If it says "for the week," the data
  must actually be for the week; otherwise the model faithfully lies, per your own
  instructions.

Status: ⚠️ both errors confirmed in executions; fixes applied and verified (support — via a
live run, the report — by counting the window across several users).

### AI-E17. The bot says "I see the photo" without having vision

- **Symptom:** a user sends a screenshot, the bot replies "Thanks, I see the photo 🙏" and
  immediately asks "what exactly is on screen — a blank screen, a loading spinner, or an
  error?" This repeats across several screenshots in a row.
- **Cause:** only the string `[photo attached]` was sent to the model — that's it. The image
  was never downloaded or analyzed at all. The phrase "I see the photo" lived in the prompt
  as a politeness and turned into an outright lie.
- **Fix:** a vision pipeline (getFile → download → data URL → a vision request → describe it
  into the message text) plus a prompt rule: rely on the analysis, don't ask "what's on
  screen"; if the analysis fails — say so honestly and do NOT claim to see it.
- **Found along the way:** replies contained `**bold**` markup, but `parse_mode` on
  `sendMessage` wasn't set — the asterisks went out to users as literal characters. Cleaned
  up in the response parser, and markdown is now banned in the prompt.
- **Rule.** Polite phrasing in a prompt must not claim a capability the bot doesn't have.
  "I see," "I checked," "I looked" — either they're true, or they don't belong in the prompt.

Status: ✅ verified with a live run on a real screenshot — the bot recognized the login
screen and guided the user through login instead of listing questions.

### AI-E18. JSON repair delivered the report, but gutted: fix the cause, not just the symptom

- **Symptom:** for two out of five users, the weekly breakdown arrived nearly empty — a
  headline, but no content. Execution status was `success`, delivery went through.
- **Cause, in two layers.** First: the node had no output ceiling set, and a long breakdown
  got cut off mid-way. Second, sneakier: our own auto-repair padded in the missing brackets
  — and nested `sections`, `correlations`, and `recommendations` INSIDE `key_insight`. The
  JSON became valid, but the structure was wrong. The app reads those fields at the top
  level and rendered nothing.
- **Fix:** `maxTokens` set explicitly (8000), plus
  `textFormat: {textOptions:{type:'json_object'}}` mode — the API itself guarantees
  validity. The repair step remains a safety net, not the actual mechanism.
- **A trap when enabling json_object:** the node returns an already-PARSED object, not a
  string. The assembler code, which did `JSON.parse`, got `[object Object]` and silently
  discarded every report. Caught in a dry run before rollout; the assembler was updated to
  accept both forms.
- **Rule:** auto-repair of data must be noisy. If it fired — that's an incident, not
  business as usual: log it and fix the cause. "Valid JSON" and "correct structure" are two
  different things — check the second one.
- **How to verify:** a dry run on the REAL context of the affected users (pulled from an old
  execution), run all the way through to the assembler, without sending anything to
  Telegram.

Status: ✅ Verified end-to-end on both affected users' contexts, top-level structure
correct, no nesting.
