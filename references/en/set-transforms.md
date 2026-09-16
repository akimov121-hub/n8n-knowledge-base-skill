# Set and Transforms

Edit Fields (Set), field mapping and transformation.

---

## ✅ Solutions

_Empty for now. Confirmed solutions are added here once a scenario has been verified._

---

## ⚠️ Errors

### SET-E1. Set typeVersion 3.4 — strict type mode

- **Symptom:** the Set node fails with `'field' expects a number but we got 'expression_string'`.
- **Cause:** typeVersion 3.4 has strict mode. A field whose value is an expression (`=...`)
  must have `"type": "string"`.
- **Fix:** set `"type": "string"` for every expression field. For constants,
  `type: number/boolean` is safe.
- **Related:** automatic typeVersion bumps on an SDK `update_workflow` break this behavior — see
  rest-api-deploy.md DEP-E2.

### SET-E2. Execute Workflow passes its OWN input item to the sub-workflow — don't insert nodes between the data source and the call

- **Symptom:** the sub-workflow receives not the prepared data but the output of the last node
  before Execute Workflow (e.g. `{success:true}` from a Telegram notification) — fields come back
  undefined.
- **Cause:** Execute Workflow (passthrough) forwards to the sub-workflow exactly the items it
  received on its own input. Inserting a "notification" node (a Telegram "🚀 Publishing…" message)
  between the data source and the call replaces the item.
- **Fix:** run the notification and the sub-workflow call **in parallel** off the same output of
  the IF/source node (fan-out), not as a chain. Alternatively, restore the item via a Set/expression
  pointing back to the source before the call.
- **Source:** the main workflow of a publishing product (incident 10.06.2026); fix — parallel
  branches from the readiness-check IF node.
- **Status:** ⚠️ Confirmed in practice.

### SET-E3. A Code node (Run Once for All Items) that returns one item silently drops the rest

- **Symptom:** a trigger/query returns N items (e.g. Postgres returned 5 rows), but downstream
  processes **only the first one** — the rest disappear without an error. In practice: an hourly
  cron workflow sent proactive messages/reminders to **only one** item from the queue per tick →
  a backlog built up and delivery lagged **by up to a day** (incident 25.06.2026).
- **Cause:** a Code node in *Run Once for All Items* mode runs once. Code like
  `const x=$('Node').item.json; … return [{json:…}]` takes the **first** item (`.item` = the first
  one under All-Items) and returns a single-element array — n8n treats that as the node's complete
  output. The chain continues with just 1 element from then on.
- **Fix:** process the entire input — loop over `$input.all()` (or `items`), match against other
  nodes via `$('Node').all()[i]` by index, and `return out` as an array of N elements. Tag each
  pushed item with `pairedItem:{item:i}` so downstream `$('X').item` resolves correctly. Skip
  invalid ones with `continue` (not `return []`). Alternative — *Run Once for Each Item* mode
  (then `return {json:…}` per item, though skipping an element is trickier there).
- **Testing fan-out without side effects:** a temporary mock workflow (Webhook → Code nodes mocked
  with production names → the Code nodes under test → Respond allIncomingItems), run the exact
  deployed code, delete afterward.
- **Source:** the proactive-notifications workflow (Code nodes `Build Prompt`, `Build Send+Book`);
  the project's change log.
- **Status:** ⚠️ Confirmed by execution analysis plus an isolated test (verified in production on
  a multi-candidate queue — the morning tick).

### SET-E4. `$('Node').item` breaks across intermediate nodes once there are ≥2 items — "Multiple matches found" only surfaces as data grows

- **Symptom:** a workflow had run fine for months, then failed with `Multiple matches found` on a
  node using the expression `$('Far node').item.json.X`, as soon as there were two items (a second
  recipient/row).
- **Cause:** `.item` is paired-item tracking — "my current item ← its ancestor in that node."
  Intermediate nodes (e.g. Postgres executeQuery) don't always preserve pairing; with 1 item there
  is no ambiguity so everything "works," but with ≥2 items n8n can't pick an ancestor → error. A
  classic slow-burn trap: tests with a single user never catch it.
