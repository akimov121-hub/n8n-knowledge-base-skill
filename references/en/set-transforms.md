# Set and transforms

Edit Fields (Set), field mapping, and transformation.

---

## ✅ Solutions

_Empty for now. Confirmed solutions get added here after a scenario has been verified._

---

## ⚠️ Errors

### SET-E1. Set typeVersion 3.4 — strict type mode

- **Symptom:** the Set node fails with `'field' expects a number but we got 'expression_string'`.
- **Cause:** typeVersion 3.4 runs in strict mode. Any field whose value is an expression (`=...`)
  must have `"type": "string"`.
- **Fix:** set `"type": "string"` on every expression field. For constants,
  `type: number/boolean` is safe.
- **Related:** the automatic typeVersion bump during an SDK `update_workflow` call breaks this
  behavior — see rest-api-deploy.md DEP-E2.

### SET-E2. Execute Workflow passes ITS OWN input item to the sub-workflow — don't insert nodes between the data source and the call

- **Symptom:** the sub-workflow receives unprepared data instead — the output of the last node
  before Execute Workflow (e.g. `{success:true}` from a Telegram notification) — with
  undefined fields.
- **Cause:** Execute Workflow (passthrough) hands the sub-workflow exactly the items it received
  on its own input. Inserting a "notification" node (a Telegram "🚀 Publishing…" message)
  between the data source and the call swaps out the item.
- **Fix:** wire the notification and the sub-workflow call **in parallel** off the same
  IF/source output (fan-out), not as a chain. Alternatively, restore the item via a
  Set/expression referencing the source right before the call.
- **Source:** the main workflow of a publishing product (incident 06/10/2026); fixed by
  splitting into parallel branches off the readiness-check IF.
- **Status:** ⚠️ Confirmed in practice.

### SET-E3. A Code node (Run Once for All Items) that returns one item silently drops the rest

- **Symptom:** the trigger/query returns N items (e.g. Postgres returned 5 rows), but
  downstream processes **only the first one** — the rest disappear with no error. In practice:
  an hourly cron workflow sent proactive nudges/reminders to **only one** item from the queue
  per tick → items piled up and reminders arrived **up to a day late** (incident 06/25/2026).
- **Cause:** a Code node in *Run Once for All Items* mode runs exactly once. Code like
  `const x=$('Node').item.json; … return [{json:…}]` grabs the **first** item (`.item` resolves
  to the first item under All-Items mode) and returns a one-element array — n8n treats that as
  the node's complete output. Everything downstream then runs with 1 item.
- **Fix:** process the entire input — loop over `$input.all()` (or `items`), match against
  other nodes via `$('Node').all()[i]` by index, and `return out` as an N-element array. Tag
  every pushed item with `pairedItem:{item:i}` so downstream `$('X').item` resolves correctly.
  Skip invalid entries with `continue` (not `return []`). Alternative: *Run Once for Each Item*
  mode (there you `return {json:…}` per item, but skipping an item is trickier).
- **Side-effect-free fan-out testing:** build a temporary mock workflow (Webhook → mock Code
  nodes named like the production ones → the Code nodes under test → Respond
  allIncomingItems), run the exact deployed code through it, then delete it.
- **Source:** a proactive-notifications workflow (Code nodes `Build Prompt`, `Build Send+Book`);
  the project's change log.
- **Status:** ⚠️ Confirmed via execution analysis plus an isolated test (production-verified
  against a multi-candidate queue during the morning tick).

### SET-E4. `$('Node').item` breaks across intermediate nodes once there are ≥2 items — "Multiple matches found" only surfaces as data grows

- **Symptom:** a workflow ran fine for months, then failed with `Multiple matches found` on a
  node using the expression `$('Distant node').item.json.X`, as soon as there were two items
  (a second recipient/row).
- **Cause:** `.item` is a paired-item trace — "my current item ← its ancestor in that node."
  Intermediate nodes (e.g. Postgres executeQuery) don't always preserve pairing; with 1 item
  there's no ambiguity so everything "works," but with ≥2 items n8n can't pick an ancestor and
  errors out. A classic time bomb: single-user testing never catches it.
- **Fix:** feed the consuming node data through **its own input** (`$json.X`): fan out directly
  from the source node (`Source → A` and `Source → B` in parallel) rather than chaining
  `Source → A → B` with a cross-reference via `.item`. If fan-out isn't feasible, use
  `$('Source').all()[$itemIndex]` when 1:1 ordering is guaranteed, or `.first()` when the item
  is known to be a singleton.
- **Source:** the `<workflow_name>` Ping node (a coaching product), first run with 2 report
  recipients, 07/05/2026. Reports survived in the DB (Store uses upsert); only the push
  notification was lost and was resent manually.
- **Status:** ⚠️ Confirmed gotcha (fix confirmed by successful ping delivery).

### SET-E5. A lookup node (HTTP/Postgres) with `executeOnce` on the main line collapses the stream → downstream per-item nodes run only ONCE

- **Symptom:** a "report with no data" — IG reach was populated for only **1 of 16 posts**,
  the rest were empty.
- **Cause:** the chain was `Posts(16) → Channel page[executeOnce] → IG token[executeOnce]
  → IG insights(per-post) → …`. `executeOnce` (and any lookup node that returns 1 item)
  **truncates the stream to 1 item**, so `IG insights`, which is meant to run once per post,
  received 1 input → ran once (for the first post only). Data for the other 15 was never
  requested.
- **Solution:** put one-time lookups (the `t.me/s` page, the token) **at the start of the
  line**, before the node that fans the stream out (the posts SELECT), and reference them by
  name: `$('IG token').first().json.token` / `$('Channel page').first().json.data`. That way
  the per-item node (`IG insights`) receives all 16 posts and runs 16 times, while the lookups
  stay available by reference.
  Order: `IG token(1) → Channel page(1) → Posts(16, SELECT ignores input) → IF → IG insights(16,
  token/page by reference) → Summary(1)`.
- **Rule:** a node that needs to run **once per element** must receive the full stream.
  Anything fetched **once** goes either at the start of the line (referenced by name) or into
  a separate branch; never place a "single-shot" node between the list source and the
  per-item processing.
- **Related:** akin to SET-E2 (Execute Workflow passes its own item), SET-E4 (`.item` breaks
  at ≥2 items).
- **Source:** a publishing product — `<workflow_name>` (digest), 07/19/2026.
- **Status:** ✅ Fixed (sandbox test: IG reach on 15/16 posts).
