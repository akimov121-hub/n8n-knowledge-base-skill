# AI Agents

AI Agent, model selection, memory, RAG, tool workflows.

> Models and prices are current as of writing (mid-2026). Check current pricing and
> provider availability before choosing.

---

## ✅ Solutions

### AI-R1. Choosing a model for the task

| Task | Model | Rationale |
|---|---|---|
| Client-facing agent, daily use | `gpt-5.4-mini` | Cheap, fast, smart enough |
| Complex reasoning agent | `gpt-5.4` | Price/intelligence balance |
| Maximum quality, analytics | `gpt-5.5` | Flagship |
| Sensitive client data (Russian Federal Law 152-FZ) | Ollama (local) | Data never leaves the server |

**Default rule:** start with `gpt-5.4-mini`, upgrade the model only if quality isn't good
enough. (Check model names against the provider's current lineup — it keeps changing.)

### AI-R2. Default stack

- **LLM:** `gpt-5.4-mini` (everyday tasks) / `gpt-5.5` (complex agents) / Ollama
  (sensitive data).
- **Memory:** Postgres Chat Memory (persistent, reliable).
- **RAG search:** Qdrant or Supabase (pgvector) — when vector search is available.
- **Embeddings:** OpenAI `text-embedding-3-small`.
- **Sensitive data:** Ollama locally, not the cloud.

### AI-R3. openAi node v2.1 parses the response as JSON (text as a dict, not a string)

- **Context:** with `options.textFormat.textOptions.type = "json_object"`, the
  `@n8n/n8n-nodes-langchain.openAi` v2.1 node returns `output[0].content[0].text` **already
  parsed as a dict**, not a JSON string. If the downstream Code node tries `JSON.parse(text)`
  on it — it falls into the fallback with empty fields, and the whole pipeline produces
  emptiness all the way to the end.
- **Solution:** in the Code node support both formats:

```js
const resp = $input.first().json;
const _t = resp.output?.[0]?.content?.[0]?.text;
let parsed = null;
if (_t && typeof _t === 'object') {
  parsed = _t;                              // new path: dict is already parsed
} else if (typeof _t === 'string') {
  try { parsed = JSON.parse(_t); } catch(e) { parsed = null; }
}
if (!parsed || typeof parsed !== 'object') {
  parsed = { /* fallback with default fields */ };
}
```

Status: ✅ Confirmed in production.

### AI-R4. A reference-lookup tool for the agent via Postgres full-text search (RAG without vectors)

- **Context:** the agent needs on-demand access to a curated reference set — quotes, FAQs,
  regulations, letter templates, product descriptions, legal norms. Full vector RAG isn't
  always available: on some shared-hosting plans the `vector` extension (pgvector) is **not
  installed** (check with: `SELECT count(*) FROM pg_available_extensions WHERE name='vector'`).
  For a corpus of tens to hundreds of records, semantic search isn't even needed.
- **Solution:** the reference set is a plain Postgres table plus full-text search. Column
  `tsv tsvector` + a GIN index. The agent's tool is a `toolWorkflow` node → a sub-workflow
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

- **Why this way:** the corpus lives in the DB, the agent pulls the top 5 → the size of the
  reference set has **no effect** on tokens or response speed (unlike stuffing "the whole
  reference set into the system prompt"). Scales to hundreds of rows instantly.
- **Important (quality):** the tool only retrieves — accuracy is the content's responsibility.
  Verify the reference set against an authoritative source. Require in the prompt that the
  agent use ONLY what the tool returned, verbatim, and not make things up when the search is
  empty.
- **Evolution:** once pgvector/Qdrant becomes available, switch to semantics without changing
  the tool's interface.

Status: ✅ Confirmed in production.

### AI-R5. Agent tool storage with human-readable numbers (memory, reminders, lists)

- **Context:** an agent tool (`toolWorkflow`) stores the user's personal records (memory,
  reminders, todos) and shows their numbers in chat. Needed: gap-free numbers that reuse freed
  slots, in-place editing, minimal follow-up questions.
- **Pattern (4 parts):**
  1. **Number = `id`, gap-fill.** `save/set` inserts at the smallest free `id`
     (see postgres.md PG-R3). The displayed number is the real PK, so `delete`/`update`
     address it directly.
  2. **`update` operation (in-place edit).** Separate Route output: input `id` + new full
     `content`, recompute derived fields (for semantic memory — a fresh embedding),
     `UPDATE … WHERE id=… RETURNING id`. NOT delete+insert (otherwise the number moves to the
     end).
  3. **Display — a plain number.** In the Code formatter: `'  '+r.id+'. '+text`, not `[id N]`.
     In the prompt — call it a "number", not an "id".
  4. **Prompt autonomy.** Perform delete/edit immediately if the record is unambiguous. Ask a
     follow-up ONLY when it's ambiguous. `id` unknown → `search`/`list` first.
- **Deployment:** update the production `toolWorkflow` via PUT — keep node ids, add
  credentials to every node, after the PUT verify it's active with a retry (see
  rest-api-deploy.md DEP-E6). Test SQL against a local Postgres before production.

Status: ✅ Confirmed in production.

### AI-R6. Long-term memory of the interlocutor "like a real person" (recall + capture)

- **Context:** the bot must remember facts about the interlocutor across conversations and
  recall them itself without an explicit "remember" request (example: "going to the park with
  my dog Rex" → later the bot already knows who Rex is). Separate from conversation memory.
- **Pattern (2 independent mechanisms):**
  1. **Recall = "everything into context," no embeddings, no search.** Before the agent, a
     `Load Facts` node (Postgres) pulls ALL of the user's facts and concatenates them into one
     block:
     `SELECT COALESCE(string_agg('- ['||category||'] '||content, E'\n' ORDER BY category,id),'') AS facts_block ... WHERE telegram_id=...`.
     The block is inserted into the System Message via the expression
     `{{ $('Load Facts').first().json.facts_block }}`. Only this way does a fact surface on its
     own, without a command to search: semantic search does NOT work here — the agent won't
     think to search for "Rex." No embeddings needed while facts per user number in the tens to
     hundreds.
  2. **Capture = the agent itself calls the `save/update/delete` tool** (based on AI-R5, but
     without embeddings). KEY POINT: one soft instruction in the middle of the prompt is NOT
     enough — the model focuses on answering in character and silently skips saving. A **hard
     directive at the very TOP** of the System Message is needed: "system action — memory,
     ALWAYS executed before the reply; if the message contains a new/changed fact about the
     interlocutor, you MUST call memory.save/update, or it's an error." After that, it calls it
     reliably.
- **An important novelty caveat:** the agent judges "is this fact new" by the **conversation
  window**, not by the DB. A fact already stated within the window will NOT be saved again.
  Anything discussed BEFORE saving actually started working won't make it into the DB on its
  own — it has to be entered manually. To prove that recall comes from memory and not from the
  window, clear the conversation history.
- **`telegram_id` — not from the model.** In a multi-user bot, `telegram_id` is passed into the
  tool via the expression `$('Normalize').first().json.telegram_id` (not `$fromAI`), otherwise
  the model can mix up users.
- **Not implemented (backlog):** a separate fact-extraction module on a mini-model
  (deterministic pass after the reply, writes bypassing the agent). A standalone technique for
  scenarios where missing a fact is costly: lead qualification, gathering profile data from
  conversation, CRM enrichment, support with detail capture. A deterministic pass (always fires,
  doesn't depend on the model's mood) is more reliable than "the agent decides for itself."

Status: ✅ Confirmed in production (the bot recalled a fact after a full conversation-history
wipe — recall only from fact memory).

### AI-R7. A consultant tool writes its own result to the DB (deterministically), and returns the original answer to the agent

- **Context:** the main agent needs to save the sub-agent's structured result (a program, a
  plan, numbers) into the profile. If the agent itself does this via an update tool — that step
  is probabilistic and **silently skipped** (example: a program was generated, but
  `program_json` stayed NULL).