- **Fix:** feed the consuming node data through **its own input** (`$json.X`): fan out directly
  from the source node (`Source → A` and `Source → B` in parallel) rather than chaining
  `Source → A → B` with a cross-reference `.item`. If fan-out doesn't fit — use
  `$('Source').all()[$itemIndex]` when 1:1 ordering is guaranteed, or `.first()` when there is
  definitely only one item.
- **Source:** `<workflow_name>`, the Ping node (a coaching product), first run with 2 report
  recipients, 05.07.2026. The reports survived in the DB (Store — upsert); only the push was lost
  and was resent manually.
- **Status:** ⚠️ Confirmed gotcha (fix confirmed by successful ping delivery).

### SET-E5. A lookup node (HTTP/Postgres) with `executeOnce` in the main line collapses the stream → per-item nodes downstream run only ONCE

- **Symptom:** "report with no data" — IG reach was populated for only **1 post out of 16**, the
  rest were blank.
- **Cause:** the chain was `Posts(16) → Channel page[executeOnce] → IG token[executeOnce]
  → IG insights(per-post) → …`. `executeOnce` (and any lookup node that returns 1 item)
  **truncates the stream to 1 item**, so `IG insights`, which is supposed to run once per post,
  received 1 input → ran once (for the first post). Data for the other 15 was never fetched.
- **Fix:** place one-off lookups (the `t.me/s` page, the token) **at the start of the line**,
  before the node that expands the stream (the posts SELECT), and reference them by name —
  `$('IG token').first().json.token` / `$('Channel page').first().json.data`. Then the per-item
  node (`IG insights`) receives all 16 posts and runs 16 times, while the lookups stay available
  by reference.
  Order: `IG token(1) → Channel page(1) → Posts(16, SELECT ignores its input) → IF → IG insights(16,
  token/page by reference) → Summary(1)`.
- **Rule:** a node that needs to run **per element** must receive the full stream. Anything fetched
  **once** goes either at the start of the line (referenced by name) or in a separate branch — never
  a "one-off" node placed between a list source and per-item processing.
- **Related:** akin to SET-E2 (Execute Workflow passes its own item) and SET-E4 (`.item` breaks with
  ≥2 items).
- **Source:** a publishing product — `<workflow_name>` (digest), 19.07.2026.
- **Status:** ✅ Fixed (sandbox test: IG reach populated for 15/16 posts).

### SET-E6. A Code node with `executeOnce` sees only the first element: `$input.all()` returns one out of N ⚠️

- **Symptom:** a broadcast went out to three recipients (the Telegram node produced 3 items, each
  with its own `message_id`), but the "Collect" Code node right after it saved only the first chat.
  The "Acknowledge/Close" buttons only edited the message for the first recipient; the node never
  saw the delivery failure for the second and third, so retries never fired. Execution status:
  success.
- **Cause:** the Code node, running in *Run Once for All Items* mode, had the `executeOnce: true`
  flag set ("Execute Once" in the node's settings). The flag doesn't mean "run the code once" — the
  code already runs once in that mode — it means "pass the node only the first input item." Inside
  the node, `$input.all()` is a single-element array; meanwhile `$('Other node').all()` still
  returns all of them, which masks the bug: comparing by index yields `undefined` with no exception
  raised.
- **Fix:** never set `executeOnce` on Code nodes that read `$input.all()`. The flag is only meant
  for lookup nodes with no input fan-out (a single-query Postgres/HTTP call) — and even then, watch
  out for SET-E5.
- **How to find it:** scan the scenarios' JSON and flag nodes where `type == code && executeOnce &&
  '$input.all()' in jsCode` (as of 15.09.2026, three turned up: "Collect" and "Remove logged" in
  one workflow (`<workflow_name>`), "Summary" in another (escalation)).
- **Why testing missed it:** at test time there was only one recipient — a single element behaves
  the same with or without the flag. Broadcasts should be tested with at least two recipients.
- **Source:** a corporate client, `<workflow_name>`, 14–15.09.2026.
- **Status:** ⚠️ Logged 15.09.2026.
