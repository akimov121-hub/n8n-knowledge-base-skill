# Set and transformations

Edit Fields (Set), field mapping and transformation.

---

## ✅ Solutions

_Empty for now. Confirmed solutions are added here after the scenario has been verified._

---

## ⚠️ Errors

### SET-E1. Set typeVersion 3.4 — strict type mode

- **Symptom:** the Set node fails with `'field' expects a number but we got 'expression_string'`.
- **Cause:** typeVersion 3.4 runs in strict mode. Any field whose value is an expression (`=...`)
  must have `"type": "string"`.
- **Fix:** set `"type": "string"` on all expression-valued fields. For constants,
  `type: number/boolean` is safe.
- **Related:** automatic typeVersion bumping during the SDK update_workflow breaks behavior — see
  rest-api-deploy.md DEP-E2.

### SET-E2. Execute Workflow passes ITS OWN input item to the sub-workflow — don't insert nodes between the data source and the call

- **Symptom:** the sub-workflow receives not the prepared data but the output of the last node before
  Execute Workflow (e.g. `{success:true}` from a Telegram notification) — fields are undefined.
- **Cause:** Execute Workflow (passthrough) forwards to the sub exactly the items it received as
  input. Inserting a "notification" node (Telegram "🚀 Publishing…") between the data source
  and the call replaces the item.
- **Fix:** wire the notification and the sub-workflow call **in parallel** from a single output of
  the IF/source (fan-out), not as a chain. Alternatively, restore the item before the call via a
  Set/expression referencing the source.
- **Source:** the main workflow of a publishing product (incident 2026-06-10); the fix was parallel
  branches off the IF readiness check.
- **Status:** ⚠️ Confirmed in practice.

### SET-E3. A Code node (Run Once for All Items) that returns a single item silently drops the rest

- **Symptom:** the trigger/query returns N items (e.g. Postgres returned 5 rows), but the downstream
  processes **only the first** — the rest disappear without an error. In practice: a cron workflow
  (hourly) sent proactive messages/reminders to **only one** entry from the queue per tick →
  backlog buildup and **delays of up to a full day** (incident 2026-06-25).
- **Cause:** a Code node in *Run Once for All Items* mode runs once. Code like
  `const x=$('Node').item.json; … return [{json:…}]` takes the **first** item (`.item` = the first
  one in All-Items mode) and returns an array with a single element — n8n treats this as the node's
  complete output. The chain downstream then continues with just 1 element.
- **Fix:** process the entire input — loop over `$input.all()` (or `items`), correlating with other
  nodes via `$('Node').all()[i]` by index, and `return out` as an array of N elements. Set
  `pairedItem:{item:i}` on every push so that downstream `$('X').item` resolves correctly. Skip
  invalid ones with `continue` (not `return []`). An alternative is the *Run Once for Each Item*
  mode (then `return {json:…}` per item, but "skipping" an element is harder).
- **Verifying fan-out without side effects:** a temporary mock workflow (Webhook → Code mocks named
  the same as in production → the Code nodes under test → Respond allIncomingItems), run exactly the
  deployed code, then delete it afterward.
- **Source:** the proactive-notifications workflow (Code nodes `Build Prompt`, `Build Send+Book`);
  the project changelog.
- **Status:** ⚠️ Confirmed via execution analysis + an isolated test (production verification on a
  multi-candidate queue — the morning tick).