- **Solution:** the write is performed by **the consultant's own tool workflow** as a side
  effect, independent of the LLM. After the Agent node: `Persist` (Code: parses the agent's
  `{output}`, builds `_sql`+`_params` based on `mode`) → `Save?` (IF `_save`) → `Save Field`
  (Postgres `executeQuery`, `query="={{ $json._sql }}"`,
  `queryReplacement="={{ $json._params }}"`) → `Return` (Code:
  `return [{ json: $('Agent').first().json }]` — returns the agent's exact answer). One
  Persist node covers several modes (PLAN→`nutrition_json`, CAFFEINE→`caffeine_mg/norm`).
  `telegram_id` comes from the input `profile`.
- **Gotcha (Code node):** with `runOnceForEachItem`, returning `[{ json: { …array… } }]` failed
  with `A 'json' property isn't an object [item 0]`. Fix: **`runOnceForAllItems`** +
  `const out = $input.first().json;`.
- **Where this doesn't apply:** for large docs (program/nutrition_json) — the consultant
  writes them, and the "save it yourself" instruction is removed from the main agent's prompt
  (otherwise you get a double write/race). Small scalars (weight, age) — the agent still uses
  the update tool.

Status: ✅ Confirmed (isolated test of PLAN+CAFFEINE, production).

### AI-R8. A proactive reminder must repeat the user's request VERBATIM

- **Context:** on direct request, the agent sets a one-time reminder (the request text sits in
  `note`), and a separate cron workflow later generates the reminder text via a short
  mini-LLM prompt and sends it (`Build Prompt` node, `kind='reminder'` branch).
- **Gotcha:** a generic mini-LLM prompt ("write a warm message to the user") degenerates into a
  **routine well-being check** and does NOT mention the actual request. The user doesn't
  recognize the message as the reminder they ordered — it looks like a random nudge. (The tool
  was being called and fired as expected — the bug wasn't "the tool didn't fire" but the
  generation text plus up to an hour of lateness from the hourly cron.)
- **Solution:** hard-wire into the reminder-branch prompt: "This is a reminder at the user's
  EXPLICIT request. Start with the words 'You asked me to remind you about…' and state the
  gist VERBATIM (`note`). Do NOT turn it into a routine check-in question. If the gist is
  empty, give a short neutral reminder." In other words, `note` must appear in the text
  verbatim, not merely serve as "mood" for free generation.
- **General principle:** when a deferred message is produced by a separate LLM step, the
  user's original intent must be carried into the text verbatim — free generation around a
  "theme" loses recognizability for the recipient.

Status: ✅ Confirmed (short test).

### AI-R9. Deduplicating a bloated system prompt — a single canon instead of repeats (−25% tokens/turn)

- **Context:** a large agent's system prompt goes to the LLM on EVERY call — every extra line
  is paid for on every turn. A mature prompt accumulates duplicates over time: the same
  rule/prohibition rewritten in 3 sections "just to be safe." Example: the prompt had bloated
  to ~50,000 chars.
- **Solution:** collapse repeated blocks into **a single canon** + references/deltas:
  - a triple prohibition of tool/field names → one canon (in the "Hard Contract"), referenced
    by the other sections;
  - three copies of "when to call sub-agent X" → one routing section;
  - three "rewrite the reply in your own voice" → one shared block + short deltas by type;
  - three "degrade on tool error" → one;
  - ~60 listed names for gender detection → a rule + a short example list.
  Result: 49,967 → 37,613 chars (**−24.7%**) with no change in behavior or meaning.
- **Gotchas/limits:** (1) dedup is a prompt refactor — verify it behaviorally (run scenarios),
  not just by diffing; (2) do NOT leave a changelog/"before-after" in the prompt body — that's
  more tokens on every turn again; keep history in a version log. (3) the canon must sit in a
  section the agent will definitely read (top/contract), otherwise "see above" references are
  risky.

Status: ✅ Confirmed (short test).

### AI-R10. Hide a long structured bot output inside a Telegram expandable quote (`<blockquote expandable>`) via a sentinel marker + a post-processing node

- **Context:** the agent sometimes returns a large structured block (a workout program, a
  meal plan, a long list) — in chat this is a "wall of text" that's awkward to scroll through.
  Telegram can collapse such a block into an expandable quote
  (`<blockquote expandable>…</blockquote>`), but the tag only works **with `parse_mode` HTML or
  MarkdownV2** and the content inside must be properly escaped.
- **Solution (separating model responsibility from deterministic-node responsibility):**
  1. **The model wraps the block in a plain sentinel**, rather than writing HTML itself: in the
     System Message — a rule: "wrap the full program/plan in `⟪PROGRAM⟫ … ⟪/PROGRAM⟫`, and give
     1-2 lines of summary before the marker." The LLM isn't asked to generate valid HTML (it
     would mess up escaping) — only to place a cheap text marker.
  2. **A separate post-processing node before sending** (`Clean Output`, Code) finds the block
     by the sentinel and converts it into `<blockquote expandable>`, **escaping only the
     block's content** (`&`→`&amp;`, `<`→`&lt;`, `>`→`&gt;`) — leaving the rest of the text
     untouched.
  3. **Backward compatibility:** no sentinel → the text goes through unchanged. Existing
     replies don't break.
- **Important nuances:**
  - The pipeline's `parse_mode` must already be HTML (see telegram.md TG-R2) — the tag in
    plain-text mode would arrive as raw text.
  - **Don't cut the block with the 4096 splitter inside** — breaking `<blockquote>` in the
    middle corrupts the markup. If a block might exceed the limit, cut at block boundaries,
    not inside one.
  - Pick an "exotic" sentinel (`⟪ ⟫`) to avoid colliding with ordinary user/model text.
- **General technique:** build any "model generates → node formats" task with a fragile target
  syntax (HTML, MarkdownV2, JSON) this way: the model places a simple marker, and a
  deterministic node turns it into the target syntax with correct escaping. More reliable than
  trusting the LLM with escaping.

Status: ✅ Confirmed (renders correctly in Telegram).

### AI-R11. The agent must honestly state the coarseness of cron delivery timing, not promise an exact time

- **Context:** on request the agent sets a one-time reminder, but delivery is handled by a
  separate cron with coarse granularity (**once an hour** — a deliberate design, not a
  "reminder manager"). The user asks "remind me in 10 minutes."
- **Gotcha:** the agent friendly repeats after the user — "Sure, I'll remind you in 10
  minutes." But the cron fires on the nearest hourly tick → it actually arrives within the
  hour. Promising an exact time **is misleading** (especially for intervals shorter than the
  cron's step), even though the mechanism itself works fine.
- **Solution:** wire honesty about timing into the prompt: don't promise delivery to the minute
  and don't repeat "in N minutes" if N is smaller than the cron step; for a short interval —
  warn "I check in roughly once an hour, I won't hit this exact minute, I'll remind you within
  the next hour" and still set it (or offer to tie it to a specific time if precision is
  needed); for a specific time/hour — confirm without "down to the second." Don't change the
  cron frequency for this.
- **General principle:** when execution is asynchronous and coarse in timing (cron, queue,
  batch), the precision expectation is set by the agent's wording — the phrasing should match
  the real delivery granularity, not the user's wish. Fix it with expectation-setting, not
  necessarily frequency.

Status: ✅ Accepted for rollout (change live in production; final visual check by a tester).
Related to AI-R8 (same cron).

### AI-R12. A web chat over the SAME agent — cross-channel Telegram↔web history via shared sessionKey=telegram_id

- **Context:** a Telegram bot-agent gained a second channel (web/Mini App chat). It needs to be
  **one conversation**, continuing across both channels, not two isolated bots with separate
  memory.
- **Solution:** build the new channel as an agent with **three shared components**: (1) **the
  same system prompt** (read from the same `prompts.<name>`), (2) **the same Postgres Chat
  Memory with the same `sessionKey`** = the user's `telegram_id` (not the web session's
  chat_id!), same window, (3) **the same model**. Then both branches read/write the same
  `<bot>_chat_history` table under one key → history is unified: reply in Telegram — it shows
  in the web, and vice versa.
- **Isolation without sharing nodes:** the web workflow itself is **separate**
  (`<workflow_name>`), it doesn't touch the main bot (`<workflow_name>` stays byte-identical).
  What's shared is only data (the memory table, the prompt row, the model credential), not
  nodes. This protects the main bot from regressions when the web channel is edited.
- **`chat/history` for the UI — ordering.** To give the frontend the **last N** messages while
  keeping them **ascending** (oldest first): `SELECT … ORDER BY id DESC LIMIT N`, then **reverse
  the array in a Code node**. `ASC LIMIT N` would return the start of the history (not the
  tail); `DESC` without reversing gives a backwards feed. Filter out tool messages
  (`type='tool'` and technical assistant lines).
- **Boundary (deliberate):** the web agent can deliberately NOT get the main bot's tools
  (sub-agents/proactive/reminders) for the sake of isolation — then the web chat is a "pure
  conversation" on shared memory, while tool-driven scenarios stay on Telegram. Tool parity is
  a separate task.
- **Mirroring** the second channel into the Telegram conversation (so web replies also appear
  in the chat with the bot) — see telegram.md TG-R16; web-channel login via a Login Widget —
  telegram.md TG-R14.

Status: ✅ Confirmed in production (history is ascending; `chat/send` writes both messages into
shared memory, the agent's reply arrives).

### AI-R13. The deferred follow-up tool must be able to fire TODAY: remove the hard floor, past time → nearest slot in the window, else next day (mirror the reminder tool)

- **Context:** the agent has a "schedule follow-up" tool (`schedule_followup`): argument
  `in_days` + hour, writes to `proactive_queue`, delivery handled by a cron. Logically "let's
  talk tonight / later today" is a follow-up for **today**.
- **Gotcha:** the tool had a hard floor `in_days = Math.max(1, …)` — structurally it could NOT
  schedule for today. Any "tonight" request silently slid to tomorrow (`due_at` = tomorrow
  19:00). From the user's side: "promised to talk tonight — and didn't write today." The bug
  wasn't in the tool call (it fired fine), it was in the lower bound of the range and in the
  argument description, which didn't allow "today."
- **Solution (three layers, all required):**
  1. **Remove the floor:** `in_days = Math.max(0, Math.min(30, Number(t.in_days)||0))` (0 =
     today).
  2. **Mirror `due_at` calculation with the reminder tool, not a "naive +N days":** if the
     computed `due0 <= now()` (in the user's timezone) — shift to the **nearest whole hour
     within the cron's delivery window** (≤ the upper bound, e.g. 20:00); if already past the
     window — to the **next day** at the requested hour. `in_days≥1` (future) — unchanged. Done
     as a CTE `adj` in the Insert node's SQL, following the reminder tool's pattern.
  3. **The argument description and the prompt must explicitly allow "today":** the tool
     description/`$fromAI` hint text — "0 = today (this evening/later today), 1 = tomorrow, 2-3
     = a couple of days from now"; in the agent's prompt — route "tonight/later today" →
     `in_days:0` (or `set_reminder` for a specific time), "tomorrow/in a couple of days" →
     `in_days≥1`. Without this, the model still schedules for tomorrow even if the backend
     already supports today.
- **General principle:** a scheduling tool's allowed range must include "now/today," and the
  delivery-time calculation must account for the requested moment possibly already having
  passed (shift to the nearest slot, not blindly to tomorrow). Both the range's bounds AND the
  argument description AND the prompt must allow the near boundary — otherwise a "tonight"
  promise is structurally impossible. Reuse the "past time → nearest slot / next day" logic
  from the reminder tool, don't reinvent it.
- **Cap nuance:** if proactive messages have a daily cap (e.g. soft proactive 1/calendar day),
  a same-day follow-up will only fire if there hasn't already been a proactive message today.
  For guaranteed same-day delivery, pair with `set_reminder` (bypasses the window/cap).

Status: ✅ Confirmed (tests: in_days=0 in the morning → today 19:00; in_days=0 late/at night →
tomorrow; in_days≥1 as before). Mirrors AI-R8/AI-R11 (same cron) and the reminder tool's logic.

### AI-R14. "Proactive" tool calling: the phrasing "answer first in words, then call" suppresses the call — needs "call in the same turn, it's a background tool" + an example + "when in doubt, call it"

- **Context:** a tool-using agent (a product coach) needs to, in the right situations, call a
  background tool ON ITS OWN, without the user asking (`schedule_followup` — schedule a
  "how did it go" check), while returning ordinary text to the user.
- **Gotcha 1 — ordering:** the instruction "Answer in words first, THEN quietly call" doesn't
  work. For a tools-using agent the final text always comes LAST (ReAct loop:
  think→tool→…→final answer); "answer before the tool" is physically impossible, and such
  phrasing makes the call **optional** — the model gives its answer and drops the tool.
  Symptom: on an explicit trigger ("let's see how the day goes") the tool wasn't called, even
  though the prompt described it as mandatory.
- **Gotcha 2 — softness:** "call it when appropriate… don't overdo it" is read by the model as
  "by default, don't call it." For proactive (not directly requested) behavior, that's not
  enough.
- **Solution (prompt wording):**
  1. **Remove the false ordering:** not "answer first, then call," but "**call it in the SAME
     turn; it's a BACKGROUND tool — the user doesn't see the call; the order of the call and
     the text doesn't matter, what matters is not forgetting to call it**."
  2. **The tool is a duty, not an option** + a list of explicit trigger phrases verbatim ("let's
     see how the day goes," "we'll see," "we'll figure it out later") + a rule: "**when in
     doubt — set it**" (better to over-set than to stay silent).
  3. **A concrete example** of input→call:
     `"let's see how the day goes" → schedule_followup(in_days:0, when:evening, note:"…")`.
  4. For **anti-duplication and cancellation**, give the agent visibility into what's already
     scheduled: load open `proactive_queue` records into context on every turn (a `Load
     Pending` node → into `packet`) + a separate cancel tool (`cancel_followup`, UPDATE
     done=true by id). This way the model doesn't spawn duplicates and cancels the touch itself
     when the user reports back on the topic.
- **Verification:** before the fix the tool wasn't called on the explicit trigger; after — it
  reliably fires both for setting (`schedule_followup`) and for cancelling on the user's report
  (`cancel_followup`), and no repeated questions come from the evening cron tick.

Status: ✅ Confirmed (setting and cancelling both verified in production).

### AI-R15. Weekly deep review of user data — a dedicated prompt + volume-based calibration + strict JSON for a collapsible UI

- **Task:** not "a recap of period metrics," but an analytical review — correlations between
  metrics, causes, dynamics, personal recommendations. The source of the product coach's
  "signature" feature: a "Weekly Summary."
- **Solution (pattern):**
  1. **A dedicated prompt** (a row in `prompts`, read by the workflow), not the main agent
     prompt — it inherits persona/voice/philosophy from the main prompt, but adds an analysis
     METHODOLOGY: explicit requirements to look for correlations (X→next-day Y), the main
     lever, dynamics + period-over-period comparison, anomalies, a hidden risk, a "quiet win,"
     and ONE key non-obvious finding with numbers.
  2. **Rich input:** not just diary metrics, but also the user's **conversation** (history,
     filtered by `type<>'tool'`, the last N messages, stripped of technical headers) — this
     supplies the "why" behind the numbers; plus profile/goal/facts and the previous period for
     comparison.
  3. **Volume-based calibration:** pass into the prompt the number of days with data
     (`days_with_data`) and a tiering rule: little (≤1) — an honest snapshot without analysis
     plus a nudge to log more; 2-3 — a light review, do NOT invent correlations/risks; ≥4 —
     full depth. Enforce hard: "honesty matters more than volume, omit empty sections."
     Otherwise the model fabricates connections out of thin air.
  4. **Strict JSON by schema** (headline / key_insight{title,text} / sections{...} /
     correlations / hidden_risk / quiet_win / overall / recommendations[{what,why,how}]) —
     parsed into the DB and rendered as collapsible cards. `key_insight.title` — a short LABEL
     (3-6 words), otherwise the model writes a whole sentence there and the "title" balloons.
  5. **Localization in the text:** explicitly require Russian names for metrics (not the Latin
     keys mood/energy) — otherwise the review is peppered with English keys from the input
     data.
- **Generation gate:** only with ≥2 non-empty entries (otherwise there's nothing to review),
  an opt-in toggle, no report yet for this period (UNIQUE + NOT EXISTS). Once a week — an
  expensive model isn't a concern (big model, big prompt).

Status: ✅ Confirmed (review quality on real data, graceful degradation on sparse data).
Related: AI-R6 (memory about the interlocutor).

### AI-R16. Hard length limit: an LLM can't count characters → build in a margin plus deterministic trimming

- **Symptom:** you ask the model for a "summary ≤950 characters," and it regularly overshoots
  (1000-1100). Telegram's caption limit is ≤1024, our working ceiling is 950 → the post
  doesn't go through / the button gets blocked.
- **Cause:** an LLM **can't count characters precisely** — it treats "≤950" as "roughly that
  much," aiming near the limit and often overshooting.
- **Solution (two measures):**
  1. **Build a margin into the prompt's target:** ask for less than the limit — "aim for
     820-880 characters, STRICTLY ≤950, better shorter than right at the edge." The model
     lands under the threshold.
  2. **A deterministic guarantee after the model** (Code node), not trusting the LLM's count:
     if the result exceeds the limit — cut at a **sentence boundary**:

```js
let text=(p.post||'').trim();
if(text.length>950){const cut=text.slice(0,950);const m=cut.match(/[\s\S]*[.!?…](?=\s|$)/);text=(m?m[0]:cut).trim();}
```

     Guarantees ≤950 without breaking mid-word (only the trailing sentence gets cut, which,
     with the margin in the target, happens rarely).
- **General rule:** for ANY hard numeric constraint that an LLM produces (length, item count),
  don't rely on the prompt alone — add a deterministic check/trim in code. The prompt lowers
  the frequency, the code guarantees it.

Status: ✅ Confirmed.

### AI-R17. A cap of "N proactive touches per day" — a counter+date in the profile, not a single last_at; plus time spacing

- **Context:** proactive messages (product coach) need to be capped per day. Originally it was
  1/day via `last_proactive_at::date < today` — but that's hard-coded to 1 and doesn't scale
  to "up to 2."
- **Gotcha:** a single `last_proactive_at` timestamp doesn't hold a counter — "2 per day" can't
  be built on it.
- **Solution:** add `proactive_day date` + `proactive_count int` to the profile. Selection
  gate: `(proactive_day IS DISTINCT FROM today OR COALESCE(proactive_count,0) < N)`. On send
  (non-reminder), update: `proactive_count = CASE WHEN proactive_day=today THEN count+1 ELSE 1
  END, proactive_day=today`. Compute the date in the user's timezone
  (`(now() AT TIME ZONE timezone)::date`).
- **Time spacing (important):** with fan-out (processing all candidates on a tick) and no
  interval, two touches could land in consecutive hourly ticks back to back. Add
  `AND last_proactive_at < now() - interval '4 hours'` — then N touches are spread out instead
  of arriving in a batch. Dedup by telegram_id in Build Prompt guarantees ≤1 message per user
  per tick.
- **Explicit reminders (reminder) are not capped** — a separate branch, doesn't touch the
  counter.

Status: ✅ Confirmed (migration done, gate enforced).

### AI-R18. A monetization gate before the agent: fail-open, after dedup, absolute counters from the gate

- **Task:** trial/plan limits for an LLM bot (trial N messages/M days, monthly limits, channel
  gates for voice/photo by plan, an 80% warning).
- **Pattern:** the chain `Plan Load (SELECT counters, alwaysOutputData) → Plan Gate (Code:
  decides block/warn + ABSOLUTE new counter values) → IF → Plan Count (UPSERT) → IF Warn →
  Send Warn` is inserted **after dedup** of incoming messages (Telegram retries don't double
  the count) and **before** the buffer/agent.
- **Key decisions:** (1) **fail-open** — the whole Code gate wrapped in try/catch, on an
  internal error the message passes through: the gate must not crash the bot; (2) the counters
  are computed by the GATE and passed as absolute values — UPSERT simply writes them (no
  increment race conditions in SQL, monthly reset is a `plan_month` comparison); (3) UPSERT
  creates a profile row for a newcomer with plan='trial' and `ON CONFLICT` does NOT touch
  `plan` and `trial_started_at` (COALESCE); (4) the `tester` plan (unlimited) for internal use
  — counters bypassed; (5) block-messages — warm, mention the plans, kept in one place (Code).

Status: ✅ Confirmed (both environments; logic covered by a unit test — 16 cases).

### AI-R19. Static user data goes into the system context, not into a mandatory tool call

- **Symptom that prompted this:** the prompt required "mandatory first step —
  read_profile_tool" → almost every agent turn started with an extra full context pass
  (~26k tokens) for data that's already known BEFORE calling the model.
- **Solution:** the context-assembly node (in both the main bot and the web chat, nodes `Merge
  for Agent`/`Chat Prep`) appends a `[USER PROFILE]: {json}` block to the systemMessage; heavy
  fields (full programs) and internal ones (plan counters) are excluded from the block — the
  tool remains for those. Prompt: the mandatory step is removed, the tool is "after your own
  edits in the same turn, or for heavy fields." Put the block at the TAIL of the system
  message (the static prefix cache stays intact; the profile changes rarely).
- **Effect:** on ordinary turns the tool isn't called (verified via runData); bonus — the
  tool-less agent (web chat) sees the profile for the first time.
- **Observation:** on a DIRECT question "tell me my data," the model still plays it safe with a
  tool call (cost = same as before, not worse). Tightening this via the prompt is up to the
  owner.
- **Pattern verification:** execution runData — the block is visible in inputOverride, and the
  nodes show no tool call.

Status: ✅ Confirmed.

### AI-R20. Consent under 152-FZ in a Telegram bot: a deterministic gate + an inline button, not via the LLM

- **Task:** explicit consent to process personal data (including special categories) BEFORE
  any data collection, with the date/version recorded.
- **Pattern:** (1) Trigger updates += `callback_query`; the `Is Callback?` branch FIRST (before
  access gates) → UPSERT `consent_at/consent_version` (COALESCE — a repeat click doesn't
  overwrite the date) → answerCallbackQuery → welcome message; (2) the `Consent Load → IF`
  gate sits AFTER dedup and BEFORE the limit counter (consent doesn't burn trial usage) —
  without consent, a fixed text with an "I agree" button and a link to the policy is sent, the
  message content is not processed; (3) the consent text and its recording are handled only by
  deterministic nodes (legal reproducibility), the LLM is not involved; (4) the welcome message
  after the button is conditional (`RETURNING name`): for a new user — introduction, for an
  existing one — "let's continue, repeat your message"; (5) in web/miniapp — the same
  `consent_at` via a branch in the build-SQL, the app shows an overlay.

Status: ✅ Confirmed.

### AI-R21. A conditional prompt phase block (trial/segment) without breaking the prefix cache

- **Task:** guide a user segment (trial) along a "route" with extra instructions, without
  bloating the prompt for everyone else or killing OpenAI's cache.
- **Pattern:** the phase instruction block is stored as a separate row in `prompts` (editable
  without touching the workflow); the context-assembly node inserts it CONDITIONALLY
  (`plan==='trial'`) into the system message BETWEEN the static part and the profile — ordering
  "rare to frequent": [static prompt | phase block (changes once a day — only {DAY}) | profile
  (changes more often)]. For other segments the system message doesn't change by a single
  byte. Milestone days are "a soft guideline, don't break a live conversation"; hard phase
  events (a day-7 offer) are NOT done via the LLM, but by a deterministic INSERT into the
  proactive queue when the phase starts (idempotent by note).

Status: ✅ Confirmed.

### AI-R22. Paid content generation in a webhook: gate → LLM-JSON → gpt-image → INSERT (quota doesn't burn on failure)

- **Task:** a "generate for money" feature (LLM + image) with plan limits, synchronous in a
  webhook router, with no risk of charging quota for a failed attempt.
- **Pattern:** node order = order of the money: (1) context in one SELECT (profile minus heavy
  fields + weekly/daily counters + last N titles to avoid repeats); (2) a Code limit gate by
  plan (deny returns as `ok:true, data:{denied,text}`, so the client shows a soft toast, not an
  error); (3) HTTP → chat/completions (`response_format: json_object`, a schema with all
  fields in the system message, including `theme` from a fixed list); (4) Code schema
  validation — `throw` HERE is cheap: no INSERT has happened yet, quota is intact; (5) HTTP →
  images/generations with `output_format: "webp", output_compression: 80` — instead of a ~1.5MB
  PNG you get a ~110KB base64, storable directly in a PG column; (6) INSERT with a
  queryReplacement array → RETURNING for the response to the client (include the photo in the
  `create` response, but NOT in `list` — serve it via a separate `get`).
- **Sending a raster image via the bot:** the client sends a JPEG data URL → Code: strip the
  prefix + `prepareBinaryData` → Telegram `sendPhoto` multipart (`formBinaryData`).
- **Cost:** a mini-LLM ≈ pennies + the image generator at medium 1024² ≈ $0.04-0.05 per
  generation; limits (1-2/week by plan, unlimited with a daily ceiling for the top tier) are
  enforced by the SERVER, the client only displays them.

Status: ✅ Confirmed (live E2E, several generations).

### AI-R23. Vision for food: a mini-model is no worse than the flagship and cheaper; recognition via HTTP + max_completion_tokens

- **Symptom/task:** food-photo recognition (calories/macros from a plate photo). Model choice
  and how to call it.
- **Measured, not guessed:** on real food photos with known calories/macros (reference = our
  own generated recipes), a cheap mini-model was compared against a top-tier predecessor model.
  On calories, the mini-model was no worse (−2…−13% off the reference for both), but it
  **breaks the plate down into items with gram amounts in more detail**, where the older model
  lumps everything into one dish. Mini is far cheaper → mini was chosen. Recognition cost per
  active user dropped several-fold.
- **Call gotchas:**
  - the langchain `openAi` node (resource:image, analyze) sends `max_tokens` → on newer models
    this fails with `'max_tokens' is not supported, use max_completion_tokens`. Fix: **a direct
    HTTP call** `POST /v1/chat/completions` with `max_completion_tokens`, the image as an
    `image_url` data URL.
  - From a Telegram binary: a separate Code node with `getBinaryDataBuffer(0,'data')` →
    `'data:'+mime+';base64,'+buf.toString('base64')`.
- **The food prompt — 3 explicit modes (otherwise numbers from screenshots get lost):** (1) a
  photo of REAL food → `[FOOD]{items,total,note}`; (2) a SCREENSHOT with data (a food diary,
  an activity screen, a document) → a detailed transcription of ALL numbers on screen in
  Russian (otherwise the model would write "this is a screenshot" and the numbers never reached
  the agent — caught on real user screenshots); (3) anything else — a plain description.
  `[FOOD]` only in mode 1.
- **Recognition's plan gate matches the main bot** (some plans allow it, others don't); the
  client resizes photos to 1024/JPEG before sending.

Status: ✅ Confirmed.

### AI-R24. Long-term memory hygiene: TTL + a weekly automatic review with a code guard

- **Symptom:** a memory agent piles up "facts forever" without distinguishing
  permanent/temporary — stale promises and one-off events clutter the context and end up in the
  wrong category.
- **Solution (3 layers):**
  1. **The write prompt:** strict categories (family = permanent facts + pets only; promises
     with a date → the reminder tool, not memory; one-off incidents — don't write them; before
     saving, check for an existing record → update, not a duplicate).
  2. **TTL:** an `expires_at` column; the memory tool accepts `ttl_days` (only for temporary
     circumstances); a nightly cleanup deletes expired ones.
  3. **A weekly review** (cron on Sunday + a manual webhook): a mini model looks at each
     user's dated facts → JSON `{delete[],update[]}`. **CRITICAL — the code guard is stricter
     than the prompt:** the applying Code node only allows delete for the "events"/"context"
     categories, update only when the text actually changed (not a no-op); family/values/
     habits/work are untouchable at the SQL level. A dry run caught the mini model trying to
     delete a useful habit — without the code guard, the review itself would have destroyed
     memory.
- **Test before going live:** run the review in dry-run mode (return the plan only, don't
  apply it) against real data for several users — see what the LLM wants to delete.

Status: ✅ Confirmed (production run: a duplicate was cleaned up, useful data untouched).

### AI-R25. Guided agent sessions: launched by a trigger phrase from the UI + a protocol in the prompt + a proactive schedule

- **Task:** structured practices (a CBT thought-review, grounding, an evening wind-down) and
  periodic "sessions" with the agent — without a dedicated session backend.
- **Solution in three parts:**
  1. **Launched by a trigger phrase:** a button in the app sends a canonical phrase into the
     chat ("…let's work through an anxious thought (practice)"). The web client auto-sends it
     into its own chat; a Telegram Mini App can't send AS the user → the phrase goes to the
     clipboard + the mini app closes into the chat (fallback: show the phrase as text).
  2. **A protocol in the prompt, not in code:** for each practice — numbered steps and ground
     rules ("one step per message, wait for a reply, 5-10 minutes, no lectures/diagnoses, close
     with a warm wrap-up"). The LLM tracks session state itself via chat history — a separate
     state machine isn't needed.
  3. **A periodic session — via the existing proactive queue:** a weekly cron does an INSERT
     into the proactive queue (a kind marker, a deterministic time) with a plan/opt-in/consent
     gate and idempotency per week (NOT EXISTS … kind AND due_at >= start of week); the regular
     proactive workflow delivers it, the prompt describes the scenario by kind.

Status: ✅ Confirmed (based on reviewing a third-party wellness app, Wysa).

### AI-R30. An action in a mini app continued by the agent in chat (seeding chat memory)

- **Task:** a button in a Telegram Mini App should launch a conversational scenario (a
  practice) in the NATIVE chat with the bot. A mini app (launched from the menu) can't send a
  message as the user — only copy-paste (confusing) or closing.
- **Solution:** an action in the mini app's router (`practice/start`): (1) the bot sends the
  scenario opener into the user's chat via `sendMessage` (a token from config), (2) that same
  message is **inserted into the agent's chat memory** as an ai reply —
  `INSERT INTO <bot>_chat_history(session_id, message) VALUES (tid,
  '{"type":"ai","content":"…","tool_calls":[],"additional_kwargs":{},"response_metadata":{}}'::jsonb)`.
  The mini app closes; the user sees the started scenario in the chat, replies — the agent
  picks up the next turn WITH CONTEXT (the opener is already in its memory).
- **Key point:** `memoryPostgresChat` (LangChain) reads the last N by `session_id`; a manual
  INSERT into the same table in the same format = seamless continuity without spending tokens
  and without fake Telegram updates. Verify the message format against a live row (in our
  case, `content` sits directly in the message, not inside `data`).
- **Web version:** the scenario is driven through the built-in chat (a regular `chat/send`) —
  no action needed.

Status: ✅ Confirmed (verified on-device, Telegram-only).

### AI-R26. A crisis protocol in a wellness agent's prompt (safety)

- **Task:** an AI habit coach might encounter a user in acute distress (thoughts of self-harm,
  etc.). It must neither ignore it nor play doctor.
- **Solution:** a "CRISIS AND SAFETY" block that takes priority over everything: (1) recognize
  direct/indirect red flags; (2) a warm, serious response without panic or moralizing; (3)
  directly name free government hotlines (in Russia: EMERCOM 8-495-989-50-50, national
  8-800-2000-122, emergency 112); (4) check for immediate danger and suggest calling/reaching
  out to someone close; (5) do NOT diagnose, dismiss, or steer the conversation back to
  workouts/nutrition. A caveat: "this is a rare scenario — don't see a crisis in ordinary
  fatigue," to guard against false positives. Duplicate the contacts in the UI too (an in-app
  card with tel: links).
- **Testing a safety feature — via an isolated LLM call** with a synthetic (fictional) signal,
  not on a real user. Result: all numbers plus the safety check were delivered, without any
  diagnoses.

Status: ✅ Confirmed (tested in isolation).

### AI-R27. Verifying a prompt change on a SYNTHETIC profile + safe prompt deduplication

- **Task:** change a large agent system prompt without risk (behavior regression) and without
  writing to the DB or touching a real user.
- **Isolated behavior test (0 records, 0 real users):**
  - A temporary workflow: Webhook → HTTP to the model → Code(extract). `system` = the NEW
    prompt + a synthetic profile block (`{"name":"Testina","gender":"F",...}` — clearly fake,
    not a real person). `user` = a provocative message targeting the behavior under test. Read
    the response, delete the workflow.
  - **You must NOT** mint a real tester's session and send it into their live conversation —
    that's impersonation in an external system (the classifier rightfully blocks it).
    Synthetic data is the safe substitute.
  - Example checks: "send it to my email" → an honest refusal + an alternative (the honesty
    block is intact); logging food for a woman → refers to herself in the feminine (the gender
    rule is intact).
- **Prompt deduplication without losing quality:** a large prompt often accumulates DUPLICATE
  semantic blocks (e.g., "what I can/can't do" twice). Merging = token savings on EVERY message
  without changing behavior — but only as a **union**: keep one block, fold the unique points
  of the second into it, dropping nothing. Check the length delta and run the isolated test.

Status: ✅ Confirmed (synthetic run passed).

### AI-R28. Robust LLM-JSON parsing in a write tool node: bracket repair + an honest `throw` (not a silent `{}`)

- **Symptom:** the user dictated/photographed a meal, the bot replied "Logged your food for
  today…," but in the Mini App "Nutrition by plan" showed empty buttons 1-5 (instead of the
  "% of norm" pill), the Nutrition tab was empty. The bot's execution — `success`.
- **Cause:** the write node parsed the `metrics` argument from the agent as
  `try{JSON.parse(s)}catch{metrics={}}`. The model returned `food_log` with broken JSON — an
  array opened with `[`, closed with `}` instead of `]` (a missing `]`). `JSON.parse` threw →
  the catch block **silently zeroed out ALL metrics for that call** → nothing was recorded,
  only the SELECT recent ran → the tool returned success → the agent concluded it had logged
  the meal, and **lied to the user**. Silent data loss under a "success" status.
- **Solution (two layers):**
  1. **Repair the JSON before handing it off** — `repairJson()`: a character-by-character pass
     with a bracket stack (accounting for strings/escapes), fixes mismatched/missing `]`/`}`
     (a closer for the wrong bracket type gets replaced with the correct one for the top of the
     stack; unclosed ones get closed at the end), strips a ``` code-fence wrapper. The missing
     `]` in `food_log` gets fixed.
  2. **Don't lie on failure** — if it still doesn't parse after repair: `console.log(raw)` +
     `throw` (the agent gets an error → honestly says "couldn't log it, try again"), NOT a
     silent `metrics={}`. The valid path is byte-for-byte unchanged; an empty `metrics`
     (read-only) doesn't throw.
- **Verification:** tested against a real broken JSON from a user (several dishes get fixed and
  logged), against valid input (unaffected), against garbage (throws), and against empty
  (SELECT only).
- **General principle:** parse any LLM-produced JSON argument going into a write tool node with
  repair + an honest error on failure; NEVER substitute `{}`/`[]` disguised as success —
  otherwise the agent reports success while there's no data. Related: AI-E4 (parser robustness
  against garbage), AI-R7 (deterministic write bypassing the LLM), AI-R22 (throw before the
  write — quota stays intact).

Status: ✅ Confirmed.

### AI-R29. A day-by-day proactive funnel: deterministic selection + free-form text from the LLM

- **Task:** guide a user through a paid trial (or onboarding) with touches, rather than
  hoping the model schedules follow-ups on its own. Incident: trial guidance "stopped" — the
  agent wrote once, the user replied, no new follow-up got scheduled by the agent, and the
  "silent for ≥3 days" threshold hadn't kicked in yet → silence during the most conversion-
  critical week. There was no systemic funnel at all — guidance rested entirely on the model's
  judgment.
- **Pattern (who decides what):**
  - **SQL, not the LLM, picks the touch day.** A branch in candidate selection: `plan='trial'`,
    day = `extract(day from now() - trial_started_at)`, `dn IN (1,3,5,6)`. Deterministic,
    reproducible, visible in the DB.
  - **The day's goal is in the prompt, the text comes from the model.** Each day has its own
    goal (first step → get them into an untouched section → show progress → gently mention the
    subscription). Templated blasts read as spam; a goal plus profile/diary context produces
    living text.
  - **Repeat protection — a log in the same queue:** after sending,
    `INSERT INTO proactive_queue (kind='trial', note='day:N', done=true)`, and in selection
    `NOT EXISTS(... kind='trial' AND note='day:N')`. No separate table needed, idempotent no
    matter how many scheduler ticks run.
  - **Branch priority via `prio` + `UNION ALL`:** an explicit user reminder (0) → the funnel
    (1) → a follow-up (2) → silence (3), plus a "one message per tick per user" dedup.
  - **The silence threshold depends on stage:** 1 day on trial, 3 days once paid —
    `CASE WHEN p.plan='trial' THEN interval '1 day' ELSE interval '3 days' END`. A uniform
    threshold either loses trial users or annoys subscribers.
- **A mandatory window:** send only during daytime hours in the user's `timezone`, and no more
  often than once every N hours, otherwise the funnel overlaps with regular touches.

Status: 🟡 Draft (deployed, selection verified dry; confirmation pending the first live
touches).

---

## ⚠️ Errors

### AI-E1. `pairedItem` breaks on the `@n8n/n8n-nodes-langchain.openAi` v2.1 node

- **Symptom:** in an expression after openAi v2.1 you write `$('Node_Name').item.json.field` —
  it returns `undefined`. The downstream node receives empty parameters (in Telegram sendPhoto
  this produces `Bad Request: there is no photo in the request`), even though the predecessor
  actually returned a value.
- **Cause:** the openAi v2.1 node doesn't propagate pairedItem metadata. `.item` relies on
  pairing to determine "which parent item produced the current one," and without it, it falls
  through to undefined. Especially painful in chains like
  `imgBB → openAi → sendPhoto`, where the URL is fetched via a cross-node reference.
- **Fix:** in ALL cross-node expressions where at least one openAi v2.1 node sits between the
  source and the consumer — use `.first()` instead of `.item`:
  - ❌ `$('Upload to imgBB').item.json.data.url`
  - ✅ `$('Upload to imgBB').first().json.data.url`
- **Prevention:** default to writing `.first()` always in cross-node references — it's safe in
  all cases, and use `.item` only when pairing is explicitly needed.

### AI-E2. An agent tool (`toolWorkflow`) silently reports "unavailable" when the sub-workflow is deactivated

- **Symptom:** the agent replies "the tool is currently unavailable" in chat. Meanwhile the
  main bot's execution status is **success** (not error), so it doesn't show up in the logs
  right away. In the runData of the problem tool node you find:
  `{"response": "There was an error: \"Workflow is not active and cannot be executed.\""}`.
  The tool sub-workflows themselves have **no separate executions** (the call happens inline
  in the parent).
- **Cause:** the sub-workflow tool (triggered by `executeWorkflowTrigger`) got switched to
  `active=false`. When called via AI Agent, such a workflow returns an error as a string in
  `response`, and the agent interprets it as "tool unavailable." The parent execution stays
  success — the error never surfaces to the Error Workflow.
- **Fix:** reactivate the tool workflow: `POST /api/v1/workflows/{id}/activate`. For
  `executeWorkflowTrigger`, activation is safe — it doesn't run on its own.
- **Diagnosis:** don't trust the parent's `success` status. Open the main bot's execution with
  `includeData=true`, check each tool node's `response` in `runData` for the substring `There
  was an error`. In parallel, `GET /api/v1/workflows/{id}` for each tool → check `active`.
- **Prevention:** keep tool workflows always active. **Important:** after
  `POST /api/v1/workflows` a new tool workflow is created inactive — activate it right away.

### AI-E3. Postgres Chat Memory throws `Got unexpected type: constructor` on a manual INSERT into history

- **Symptom:** after a third-party workflow (a proactive sender) wrote a message into the chat
  history table, **EVERY subsequent user message crashes the main bot** — the Memory node
  throws `Got unexpected type: constructor` (visible in the memory node's runData and in the
  Error Workflow). Before that first manual INSERT, the bot worked fine.
- **Cause:** the write was made in the langchain **Serializable `toJSON()`** format —
  `{"lc":1,"type":"constructor","id":["langchain_core","messages","AIMessage"],"kwargs":{"content":…}}`.
  But n8n's Postgres Chat Memory loader (`mapStoredMessageToChatMessage`) only understands
  **StoredMessage**: `{"type":"ai"|"human"|"tool","content":"…",…}` (content at the TOP level,
  no `kwargs`). On `type:"constructor"` it throws, and the whole agent turn crashes.
- **Fix:** a manual INSERT into the history table — StoredMessage format only:
  `{"type":"ai","content":<text>,"tool_calls":[],"additional_kwargs":{},"response_metadata":{},"invalid_tool_calls":[]}`.
  Fix already-broken rows with an UPDATE:
  `SET message=jsonb_build_object('type','ai','content',message->'kwargs'->>'content','tool_calls','[]'::jsonb,'additional_kwargs',COALESCE(message->'kwargs'->'additional_kwargs','{}'::jsonb),'response_metadata',COALESCE(message->'kwargs'->'response_metadata','{}'::jsonb),'invalid_tool_calls','[]'::jsonb) WHERE message->>'type'='constructor'`
  (content is preserved).
- **Prevention:** (1) when manually writing a row that ANOTHER node reads (chat memory, a
  queue) — first read the actual row written by the consumer itself, and replicate its schema
  1:1, don't guess. (2) an E2E test of a proactive send must cover the FULL cycle including the
  consumer: send a message AND then reply as the user, confirming memory loads; "the message
  went out" isn't enough. (3) don't copy inherited code (a format from a previous version)
  without checking — inherited ≠ correct.

Status: ⚠️ Confirmed in production, fixed (write format + repairing broken rows via UPDATE).

### AI-E4. A leaked-tool-call recovery parser duplicates a write on a repeated marker

- **Context:** gpt-5.4 sometimes "leaks" a tool call as text — printing
  `to=functions.<tool>` + JSON arguments directly into the reply instead of a proper tool
  call (often with garbage tokens mixed in). Protection: a recovery node (Code) scans the
  agent's `output` for `to=functions.X`, parses the JSON and applies the write itself
  (INSERT/UPDATE) — the tool itself never actually ran.
- **Symptom:** one action got written **twice** from a single turn (two identical reminders in
  `proactive_queue`).
- **Cause:** the leaked text contained the `to=functions.X` marker SEVERAL times, but there was
  only one JSON argument block. The parser takes the next `{…}` after each marker → both
  markers grabbed the SAME JSON → two identical INSERTs.
- **Fix:** dedup identical statements within one recovery pass:
  `const uniq=[...new Set(stmts)]; if(!uniq.length) return []; return [{json:{sql:uniq.join('\n')}}]`.
  Safe: distinct legitimate writes differ in text and are kept, idempotent repeats collapse.
- **Prevention:** design any parser of non-deterministic LLM output (a leaked tool call, free
  text) to be robust against repeats/garbage — dedup the results or bind the JSON to a specific
  marker by position.

Status: ⚠️ Confirmed in production, fixed (dedup in Parse Writes).

### AI-E5. A follow-up slid to tomorrow because of a hard `in_days≥1` floor — the "tonight" promise wasn't kept

- **Context:** the agent agreed with the user "let's talk tonight," and called the deferred
  follow-up tool (`schedule_followup`).
- **Symptom:** the message arrived not tonight but the next day at 19:00. From the user's
  side — "promised and didn't write today."
- **Cause:** the tool's Build node had `in_days = Math.max(1, …)` — the range's lower bound was
  1 (tomorrow). The tool structurally couldn't schedule for today; any "tonight" request got
  rounded up to tomorrow. The tool was called correctly and wrote to `proactive_queue` fine —
  the bug was in the argument's range, not the mechanics.
- **Fix:** see AI-R13 — remove the floor down to 0, shift a past time to the nearest slot in
  the window (otherwise the next day), and make the argument description and the prompt
  explicitly allow "today."
- **Prevention:** check the lower bound of any scheduling tool — does it allow "now/today"; a
  hard `Math.max(1, …)` on a time unit is a red flag if the scenario is meant to fire within
  the same period.

Status: ⚠️ Confirmed, fixed (see AI-R13).

### AI-E6. Switching the OpenAI model generation breaks nodes with obsolete parameters — a bare-request smoke test does NOT catch this

- **Symptom:** after a mass model swap to a new generation, the bot crashes on the first live
  message: `Unsupported parameter: 'temperature' is not supported with this model`; next in
  line was `max_tokens` ("Use 'max_completion_tokens'").
- **Cause:** newer model generations don't accept sampling parameters (`temperature`, `top_p`,
  penalties) or the old `max_tokens`. A "bare" smoke ping (model+messages only) passes fine —
  but production nodes carry options parameters left over from the old model. The parameters
  live on DIFFERENT node types: `lmChatOpenAi` (options.temperature) and `openAi`
  (options.temperature + options.maxTokens) — you must scan all node types for the model name
  appearing in parameters, not just lmChat.
- **Fix:** when switching models — (1) go through ALL nodes in ALL workflows where the model
  name appears and strip temperature/topP/penalties/maxTokens; (2) run the smoke test WITH the
  production nodes' parameters, not a bare request; (3) check executions of all affected bots
  for a period after the switch.

Status: ⚠️ Gotcha (fixed: nodes cleaned up, all PUTs ok).

### AI-E7. Reasoning models break the response parser: the text isn't in `output[0]`, it's in the item with `type='message'`

- **Symptom:** the app "hangs" during generation. The n8n execution is **success in 17-21s**,
  but the parsing node returned `{header:"",post:""}` → an **empty post** got saved to the DB →
  the frontend polls for the result until timeout. No error anywhere: both n8n and the model
  reported "success."
- **Cause (the TRIGGER was a switch to a reasoning model, it didn't "just break" on its own):**
  the openAi v2.1 node (Responses API) on reasoning models returns `output` as a **list of 2+
  elements**: `output[0].type='reasoning'`, the text is in `output[1].type='message'` →
  `content[0].type='output_text'`. The parser hard-coded `resp.output?.[0]?.content?.[0]?.text`
  → `undefined` → an empty object → empty fields.
  **How it was found:** a neighboring node on a mini-model (`reasoning_tokens: 0`) worked fine,
  while the node on a reasoning model (`reasoning_tokens: 355`) broke. In other words, exactly
  the nodes that had been switched to the reasoning model broke (the change was made **manually
  in the UI**, bypassing the config generator → it still showed the old model = drift).
  **Marker:** `usage.output_tokens_details.reasoning_tokens > 0` → the model is in reasoning
  mode → a different response envelope.
- **Solution — find the message item, don't index into it:**

```js
var _o = resp.output || [];
var _m = _o.find(x => x && x.type === 'message') || _o[_o.length-1] || {};
var _c = _m.content || [];
var _i = _c.find(x => x && (x.type === 'output_text' || x.text !== undefined)) || _c[0] || {};
var _t = _i.text;   // a string, or already an object (json_mode)
```

  Works for non-reasoning models too (there, message is the only element). Apply this in both
  Code nodes and Set expressions.
- **Scale (important):** this pattern is copy-pasted across all AI branches — it hit more than
  a dozen nodes in several production workflows (preprocessor, editor, co-writer, text edits,
  art-director/image_prompt, caption generation, time parser). **When this kind of breakage
  happens, check ALL parsers at once** (`grep 'output?.[0]'` across the export), not just the
  one you noticed.
- **Prevention (root cause of "silent hanging"):** an empty result from an AI call IS a
  **failure**, not "still processing." The frontend/consumer must be able to tell the
  difference: if the backend responded but the text is empty, surface an error to the user
  immediately, don't poll until timeout.
- **Diagnosis:** "hanging" + successful ~20s executions → don't look at timeouts, look at
  **what the parser actually saved**; then inspect the live response's `output` structure
  (`executions/{id}?includeData=true`).
- **Rule going forward:** switching a model's generation/type = **a change in response shape**,
  not just quality. After moving to a reasoning model, re-check ALL parsers on that branch
  (`grep 'output?.[0]'` across the export) and run one live call per AI branch. Editing the
  model in the UI, bypassing the config generator, creates drift. Related entry: AI-E6
  (switching generations breaks nodes with obsolete parameters).

Status: ✅ Fixed and verified live.

### AI-E8. The agent talks about ITSELF using the user's gender

- **Symptom:** a male bot-persona writes to a woman about its own actions in the feminine
  grammatical gender: "Name, **[logged, feminine form]** your food for today" (should be the
  masculine form). Caught by a female tester.
- **Cause:** the persona's gender is nowhere stated explicitly in the prompt, while the profile
  field is described as "`gender` — M/F. For calculations and **addressing in the correct
  gender**." The LLM interprets this broadly: it adjusts ALL of its speech to match the
  interlocutor's gender, including verbs about itself. Russian first-person grammatical gender
  isn't "style" — it's grammar the model borrows from the interlocutor's context.
- **Solution:** an explicit hard rule in the persona block, separated from the profile field:
  - "You are male. You speak about YOURSELF ONLY in the masculine, always, regardless of the
    interlocutor's gender: [logged, watched, glad — masculine forms]. Never carry the user's
    gender over onto yourself."
  - "`gender` affects ONLY words about the USER: you [logged, passed — feminine forms if
    female]."
  - Add a concrete anti-example from the bug ("Name, [logged-feminine]…" → "Name, [logged-
    masculine] your food…") — the model responds better to these than to an abstraction.
- **Testing without intruding into someone else's conversation:** a temporary workflow with the
  same system prompt + a SYNTHETIC female profile block + a single LLM call on the production
  model. Don't mint a real user's token and don't write into their live conversation — that's
  impersonation.
- **Lesson:** any role/grammatical persona traits (gender, age, "I") must be stated explicitly
  — what's "obvious from the character's name" isn't a given for the model.

Status: ✅ Fixed and verified.

### AI-E19. Decoupling a subsystem — an access gate keeps living in NEIGHBORING workflows (silently disables features)

- **Symptom:** after decoupling the product from the old access system, everything works —
  login, payment, chat. But a separate feature (proactive messages) silently stops firing for
  some users, with no errors anywhere.
- **Cause:** only the obvious `Check Access` nodes in three workflows were edited, but the
  condition `EXISTS (SELECT 1 FROM bot_access WHERE ... AND active)` remained in the proactive
  workflow — hidden inside a large candidate-selection SQL query rather than in a node with a
  descriptive name. As long as rows kept being created in `bot_access` by inertia, nothing
  broke; once they stopped, the feature died without a single error.
- **Fix:** after decoupling/replacing a gate, grep **across all workflows** for the old table's
  name, not by node names: `grep -l 'bot_access'` on the export, or a SQL query against
  `workflow_entity.nodes::text LIKE '%bot_access%'`. Bring everything found in line with the
  single access rule.
- **Rule:** the access gate must live in ONE place (a DB view/function or a single
  sub-workflow). Duplicating the condition across workflows is a guaranteed desync the next
  time the access model changes.

Status: ⚠️ Gotcha (found and removed; see AI-R29).

### AI-E9. A support echo bot loses the owner notification when delivery to the user fails

- **Symptom:** a support bot (a plain Webhook+HTTP, Mirror Echo pattern) — executions fail on
  the `Send Reply` node with `Bad Request: chat not found`. This fails the ENTIRE scenario, and
  the owner notification (`Notify Owner`), which sits AFTER sending the reply to the user,
  **never fires — the bug report is lost**.
- **Cause:** 1) "chat not found" with a valid private `chat_id` means the user hasn't pressed
  `/start` with the bot yet (or blocked/deleted it) — a normal production situation, the bot
  can't initiate the conversation. 2) The Telegram HTTP nodes have no `onError` → any delivery
  failure crashes the whole chain; the owner notification, tied to the success of the previous
  node, dies along with it.
- **Solution:**
  - On ALL outbound sends (`sendMessage` to the user, a start message) — set
    `onError: continueRegularOutput`, so a delivery failure doesn't crash the scenario.
  - **The owner-notification branch must not depend on the reply-to-user succeeding.** Either
    `onError=continue` on `Send Reply` (so the chain proceeds to Notify), or split them in
    parallel from a shared upstream node. Notify reads data from the parser node
    (`$('Parse Reply')`), not from the send — then it always goes through.
  - The root cause of "chat not found" can only be fixed by the user: pressing `/start`. But the
    bug notification should get through regardless.

Status: ⚠️ Caught in live executions and fixed.

### AI-E20. A prompt inside an HTTP node's `jsonBody`: over-escaping crashes the node with `invalid syntax`

- **Symptom:** the system prompt was replaced in an HTTP node (a direct model call, `jsonBody`
  = `={{ JSON.stringify({...}) }}`) — the workflow saved without errors, the API returned 200,
  but on the very first run the node fails with `invalid syntax`, the bot goes silent for
  users.
- **Cause:** the prompt sits inside a JS string in double quotes. In the export via the API,
  the quotes show up as `\"`, and it's easy to "on autopilot" escape them once more (`\\"`),
  producing a broken expression. n8n's validation and save don't catch this — only an actual
  run does.
- **Fix:** escape EXACTLY once: `"` → `\"`, a newline → `\n`. Also check in advance that the
  prompt text doesn't contain `{{` / `}}` — n8n would treat them as its own expression syntax.
- **Mandatory step:** after editing a prompt — a live run, not just a `GET` check (text in the
  export can look correct while the expression is broken). For webhook bots with a secret:
  POST to the webhook with the right header and verify `status='success'` in
  `execution_entity`, otherwise the edit "went through" while the bot is dead.
- **Safety net:** before editing, save the original `jsonBody` to a file — rollback then takes
  a single PUT.

Status: ⚠️ Gotcha (hit it, fixed it).

### AI-E10. The agent lies about "yesterday": the dates are in the data, but the model subtracts them wrong

- **Symptom:** the agent states EXACT numbers (calories, sleep, mood match the DB to the last
  digit), but attaches them to the wrong day: "after yesterday's poor sleep" — though yesterday
  the sleep was 7 hours and the poor sleep actually happened the day before. The user catches a
  fact that never happened and stops trusting the whole review, even the correct numbers.
- **Cause:** the context includes the current time (`[YYYY-MM-DD HH:mm]` in the message header)
  and diary entries with a `date` field in ISO format. The model computes "today minus the
  entry's date" ITSELF — and gets this step wrong, especially when the sample has several
  similar consecutive days. The data isn't a hallucination: a real day is picked, just the
  wrong neighbor.
- **Fix, via the prompt, no pipeline changes:** a rule: "before saying 'yesterday/the day
  before/on the weekend' — subtract the entry's date from the current date and check the word;
  if the day matters for the point being made (poor sleep, a relapse, a record) — name the date
  or weekday: 'on Saturday, 07/25'; if there are no entries for the needed day, say so, don't
  substitute a neighboring one."
- **Why this works:** requiring a stated date makes the error visible to both sides and forces
  the model to perform the subtraction explicitly rather than "by eye." Confirmed: after the
  fix, all numbers and days in the response matched the DB, the poor sleep was renamed from
  "yesterday's" to "recent" (correct).
- **Alternative, if the prompt doesn't help:** feed ready-made labels into the context
  (`the day before yesterday (Sat, 07/25)`) instead of bare dates — then there's nothing to
  compute. More expensive (requires editing context assembly everywhere), so start with the
  prompt.
- **How to catch this:** export the agent's replies and grep for relative words
  (`yesterday|the day before|recently`), then cross-check the numbers mentioned against the
  table for the corresponding dates. An off-by-one-day error shows up immediately.

Status: ⚠️ Gotcha → ✅ fix confirmed.

### AI-E11. A male agent talks about itself in the feminine, mirroring its interlocutor

- **Symptom:** in a conversation with a woman, the agent suddenly talks about ITS OWN actions
  in the feminine: "Name, [logged-feminine] your food for today" (should be masculine). The
  persona breaks, looking like an engine glitch.
- **Cause:** the model agrees Russian verb gender with the nearest animate context — which
  turns out to be the female interlocutor, whose name and gender are mentioned nearby. It
  happens about once in ~150 replies, so it's almost never caught in manual testing.
- **Fix:** an explicit prompt rule: "regardless of the interlocutor's gender, speak about your
  own actions in the masculine: logged, calculated, updated, understood; don't mirror your own
  gender to the interlocutor's. Address the user in THEIR gender (the profile's gender
  field)."
- **How to catch this:** grep the agent's replies for feminine first-person forms
  (`[logged/calculated/updated/understood/noted — feminine forms]`) — for a male agent, any
  match is a defect.

Status: ⚠️ Gotcha (rule added).

### AI-E12. The agent doesn't sense that the conversation paused overnight: the dialogue history carries no timestamps

- **Symptom:** in the evening, plans "for tomorrow" were discussed; the person writes in the
  morning — and the agent continues yesterday's line: "the wind-down for today is almost done,
  leave work until morning," "you'll tackle the next bug tomorrow." But this morning IS already
  that "tomorrow." From the outside it looks like inattentiveness: the person is living in the
  new day, while the interlocutor is still finishing off yesterday evening.
- **Cause:** Postgres Chat Memory stores only role and text (`StoredMessage`), WITHOUT
  timestamps. To the model, the whole history looks like one continuous conversation happening
  "just now." The current time only arrives in the header of the new message — and on its own
  that doesn't solve the problem, because the model doesn't compare it against the history
  unless asked to.
- **Fix via the prompt:** a rule: "the dialogue history has no timestamps; the only reference
  point is the timestamp in the current message's header; hours, a night, or days may have
  passed since the previous reply. If a new day has started (by date or time of day) — plans
  'for tomorrow' from the earlier conversation have become plans for today; don't suggest
  putting things off until morning when morning has already arrived."
- **Alternative (more expensive):** write the time into the saved message text itself, or mix
  `[date time]` into the history during context assembly — then the model has nothing to guess
  at. Requires editing the memory pipeline, so start with the prompt.
- **Related gotcha:** AI-E10 — there the model subtracts diary dates incorrectly; here it fails
  to notice the gap between replies. Both are about LLMs having no innate sense of time:
  anything not stated explicitly gets filled in by guesswork.

Status: ⚠️ Gotcha (rule added).

### AI-E13. An expensive LLM feature without a counter = a hole in the economics

- **Symptom:** food-photo recognition was allowed on both the trial and premium plans, but the
  **limit was only checked for premium** (100/month). A trial user could snap as many photos as
  they wanted over 7 days.
- **Cost at stake:** a vision request ≈ $0.0023 (a 1024px image, detail high, ~1215 input
  tokens). Trial profit is +$0.29 — meaning 60 photos in the trial week wiped it out entirely,
  100 put it in the red.
- **The other half of the problem:** a neighboring route, "estimate macros from a name," had no
  gate at all — no plan check, no counter. With a leaked session token it could be hammered in
  a loop.
- **Fix:** (1) a limit on every plan, not just the top one:
  `LIMITS = { trial:5, basic:10, advanced:30, premium:100, tester:null, super:null }`, an
  unknown plan is treated as the strictest; (2) on the cheap-but-open route — a daily technical
  ceiling (30/day) not shown in the UI: plenty for a real person, an automated loop hits it
  immediately.
- **Rule:** when building an LLM feature, immediately answer two questions — how much does one
  call cost, and what stops someone from calling it a thousand times. If the answer to the
  second is "nothing" — a counter is mandatory, even for a cheap call.
- **Separately:** cross-check the model used in the workflow against the model used in the
  economics calculation — a mismatch can over- or under-state the estimated cost several-fold.

Status: ✅ Fixed (both API workflows).

### AI-E14. The model puts something other than the field's meaning into a structured field

- **Symptom:** the user asked to adjust one specific workout. The trainer sub-agent returned
  `PROGRAM` mode, where the `exercises` field (a list of exercises) ended up containing
  instruction-like phrases: "Leave 1km of easy running," "Remove 3 rounds of squats." The day's
  name came out as "Before the run," and the numeric duration field didn't match what was
  actually laid out.
- **Cause:** the prompt described WHAT to put into the field, but didn't forbid putting
  anything else there, and didn't require cross-checking the numeric field against the
  content. It also didn't distinguish between the "assemble a program" scenario and "adjust
  one workout" — the model packed the edits into the program structure.
- **Prompt-level fix:** (1) an explicit ban on instruction-like verbs in the list, with
  wrong/right examples; (2) the day-name format is a muscle group or split day, not a moment in
  time; (3) a requirement to add warm-up + main work + cool-down and cross-check against the
  duration number before returning; (4) a separate section "when NOT to use PROGRAM mode" —
  edits to a single workout are handled as plain text in the conversation.
- **General rule:** for every structured field in a prompt, it helps to give a wrong/right pair
  based on a real miss, and to explain WHAT the user will see in the UI. An abstract "list of
  exercises" is interpreted by the model more broadly than intended.

Status: ✅ Applied, awaiting verification on live conversations.

### AI-E15. The model sums things its own way: hand the total to the server, leave the model the line items

- **Symptom:** the "Fluids" metric was underreported for a user and "worked sometimes, not
  others." Digging into executions: the agent sent a breakdown
  `{water:3000, coffee:400, other:250}` and a total of `water: 3000`, and later again a
  breakdown `{water:3000, coffee:600}` and again a total of `3000`. So the breakdown was
  tracked correctly, but only the water made it into the total.
- **Cause:** the prompt said "`water` — ANY liquid consumed," but the word `water` has its own
  everyday meaning, and the model keeps falling back on that instead of the instruction's
  definition. It also had to do arithmetic (sum the breakdown) — an operation whose errors show
  up neither in logs nor in a `success` status.
- **Separately:** sometimes a drink didn't get logged at all — "two drip coffees plus a small
  energy drink" produced `{caffeine:320}` with zero milliliters, while the agent cheerfully
  reported that it had logged both drinks. The model's own account of its actions is not proof
  the action happened.
- **Fix:** take the total away from the model entirely. It only tracks the breakdown by type
  (`liquid_mix`), the write node's SQL computes the sum. Manual entries from the app aren't
  overwritten: they live as the difference between the saved total and the sum of the
  breakdown.
- **Migration gotcha:** old days have a total but no breakdown. Blindly doing "total = manual +
  sum" double-counts water. Split into three branches: a breakdown already exists → manual =
  total − breakdown; no breakdown, but the model sent a "water" key → manual = 0 (its breakdown
  is the whole day); no breakdown and no water in what was sent → manual = the whole old total,
  new drinks are added on top.
- **How to test changes like this:** generate the SQL locally, run it on the production DB
  inside a one-off workflow wrapped in `BEGIN … ROLLBACK` for several scenarios (an old day, an
  old day with no water in the input, an in-app tap, a clean day, a manual addition), compare
  against expected numbers, delete the workflow. Cheaper than catching the regression a week
  later from a complaint.
- **Rule:** the model handles facts and line items, the code handles arithmetic and totals.
  Every number the model sums itself will eventually silently drift from reality.

Status: ⚠️ error confirmed in executions; fix applied, SQL verified against 5 scenarios in a
rolled-back transaction, 🟡 awaiting confirmation on live conversations.

### AI-E16. "The last N messages" ≠ "for the period": the context window should be cut by time, not by count

- **Symptom 1 (weekly report).** For a user who barely writes in chat, the weekly report ended
  up built from events three weeks old — the model placed a quit-smoking attempt and a relapse
  on the wrong day of the week. For an active user, the same report looked flawless.
- **Cause.** The conversation was pulled as `ORDER BY id DESC LIMIT 50` — with no date
  boundary. For an active user that's exactly the current week; for a quiet one, over a month.
  Worse, the prompt-assembly code used a regex to **strip the date stamp** out of every
  message, while the block was labeled "CONVERSATION FOR THE WEEK." The model did exactly what
  it was told.
- **A separate detail:** most of the messages in the window were unanswered proactive pings
  from the bot itself ("haven't talked in a while"). For a quiet user, the chat channel carries
  almost no signal, but crowds out everything else.
- **Symptom 2 (support bot).** The support bot asked the same three diagnostic questions eight
  times in a row, even though the person had answered all of them. ONLY the current message was
  sent to the model — there was no history at all.
- **Fix.** Always bound the window by a period: a report window = the report's week, a support
  window = 12 messages and no older than 3 days. Do NOT strip timestamps: without them, the
  model can't tell Tuesday from last month. If there are no messages for the period, say so to
  the model explicitly ("no chat messages, build it from the diary") instead of feeding it a
  different period under the correct heading.
- **How to catch this.** Per user, count: total messages vs. how many fall inside the window. A
  ratio like "87 total / 0 for the week" is the diagnosis right there.
- **Rule.** A block's heading in the prompt is a promise. If it says "for the week," the data
  must actually be for the week; otherwise the model faithfully lies, per your own instruction.

Status: ⚠️ both errors confirmed in executions; fixes applied and verified (support — via a
live run, the report — by counting the window across several users).

### AI-E17. The bot says "I see the photo" without having vision

- **Symptom:** a user sends a screenshot, the bot replies "Thanks, I see the photo 🙏" and
  immediately asks "what exactly is on screen — a blank screen, loading, or an error?" This
  happens for several screenshots in a row.
- **Cause:** only the string `[photo attached]` was sent to the model — nothing else. The image
  was never downloaded or analyzed at all. The phrase "I see the photo" lived in the prompt as
  a politeness and turned into an outright lie.
- **Fix:** a vision pipeline (getFile → download → data URL → a vision request → a text
  description of the image), plus a prompt rule: rely on the analysis, don't ask "what's on
  screen"; if the analysis failed, say so honestly and do NOT claim to see it.
- **Found along the way:** replies contained `**bold**`, but `parse_mode` wasn't set on
  `sendMessage` — the asterisks went to the user as-is. Cleaned up in the response parser, and
  markdown is now banned in the prompt.
- **Rule.** Polite phrasing in a prompt must not assert a capability the bot doesn't have.
  "I see," "I looked," "I checked" — either it's true, or it comes out of the prompt.

Status: ✅ verified with a live run on a real screenshot — the bot recognized the login screen
and walked the user through logging in instead of a list of questions.

### AI-E18. Repairing broken JSON delivered the report, but gutted: fix the cause, not just the symptom

- **Symptom:** for two out of five users, the weekly review arrived nearly empty — a heading,
  no content. The execution was `success`, delivery went through.
- **Cause, two layers.** First: the node had no output ceiling set, and a long review got cut
  off mid-way. Second, sneakier: our own auto-repair filled in the missing brackets — and in
  doing so nested `sections`, `correlations`, and `recommendations` INSIDE `key_insight`. The
  JSON became valid, but the structure was wrong. The app reads those fields at the top level
  and showed nothing.
- **Fix:** `maxTokens` set explicitly (8000), plus the mode
  `textFormat: {textOptions:{type:'json_object'}}` — the API itself guarantees validity. The
  repair step remains a safety net, not the working mechanism.
- **A trap when enabling json_object:** the node returns an already-parsed object, not a
  string. The assembler code, which did `JSON.parse`, got `[object Object]` and silently
  dropped every report. Caught in a dry run before rollout; the assembler was taught to accept
  both forms.
- **Rule:** auto-repair of data must be visible. If it fired, that's an incident, not the norm:
  log it and fix the root cause. "Valid JSON" and "correct structure" are different things —
  it's the second one you need to check.
- **How to test it:** a dry run against the REAL context of the affected users (pulled from the
  old execution), through the whole pipeline down to the assembler, without sending to
  Telegram.

Status: ✅ Verified with an end-to-end run on both affected users' contexts, top-level structure
correct, no nesting.

---

## Patterns from training blueprints (a third-party training course)

The following are not our own production executions, but an analysis of someone else's
reference scenarios (a training case: a fictional computer store, retail). Not run on our
own infrastructure, entries are marked `🟡 Draft`. Their value is as an architectural
sketch for future client bots, not as a proven solution.

### AI-R31 — A pricier model for the client-facing agent, a cheaper one for the internal agent
- **Observation:** in a pair of bots on the same base, the client-facing sales agent runs on
  DeepSeek (conversing with a buyer, persuasiveness and persona matter), while the internal
  analytics bot for store staff runs on Qwen (Alibaba Cloud). The roles are split: the sales
  agent writes to LIVE customers and must sound good, the analyst answers STAFF factual
  questions about data (sales, products) — accuracy matters more than style, so cheapness is
  justified.
- **Rule for future client bots:** don't use one model for the whole project — sort agents by
  the cost of a mistake and the style requirements; the client-facing/selling contour gets a
  stronger model, the internal/reference one gets a cheaper one.
- **Source:** comparing two training workflows from the course (a selling agent vs. an internal
  sales analyst).
- **Status:** 🟡 Draft (course reference).

### AI-R32 — Call-quality analysis: dual transcription + reformatting in Code before the LLM
- **Task:** evaluate an operator's performance from a call recording — with speakers split by
  turn, not as one continuous block of text.
- **Pattern:**
  - **Two independent transcription paths for different input sources.** A voice message in
    Telegram (short, single channel) → the native OpenAI `transcribe` node. A file via a web
    form (a longer call, needs diarization) → AssemblyAI: `Upload` → `Create a transcription` →
    `Wait` → `Get a transcription` (poll). Not a single "one route for everything" — the tool
    is chosen based on the input format.
  - **Between transcription and the LLM sits a Code node that collapses the `utterances` array
    into a readable dialogue** (`Speaker N:\ntext`, joined by `\n\n`). There's no need to hand
    the agent raw JSON with timecodes — the model works with dialogue text, and the markup adds
    no value for it, only noise.
  - **The prompt assigns the role explicitly via context, not an input parameter**: "determine
    which speaker is the operator (greets on behalf of the company, asks questions, solves the
    problem), evaluate only their performance" — instead of asking the operator and customer to
    tag themselves beforehand.
  - **A strict response format with no commentary** (Operator/Topic/Score N/10/Filler
    words/Plus/Minus) — when the result is passed on into a Telegram message, no parsing is
    needed, the whole reply is already a ready-made card of text.
  - **An explicit instruction not to fill in gaps**: "the diarizer sometimes makes mistakes —
    only analyze what came through clearly" and "judge only what was actually said, don't guess
    or soften" — a safeguard against the LLM's typical weakness of filling in missing pieces
    (see [[AI-agents#AI-E10]], [[AI-agents#AI-E12]] — the same class of problem, there about
    dates and time gaps).
  - **Potential application:** a direct overlap with freelance clients' call-quality-control
    needs (the same class of task has already come up in pitches for other projects).
- **Source:** the course's call-analysis training workflow.
- **Status:** 🟡 Draft (course reference, not run).

### AI-R33 — Competitor content → a structured, evidence-backed breakdown → generation with traceable technique attribution
- **Task:** automatically analyze an export of a competitor's posts, extract working content
  techniques, and generate new posts based on them — not copying, but reusing the structure.
- **Pattern:**
  - **Two-phase structured output**: a `Structured Output Parser` forces the model to FIRST
    fill in `analysis.patterns[]` (technique name, format, angle, hook type, `evidence` — how
    many posts back the technique, `reliable` — a self-assessed boolean confidence flag), and
    only THEN generate `posts[]`, where each post has a `pattern` field referencing a specific
    technique from phase one. A finished post can't be generated "out of thin air" — it must
    point to the specific found technique it's based on. That's the traceability: if a
    technique is weak (`evidence` is low, `reliable: false`), it's visible what the generated
    post rests on.
  - **`analysis.reach_note`** — a separate field: "did reach change within the period and how
    was that accounted for" — the model must explicitly report whether it distorted the
    analysis by mixing incomparable reach numbers from old vs. new posts, rather than silently
    averaging them.
  - **`analysis.excluded`** — the number of discarded posts, also an explicit field, not
    hidden — showing the scale of the source data the conclusion was drawn from.
  - **The competitor's posts come from the VK API's `wall.get`** with an `owner_id` starting
    with a minus sign (a negative ID = a group/public page, not a user) — a working way to pull
    a competitor's public-page content without HTML scraping.
  - **The image is generated from a prompt tied to the post**: `image_prompt` (English, with a
    scene) + `image_text` (the verbatim caption on the image) — kept separate because
    image-generation models handle long Russian text in the frame poorly, but handle a scene
    from an English description well.
- **A risk the pattern itself doesn't solve:** "break down a competitor's techniques and write
  posts based on them" is close to copying rather than inspiration; for client work this is
  worth explicitly discussing with the client (a reputational, not just a legal, question).
- **Source:** the course's "content factory: analysis + visual + text" training workflow.
- **Status:** 🟡 Draft (course reference, not run).

### AI-R34 — AI on top of deterministic numbers: SQL finds, the model explains; a strict JSON schema via HTTP Request

- **Pattern:** pattern detection is rule-based in SQL (counters, intervals, period
  comparisons), card text is templated with numbers substituted in. The model receives
  already-computed facts (numbers, distributions, recent events, an "object's memory" — past
  cards with people's comments and their 👍/👎) and produces a hypothesis, what to check, a
  question for a human. Only a human changes statuses; a model failure doesn't block the
  deterministic part.
- **How to call it:** an HTTP Request `POST https://api.openai.com/v1/chat/completions` with
  `authentication: predefinedCredentialType`, `nodeCredentialType: openAiApi` (the product's
  own account, following the "one key per product" rule); the body —
  `response_format: { type: 'json_schema', json_schema: { strict: true, schema: {…,
  additionalProperties: false, required: [all fields]} } }`. For gpt-5.x —
  `reasoning_effort: 'low'` and `max_completion_tokens`, **no `temperature`** (rejected). The
  model comes from `/v1/models` on the corporate client's key (up to gpt-5.6 available; chose
  `gpt-5.4-mini`).
- **Parsing:** JSON with repair (unwrapping/code-fence stripping), a failure `throw`s into the
  shared error channel, not a silent skip (see AI-E13). The parsing node runs in **Run Once for
  Each Item** mode, otherwise after the IF, `$('Prompt').item` loses its pairing.
- **Prompt:** rules — "rely only on the given numbers and name them," "low confidence when data
  is scarce," "no internal codes," plus a paragraph of domain background. For a summary to a
  manager — a separate prompt: "no internal terminology, numbers only where they change the
  takeaway."
- **Source:** a corporate client's workflow — a hypothesis detector for an issue and a weekly
  monitoring digest, 09/10/2026.
- **Status:** ✅ Confirmed — the "Issues" section verified on a phone on 09/10/2026 (7 cards).
