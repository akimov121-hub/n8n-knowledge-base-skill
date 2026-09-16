# Postgres

Postgres nodes: `executeQuery`, Postgres Chat Memory.

---

## ✅ Solutions

### PG-R1. Controlling a value's SQL type — embed `{{ Number(...) }}` directly in the query text

- **Context:** need to pass a numeric value (id, message_id) into a query so that it lands in
  the DB as a **number**, not a string.
- **Fact (verified):** cross-node references in `queryReplacement` **are evaluated** —
  `$('Node').first().json.field` resolves normally. So feeding a value through
  `queryReplacement` with a cross-node ref is a working path.
- **Nuance:** `queryReplacement` passes the value as parameter `$1`, and the evaluated
  result often comes out as a **string**. If the target column is `bigint`/`numeric` — this
  causes an error (see PG-E1). To guarantee the value goes in as a number, two options:
  1. Wrap it in `Number(...)` right inside the `queryReplacement` expression.
  2. Embed a numeric literal directly in the SQL text:

```sql
VALUES ({{ Number($('Telegram Trigger').first().json.message.message_id) }})
```

Status: ✅ Confirmed.

### PG-R2. Arbitrary SQL / DDL via a temporary webhook workflow

- **Context:** n8n's public REST API cannot execute arbitrary SQL; Postgres Chat Memory
  nodes create tables lazily (on the first message); sometimes you need to create a table
  ahead of time, inspect the schema, do a "dry run" before a `DELETE`, or seed a lookup table
  by hand.
- **Solution:** a one-off workflow `Webhook (POST) → Postgres executeQuery → Respond to
  Webhook (allIncomingItems)`. Create it via `POST /api/v1/workflows`, activate it, hit it
  with curl, read the result from the response, then **deactivate and delete it** (don't
  leave an active public webhook lying around — a small but real risk).
- **Uses:** `CREATE TABLE`, checking the schema (`SELECT … FROM information_schema.columns`),
  a dry run of a cleanup (`SELECT count(*)` instead of `DELETE` — see what would be deleted),
  seeding lookup tables with `INSERT` using dollar-quoting (`$Q$text$Q$`) for text containing
  quotes.
- **Nuance:** the table name can be substituted via an expression directly in the
  `executeQuery` text (`{{ $json.table }}`) — n8n resolves the expression before sending it to
  the DB. Safe as long as the name comes from your own controlled list, not from user input.

Status: ✅ Confirmed.

### PG-R3. Inserting into the smallest free `id` (gap-fill, number reuse)

- **Context:** the user is shown a record's `id` as a "number" (bot memory, reminders, any
  personal list). With a plain `serial`, deletions leave gaps and new records always go to the
  end. The goal is gap-free numbering where **a freed number is taken by the next new
  record**.
- **Solution:** don't touch the schema or rebuild the PK. Instead of `serial`, insert an
  **explicit `id` = the smallest free number** via `generate_series` + an anti-join. The
  actual PK then equals the displayed number — `delete`/`update` address it directly.

```sql
INSERT INTO <table> (id, ...) VALUES (
  (SELECT COALESCE(MIN(s.n),1)
     FROM generate_series(1, COALESCE((SELECT MAX(id) FROM <table>),0)+1) s(n)
     LEFT JOIN <table> m ON m.id = s.n
    WHERE m.id IS NULL),
  ...
) RETURNING id;
```

- **Behavior:** empty → id 1; {2,3,4} exist → new record takes 1, then 5; delete from the
  middle → the next new record takes it. Existing numbers never change.
- **Nuances:** the scalar subquery in `VALUES` sees the table BEFORE the insert — correct
  behavior. Since we always insert an explicit `id`, the `serial` counter is never used → no
  collisions (important that ALL insert paths go through this pattern). For multi-user setups,
  add `WHERE telegram_id=...` to both parts of the subquery.
- **Related:** editing a record "in place" is a separate `UPDATE` by id, not delete+insert
  (otherwise the number would drift to the end).

Status: ✅ Confirmed in production.

### PG-R4. Idempotent action via atomic status capture (protection against double-tapping a button)

- **Context:** an inline button that triggers an irreversible action (publish, charge, send)
  produces **two parallel runs** on a double-tap. Both read the "ready for action" status
  before the first one changes it → the action executes twice (double post, etc.).
- **Solution:** not "SELECT status → IF", but an **atomic UPDATE with a condition** on the
  status row:

```sql
UPDATE drafts SET stage='processing', updated_at=now()
WHERE chat_id=<chat_id> AND stage='await_review'
RETURNING <needed columns>;
```

  Postgres serializes concurrent UPDATEs on a row: the first flips
  `await_review→processing` and returns the row, the second no longer finds `await_review` →
  0 rows. **Important (see PG-E2):** on 0 rows, the node emits NOT 0 items but a service item
  `{success:true}` — so the guard only works paired with an IF on the status/key field
  (`stage='processing'` / `chat_id notEmpty`) right after the UPDATE. No timers/debounces
  needed — the guarantee comes from the DB level plus a mandatory IF.
- **Nuance:** if the action can partially fail after the capture — the status stays stuck at
  `processing`; plan a rollback or drive it to a final status on every branch (onError
  continue).

Status: ✅ Confirmed.

### PG-R5. Text parameters via a `queryReplacement` array expression

- **Pattern:** pass user text into SQL NOT via string concatenation with `'`→`''` escaping,
  but as parameters `$1..$n`: in the node's options, `queryReplacement` = `={{ [expr1, expr2, ...] }}`
  (a single expression that returns an array).
- **Why an array:** the comma-separated string form of `queryReplacement` breaks on commas
  inside a value; an array expression carries commas, quotes, HTML, and line breaks without
  loss.
- **Verified live:** `SELECT $1::text...` + `={{ [$json.body.a, ...] }}` — values with
  commas/quotes/tags came back 1:1.
- **Limitation:** values come out as strings — for `bigint`/`numeric` columns still embed
  `{{ Number(...) }}` as a literal in the SQL (PG-R1, PG-E1).
- **Use case:** the publisher product — all recording of post text/instructions/caption.

Status: ✅ Confirmed.

### PG-R6. Webhook response as a list/object — `json_agg`/scalar-subquery, guaranteed 1 row

- **Context:** in the Mini App API (webhook + Respond), `list`/`get` actions read from the DB.
  If a SELECT returns 0 rows — the branch may not continue, and the `Respond to Webhook` node
  never fires → **the request hangs** on the frontend (see also the ambiguity in PG-E2: 0 rows
  → sometimes `{success:true}`, sometimes empty).
- **Solution:** wrap the query so the node **always returns exactly 1 row**:
  - list → `SELECT COALESCE(json_agg(x), '[]'::json) AS __list FROM (SELECT ... ) x;` —
    empty → `[]`, otherwise an array.
  - object → `SELECT COALESCE((SELECT to_jsonb(t) FROM (SELECT ...) t), '{}'::jsonb) AS __obj;` —
    no record → `{}`.

  A Code node then normalizes (`if(typeof v==='string') JSON.parse`), and Respond returns the
  array/object. Bonus: `bigint` inside `json_agg`/`to_jsonb` serializes as a **number** (not a
  string) — fixes epoch-ms timestamps for the frontend.

Status: ✅ Confirmed.

---

## ⚠️ Errors

### PG-E1. `invalid input syntax for type bigint`

- **Symptom:** Postgres executeQuery fails with `invalid input syntax for type bigint`.
- **Cause:** `queryReplacement` passes the evaluated value as parameter `$1` **as a string**;
  if the column is `bigint`/`numeric` and the string is empty or non-numeric — Postgres won't
  cast it and fails. (The cause is specifically the value's type, not that the cross-node
  expression "isn't evaluated" — it IS evaluated, see PG-R1.)
- **Fix:** wrap the value in `Number(...)` inside `queryReplacement`, or embed a numeric
  literal directly in the SQL via `{{ Number(...) }}`. Also make sure the value isn't empty.

### PG-E2. executeQuery on 0 rows emits item {success:true} instead of 0 items

- **Symptom:** downstream nodes after a Postgres node execute even though the query
  (UPDATE...RETURNING / SELECT) returned no rows; the item contains `{"success": true}` with
  none of the query's fields. If an Execute Workflow sits after the node — the sub-workflow
  gets called with this garbage item (`image_url=undefined`, etc.).
- **Cause:** n8n's Postgres executeQuery emits a service item `{success:true}` on an empty
  result — you cannot rely on "0 rows → branch won't run".
- **Fix:** after every "may return 0 rows" query — an IF-guard on a key field
  (`{{ $json.chat_id }}` notEmpty, or a status check), and only then the action.
- **Status:** ⚠️ Confirmed in practice.

### PG-E3. Dollar-quoting `$tag$…$tag$` with a numeric value breaks n8n's parameter scanner

- **Symptom:** the Postgres node fails with `Variable $<big number> exceeds supported maximum of $100000`,
  even though the query has no such parameter number. Example: inserting a Telegram bot token
  `<BOT_TOKEN>` via `VALUES ('bot_token', $tok$<BOT_TOKEN>$tok$)`.
- **Cause:** n8n scans the SQL for `$N` placeholders (queryReplacement) **before** sending it
  to Postgres and doesn't understand dollar-quoting. The tag's closing `$` plus the value's
  leading digits (`…$tok$<token digits>`) get read as `$<number>` → a "parameter number" out
  of range.
- **Fix: never** insert values via dollar-quoting — pass them as a native `$1` parameter via
  `options.queryReplacement` (see PG-R5). Especially critical for values starting with digits
  (ids, tokens).
- **Status:** ⚠️ Confirmed in practice.

### PG-E4. Multi-statement SQL as an n8n expression (with `{{ }}`) → "invalid syntax"

- **Symptom:** a Postgres node with a multi-statement query (several `;`), where values are
  embedded via `{{ … }}` (i.e. the whole query text became an n8n expression with a leading
  `=`), fails with `invalid syntax` before it even reaches Postgres. A single statement with
  the same `{{ }}` works fine.
- **Cause:** when the entire SQL is one n8n expression, the expression parser trips over
  complex multi-statement text (especially with regex/special characters inside `{{ }}`).
  This is a bug in n8n's expression engine, not Postgres.
- **Fix: one statement — one node** + values as native `$1` parameters (queryReplacement);
  keep `{{ }}` only in separate fields (params / chatId / text of a Telegram node), NOT in the
  SQL text. Break a multi-step operation (UPSERT + flag + log + select) into a chain of nodes.
  For regex/sanitizing — a separate Code node (Prep), with the SQL referencing already-clean
  values.
- **Status:** ✅ Confirmed (after splitting into single statements everything worked; the
  grant/revoke/add/decline smoke cycle is green).

### PG-E5. executeQuery with multi-statement SQL returns the result of the FIRST statement, not the last

- **Symptom:** a Postgres node runs one query made of several statements
  (`INSERT…; UPDATE…; SELECT…;`). All statements **execute** (records go through), but the
  node's output item contains the result of the **first** statement. A node further down that
  reads `$('This node').first().json.<field from the last SELECT>` gets `undefined`. Example:
  a grant node (3 writes + a final `SELECT bot_name`), and the message renders as "to bot
  «undefined»".
- **Check (read-only):** `SELECT 'a' AS first_stmt; SELECT 'b' AS second;` → the node returned
  `{first_stmt:'a'}`.
- **Fix:** fetch the value needed downstream with a **separate node** (a single SELECT) — its
  result is correct. Leave multi-statement writes as-is if their result is never read. In the
  Mini App API this is already split into separate nodes (Grant SQL separate from Grant
  BotName).
- **Related:** reinforces PG-R6 (one node = one needed result) and PG-E4.
- **Status:** ✅ Confirmed (read-only test + splitting into nodes).

---

## ✅ Solutions (continued)

### PG-R7. "One row per user" draft — reset carried-over fields when starting a new item

- **Context:** wizard state (a post draft) is stored as one row of `pub_drafts` per `user_id`
  (UPSERT `ON CONFLICT (user_id)`). The same row gets reused for the next post.
- **Gotcha:** the UPSERT, when generating text for a NEW post, updated `mode/idea/header/post/stage`
  but **left the previous post's derived fields untouched** (`image_url`, `ig_caption`).
  Result: the new post inherited the old cover image/caption; the frontend (polling
  `draft/get`) picked up the stale image and offered "Regenerate" instead of creating a new
  one.
- **Solution:** in the `ON CONFLICT DO UPDATE` for a new item, **explicitly clear all
  carried-over/derived fields**:
  `... SET mode=$1, idea=$2, header=$3, post=$4, stage='text', image_url=NULL, ig_caption=NULL, updated_at=now()`.
  General rule: when reusing a state row, starting a new "lifecycle" should reset everything
  that belonged to the previous one, not just the fields being overwritten.
- **Client-side guard (extra):** mirror the clearing on the frontend too
  (`S.draft.imageUrl=''; igCaption=''`), otherwise a resilient polling watcher will pick up
  the stale value again.

Status: ✅ Confirmed.

### PG-R8. Free-form JSONB merge (`data || EXCLUDED.data`) — new frontend metrics without schema migrations

- **Context:** the frontend (the "Diary" Mini App) writes daily metrics; the set of keys grows
  from version to version (added `stress`, `anxiety`, `focus`, `sleep_quality`,
  `meditation_min`, a `reflection` object, `nutrition` became a number). Storing each key as
  its own column → a migration for every new metric.
- **Solution:** keep the metrics in one JSONB column `data` and do a **partial merge** in the
  UPSERT: `INSERT … ON CONFLICT (telegram_id, day) DO UPDATE SET data = daily_checkin.data || EXCLUDED.data`.
  The `||` operator for jsonb is a shallow merge: new/changed keys get overwritten, old ones
  are preserved. `checkin/history` returns the whole `data` as-is → new keys come back to the
  frontend with no backend changes.
- **What this buys:** adding a metric on the frontend requires NEITHER a DB migration NOR a
  workflow change — just verification. Key-name validation is a light regex
  (`^[a-z][a-z0-9_]{0,40}$`), no hard whitelist. Values of any type (number/scalar/nested
  object) pass through.
- **Limits:** `||` only merges the top level — a nested object (`reflection`) gets overwritten
  wholesale, not merged key-by-key. If a deep merge is needed — `jsonb_set` or a
  `jsonb_deep_merge` function. The downside of this approach — no DB-level schema (types/
  required keys aren't guaranteed), so keep critical/indexed fields as separate columns and
  use JSONB only for the "extensible tail" of metrics.

Status: ✅ Confirmed.

### PG-R9. Scheduling idempotency: filter by stage when inserting from a reused draft row

- **Symptom:** the same post appeared in the queue twice (two `publisher_queue` rows,
  identical content and `publish_at`).
- **Cause:** the `publish/schedule` action inserted a row by reading the draft
  `... FROM pub_drafts WHERE post<>'' AND image_url<>''` — **with no stage check**. After the
  first scheduling, the stage → `queued`, but `post`/`image_url` remain → a repeat call
  (double-tap on the time button / retry / webhook redelivery) passes the filter again and
  inserts a duplicate. One call = one row (`INSERT ... SELECT FROM src`), so a duplicate means
  two calls.
- **Solution:** add an active-stage filter to `src` — insert only from a NOT-yet-completed
  draft: `... AND stage IN ('idea','text','image','preview')`. After the first scheduling
  (`stage='queued'`), a repeat call finds 0 rows in `src` → `INSERT ... SELECT` inserts nothing
  → idempotent. General rule: **any "one-shot" insert from a state row that keeps living must
  be gated by a stage/"already processed" flag**, otherwise a repeated request produces
  duplicates.
- **Extra client-side guard:** an in-flight flag against double-taps — suppresses the
  duplicate before it even reaches the backend. A unique index gives full race protection, but
  the stage filter covers the real-world case (a sequential double-tap).

Status: ✅ Confirmed.

### PG-R10. Atomic payment activation: status capture + charge + access grant in one CTE (retry-safe)

- **Context:** the payment provider's callback (Robokassa Result) must mark a payment `paid`,
  extend the subscription, and grant access — idempotently, since the provider **retries**
  the callback until it gets `OK`. Separate nodes (Mark Paid → Activate → Grant) are unsafe:
  if activation fails AFTER mark-paid, on retry mark-paid returns 0 rows ("already paid") →
  activation gets skipped → **money charged, no access, permanently** (visible only in an
  error log).
- **Solution:** a single Postgres node with a data-modifying CTE: status capture (as in
  PG-R4) as a "gate", the rest of the steps read data FROM it:

```sql
WITH paid AS (
  UPDATE payments SET status='paid', paid_at=now()
  WHERE inv_id=$1 AND status<>'paid'
  RETURNING inv_id, telegram_id, plan
),
act AS (
  INSERT INTO profile (telegram_id, plan, plan_until)
  SELECT telegram_id, plan, now()+interval '30 days' FROM paid
  ON CONFLICT (telegram_id) DO UPDATE SET plan=EXCLUDED.plan,
    plan_until=GREATEST(now(),COALESCE(profile.plan_until,now()))+interval '30 days'
  RETURNING telegram_id
),
granted AS (
  INSERT INTO bot_access (telegram_id, bot, active)
  SELECT telegram_id,'<product_slug>',true FROM paid
  ON CONFLICT (telegram_id, bot) DO UPDATE SET active=true RETURNING telegram_id
)
SELECT inv_id, telegram_id, plan FROM paid;
```

  Guarantees: (1) `act`/`granted` read `FROM paid` → they fire ONLY when the capture yielded a
  row; (2) it's all one statement = one transaction → a partial failure rolls back
  EVERYTHING, and the retry repeats it wholesale; (3) a retry after success: `paid` is empty →
  0 rows → the downstream IF on `inv_id` routes to "already processed" (Resp OK without
  re-granting/double-charging). Activation takes `plan`/`telegram_id` from the payment ROW
  (server-side values), not from the callback parameters — also a defense against parameter
  tampering.
- **Nuances:** the node must have `alwaysOutputData` (on a retry, 0 rows → an empty item → the
  IF routes to "OK"; otherwise there are no input items → downstream doesn't run → the
  provider gets no response and retries forever). A CTE name must not collide with an SQL
  keyword (`grant` → `granted`).
- **Related:** extends PG-R4 to several tables in one transaction.

Status: 🟡 Draft — applied and validated by markers, awaiting a live payment to confirm.

### PG-R11. Changing a subscription plan: charge the difference + cancel the previous parent transaction

- **Task:** a subscriber wants a more expensive plan. The naive approach ("just pay for the
  new one") causes two problems: (1) the payment provider ends up with TWO parent
  transactions, and the recurring charge keeps billing the old, cheaper one; (2) the term is
  computed as "remaining days + 30 more", even though the person already paid full price.
- **Top-up calculation:** `credit = current_price * days_left / 30`, `amount_due = max(1,
  new_price − credit)`. Cap days at the period length (`LEAST(30, …)`) — otherwise an
  accumulated time buffer would zero out the payment.
- **Canceling the previous subscription — in the same CTE as activation:**

```sql
superseded AS (
  UPDATE payments p SET superseded_at=now()
  FROM paid
  WHERE p.telegram_id=paid.telegram_id AND p.recurring=true AND p.status='paid'
    AND p.prev_inv_id IS NULL AND p.superseded_at IS NULL AND p.inv_id <> paid.inv_id
    AND paid.recurring=true
  RETURNING p.inv_id
)
```

  And the selection for billing takes the **most recent non-superseded** one:
  `... AND superseded_at IS NULL ORDER BY inv_id DESC LIMIT 1` (it was `ASC` = always the
  first one, i.e. forever the old plan).
- **Term on a plan change ≠ term on a renewal:** `CASE WHEN EXCLUDED.plan <> profile.plan
  THEN now() + 30d ELSE GREATEST(now(), plan_until) + 30d END`. Otherwise an upgrade gifts a
  double period.
- **Downgrading a plan** is done from the next billing cycle (just change the plan in the
  profile, no money touched) — this way no refunds or offer edits are needed.
- **The recurring charge amount is taken from the plan, not from the parent payment** —
  otherwise, after an upgrade with a top-up, the person would keep paying the top-up amount
  forever. Open question for the payment provider: is it allowed to charge MORE than the
  parent amount (undocumented) — to clarify when connecting the service.
- **Store consent for auto-charges in a separate table** (`recurring_consents`: who, plan,
  amount, period, offer version and URL, consent text, timestamp) and link it to the payment —
  payment systems require this when enabling recurring billing.

Status: 🟡 Draft — deployed into the payment flow, to be confirmed after the service is
connected and the first live upgrade.

---

## ⚠️ Errors (continued)

### PG-E6. After a Postgres node (INSERT/UPDATE) `$json` has NONE of your workflow's fields — an IF after it must reference the source node

- **Symptom:** an IF node after a Postgres INSERT always goes to the false branch; a feature
  (e.g. a warning) silently never fires.
- **Cause:** a Postgres node's output = the query result (for INSERT — empty/service data),
  not the item that went into it. `{{ $json.warn }}` after `Plan Count` reads a field that
  doesn't exist.
- **Fix:** in nodes after Postgres, reference the source node of the decision:
  `{{ $('Plan Gate').first().json.warn }}`. Plus `alwaysOutputData: true` on INSERT nodes
  inside the chain — otherwise an empty output breaks the whole downstream pipeline.
- **Status:** ⚠️ Gotcha (confirmed by the node's structure).

### PG-E7. Postgres regex quantifier capped at 255 — `.{300}` and `{256,}` break the query

- **Symptom:** `invalid regular expression: invalid repetition count(s)` on
  `regexp_replace`/`~`; the node failed in production after replying to the user (a
  post-processor).
- **Cause:** in PostgreSQL's POSIX/ARE regex, neither bound of `{m,n}` may exceed **255**.
  `.{300}` and `.{301,}` are invalid. Sneaky: the error isn't raised when writing the query
  into the node, but at runtime — and only on rows where that regex branch actually executes.
- **Fix:** keep bounds ≤255; express "longer than N>255" via concatenation: `.{255}.` (255
  characters + one more). Truncating to "~300" as `(.{255})[^\n]*` → `\1…`.
- **Status:** ⚠️ Gotcha (fix confirmed by a control run, dirty=0).

### PG-E8. `RETURNING` only returns the listed columns — a downstream node reading `$input` gets undefined

- **Symptom:** a node after a Postgres INSERT/UPDATE with `RETURNING inv_id` builds a
  request/body from `$input.item.json.telegram_id/plan/amount` — all `undefined`; in
  production this fails SILENTLY (the MD5 signature is computed from a string containing
  "undefined", `chat_id: NaN`, URL `api.telegram.org/botundefined/...`), and the node reports
  "success".
- **Cause:** the output of a Postgres `executeQuery` = only the `RETURNING` columns (e.g.
  `{inv_id}`); the rest of the original item's fields aren't there. `$input`/`$json` in the
  next node are exactly that.
- **Fix:** get the needed fields from the node WHERE they came from, via the paired item:
  `$('Prep').item.json.telegram_id` (not `$input`; with multiple items — use `.item`, not
  `.first()`, so pairing matches up). From `$input` — only what's actually in `RETURNING`
  (`inv_id`).
- **Status:** ⚠️ Gotcha (fix verified by markers).

### PG-E9. A key-lookup SELECT in a webhook chain without `alwaysOutputData` — a landmine for NEW users (0 rows = broken chain)

- **Symptom:** a production webhook works for every tester, but silently breaks for every new
  user: execution `success`, the last node is a Postgres SELECT with `items=0`, no response to
  the client → the frontend shows "Invalid server response".
- **Cause:** `SELECT ... FROM profile WHERE telegram_id=$1` for a new user (no row yet)
  returns 0 rows → a node without `alwaysOutputData` emits 0 items → everything after it,
  including Respond, doesn't run. Sneaky: EVERY existing account already has a row, so any
  "test on myself" passes — the regression is only caught by a genuinely new user.
- **Fix (2 layers):** (1) a scalar subquery — always exactly 1 row:
  `SELECT (SELECT trial_started_at FROM profile WHERE telegram_id=$1) AS trial_started_at;`
  (NULL if there's no profile); (2) `alwaysOutputData: true` on the node. Either one is
  enough, but set both.
- **Process rule:** any change to a payment/onboarding path must be run through a "NEW user"
  scenario (a telegram_id with no row in profile/DB), not just against existing accounts.
  After adding a Postgres SELECT to a webhook chain — immediately ask: "what happens on
  0 rows?".
- **Status:** ⚠️ Gotcha (fix: scalar subquery + alwaysOutputData).

### PG-E10. Cloud DB — an IP whitelist doesn't pick up a server migration on its own

- **Symptom:** a workflow with an external (non-internal-n8n) Postgres node fails with
  `Connection refused` after n8n moves to a new server, even though credentials and host are
  unchanged.
- **Cause:** Beget's "Cloud Databases" (cp.beget.com → Cloud → Cloud Databases → the specific
  DB → "Settings") by default allow connections only from the private network plus an
  explicit whitelist ("External network access"). The whitelist contains the old server's IP
  — on a move to a new IP, nobody adds it there automatically.
- **Fix:** go into that cloud DB's settings → "External network access" → add the new IP
  (the old one can stay for a while in case of rollback, remove it later). Check with
  `nc -zv <host> 5432` from the new server — should say `succeeded`, not `refused`.
- **Process rule:** on any server migration involving n8n workflows — check ALL external
  Postgres/MySQL credentials (not just the internal docker-compose postgres) for IP
  restrictions at the DB provider, not just DNS/domain.
- **Status:** ✅ Confirmed (fix applied and verified).

### PG-E11. A `$` sign in SQL breaks the query — the Postgres node treats it as a placeholder

- **Symptom:** the workflow fails with `Syntax error at line 2 near "..."`, even though the
  query runs fine in psql. The error position points somewhere unrelated to the actual
  problem, which is the most confusing part.
- **Cause:** n8n's Postgres node itself parses `$…` as a placeholder for `queryReplacement`.
  Any `$` in the query text — including a `$` inside a string literal — gets caught by this
  and mangles the SQL. Classic case: a regex for checking a number

```sql
CASE WHEN (e->>'k') ~ '^[0-9]+(\.[0-9]+)?$' THEN (e->>'k')::numeric ELSE 0 END
```

  Here `$` is an end-of-string anchor, but the node sees a placeholder.

  Minimal reproduction (fails without the rest of the query):

```sql
SELECT (CASE WHEN ('150' ~ '^[0-9]+$') THEN 1 ELSE 0 END) AS r;
```

- **Solution:** don't use `$` in SQL at all. For jsonb, a type check instead of a regex is
  both shorter and more reliable:

```sql
CASE WHEN jsonb_typeof(e->'k')='number' THEN (e->>'k')::numeric ELSE 0 END
```

  If a regex is still needed — move the check into a Code node, or avoid `$` anchors.
- **Why this is dangerous:** a query in a scheduled workflow (a weekly report) fails silently
  — the person simply doesn't get the report. Test new SQL against the live database before
  it goes into the schedule.
- **Status:** ✅ Confirmed (isolated with a minimal example).

### PG-E12. A multi-row `VALUES (…),(…)` with placeholders inserts only the first row

- **Symptom:** the n8n Postgres node returns `success: true`, but only one row appears in the
  table instead of two. No error, clean logs.
- **What it looked like:** `INSERT INTO t(a,b,c) VALUES ($1::bigint,'user',$2::text),($3::bigint,'assistant',$4::text);`
  with four parameters in `queryReplacement`. Only the pair `$1/$2` got written.
- **Working form:** `INSERT INTO t(a,b,c) SELECT $1::bigint,'user',$2::text UNION ALL SELECT $1::bigint,'assistant',$3::text;`
  — reusing one placeholder is fine, both rows land correctly, and commas/quotes inside values
  don't shift.
- **Why it matters:** silently losing half the records looks like "memory isn't working", and
  people start debugging the logic instead of the insert. Check with a `SELECT` after the
  node, not the node's status.
- **Related:** always pass parameters as an array `={{ [...] }}` — the string form of
  `queryReplacement` splits on commas (see PG-R5).
- **Status:** ✅ Confirmed — both forms tested against production DB in a rolled-back
  transaction.

### PG-E13. An SQL comment swallowed a comma and broke a daily workflow for three days

- **Symptom:** <workflow_name> (automatic subscription renewal) failed every day at 07:00 for
  three days straight. Error `Failed query` on the `Renewal Due` node. Nobody noticed: the
  workflow is silent, and no charges were due to anyone during those days.
- **Cause:** an edit added an explanatory note at the end of a SELECT-list line — `... AS charge_date
  -- charging happens a day before expiry, so the text shows the attempt date,` — and the
  comma separating columns ended up INSIDE the comment. The following `CASE p.plan ...` was
  left without a separator.
- **Solution:** move the comment to its own line, keep the comma in the code. The fixed SQL
  was run live — success.
- **Rule:** editing a comment in SQL IS editing SQL. A single-line `--` eats everything to the
  end of the line, including syntax punctuation. After any such edit — a live run, not
  "looks fine by eye".
- **How to catch it earlier:** daily workflows with no user-facing feedback need to be checked
  by execution status, not by waiting for a complaint. Three `error`s in a row in the list is
  already a signal.
- **Status:** ✅ Fixed and verified with a live run; no damage — nobody fell into the
  charge/warning window during those days.

### PG-R12. A new profile field — a route through seven places; there are FOUR whitelists, and they're independent

- **Task:** add a new profile field for the user (in the coaching product — anthropometry:
  `limb_ratio`, `fat_pattern`) so both the frontend and the agent itself can write it.
- **Gotcha this entry exists for:** writes to `profile` are guarded by allow-lists, and there
  are **four independent copies** — one at each entry point:
  1. the web client — its own Update SQL build;
  2. the Telegram Mini App — its own Update SQL build;
  3. the tool the agent uses to edit the profile — its own Safe SQL build;
  4. parsing of "profile notes" that the agent appends at the end of its reply — a separate
     node.

  Miss one and the field gets silently rejected **in exactly that environment**: e.g. it saves
  in the browser but not in Telegram; or the user sets it by hand but the agent can't record
  it from conversation. This is caught much later, because "it works for me".
- **Full route (7 places):** column migration → 4 allow-lists → the profile field list on the
  frontend (so the field is visible and editable) → an onboarding control → a mention in the
  prompt of whichever agent needs to take it into account. Check the mock allow-list
  separately: if it's derived from the shared field list it updates itself, if it's a
  separate list it needs to be updated by hand.
- **Run the migration as a one-off workflow and delete it afterward:** `ALTER TABLE … ADD COLUMN IF NOT
  EXISTS` (idempotent, safe to re-run) + a second query `SELECT column_name
  FROM information_schema.columns …` as self-verification in the same response. Call the
  webhook **locally** (`127.0.0.1:5678`) — never expose this kind of thing externally.
- **Targeted edits to a large prompt — replace by anchor, not a full overwrite.** A role
  prompt is tens of thousands of characters; piping the whole thing through a webhook just for
  one section is an unnecessary risk of losing something you didn't see. It works like this:
  `UPDATE prompts SET content = replace(replace(content, $1, $1 || $2), $3, $3 || $4) WHERE name='trainer' AND position('<new section marker>' in content) = 0;`
  The `position(...)=0` condition makes the edit idempotent, long texts go through
  `queryReplacement` (don't embed them in the SQL — escaping), and rollback is covered by
  prompt snapshots under git.
- **Value type:** for fields the prompt reads, store **codes** (`"long"`/`"even"`/`"short"`),
  not free text — inconsistent wording trips up the agent. Show them to the user via a
  `select` field with human-readable labels on the frontend.
- **Empty ≠ default value.** Leave an optional field nullable and explicitly tell the prompt
  "empty — don't guess": otherwise the agent will start fabricating a non-existent answer.

Status: ✅ Confirmed: columns created, all four lists accept the field, the prompt is patched
(the section was added exactly once), the frontend verified in the browser.

### PG-E14. An intermediate status with no watchdog = a record hangs forever

- **Symptom:** a scheduled post didn't go out at its time and never went out at all. In the
  queue, the row sat in status `publishing`; the scheduler runs every 30 minutes, processes
  `success` items, and never touches it. No error in the queue; in the app the post just
  showed as "publishing…".
- **Cause — not a crash, but a gap in the state machine.** The scheduler captures a row
  atomically: `UPDATE … SET status='publishing' WHERE status='scheduled' AND publish_at <= now() RETURNING …`
  (PG-R4). The reverse transition (`published`/`error`) is set by the **last node** of the
  sub-workflow. So any failure before that node — and it failed on the very first meaningful
  node, downloading the cover image — leaves the row stuck in `publishing`. And the capture
  only picks up `scheduled`. The state became terminal by accident: nothing can get it out.
- **Solution — a watchdog in the same scheduler, as a separate dead-end branch off the trigger:**
  ```sql
  UPDATE queue SET status='error', updated_at=now()
  WHERE status='publishing' AND updated_at < now() - interval '20 minutes'
  RETURNING id, header;
  ```
  A dead-end branch, not part of the main line: its rows must not feed back into the capture.
  Publishing takes seconds, so 20 minutes is comfortably more than any normal run.
- **Why `error`, not `scheduled`:** automatically putting it back in the queue is only safe if
  it's certain that nothing went out. The failure could also have happened AFTER sending —
  then a retry would duplicate the post for subscribers. `error` is visible in the UI with a
  retry button: the decision is made by a human.
- **General rule:** every intermediate status (`publishing`, `processing`, `locked`,
  `in_progress`) must have a time-based watchdog. Otherwise the very first failure before the
  final node creates a record nobody will ever pick up — and silently, because every scheduler
  run still reports `success`.
- **How to verify:** after deploying, wait for a normal scheduler tick and confirm the
  watchdog ran (0 rows = `{success:true}`) and didn't disturb the main line.

Status: ✅ Confirmed in production — a queued post published normally (a few seconds,
channels succeeded, the record was removed from the queue). Meanwhile the watchdog runs idle
on every tick without interfering with the capture.

### PG-E15. A DB connection outage takes down all scenarios at once; retries help, but only a little

- **Symptom:** within one minute, several scenarios failed at once across several unrelated
  products. All of them errored on a Postgres node with: `The DNS server returned an error, perhaps the
  server is offline`. The scenarios have nothing in common except that all of them hit a
  shared cloud DB at that minute.
- **What this actually is:** not a query error but a connectivity glitch. The query never ran
  at all — so retrying it is safe. But by default the Postgres node doesn't retry anything and
  fails on the first attempt.
- **How much retries can actually buy you (measured, not from the docs):**
  n8n's `waitBetweenTries` is **hard-capped at 5 seconds** — setting 15,000 ms produced the
  same 5 s (tested against a node designed to fail: 3 attempts × 2 s timeout + 2 pauses = 16 s
  instead of the expected 36 s). The `maxTries` ceiling is 5. Total maximum ≈ **20 seconds**
  of waiting. That won't cover a full minute-long outage — but it fully covers short blips.
- **Where NOT to put a retry:** a bare `INSERT` that creates a new row. The disconnect could
  have happened after the commit but before the response — then a retry inserts a second
  identical row. In one of the products, exactly two nodes were left without a retry: inserting
  a post into the queue (a duplicate = the same post sent to subscribers twice) and writing to
  the publication log (a duplicate = skewed analytics). All other Postgres nodes — `SELECT`,
  `UPDATE`/`DELETE` by key, and `INSERT … ON CONFLICT` — retry freely: they're idempotent.
- **What actually saves the day:** not the retry, but the fact that the schedule keeps moving.
  The affected schedules recovered on the next tick, with no data lost. The retry exists so
  that a person who happened to tap something at that exact second doesn't get an error.
- **Rule:** any network call to the DB — always with a retry, except non-idempotent inserts.
  And treat the retry as a mitigation, not a guarantee: the scenario must remain correct even
  after all retries are exhausted.

Status: ✅ Verified — the vast majority of Postgres nodes in one product (86 of 88) were
switched to 5 retries, both schedules ran a normal tick after deployment, and the app's actions
respond as before.

### PG-E16. `$N::timestamp` with an empty string fails even inside `CASE WHEN $N = ''`

- **Symptom:** `invalid input syntax for type timestamp: ""` — the parameter's type is
  inferred from the first cast at planning time, and `CASE` doesn't save you from that.
- **Rule:** pass optional dates/numbers from a form as a string and cast via
  `NULLIF($N, '')::timestamp` (`::bigint`, `::int`) — the parameter stays text, and NULL is
  cast without an error.
- **Source:** a one-off import of a third-party log.
- **Status:** ✅ Confirmed.

### PG-R13. Cards with a partial unique index: close old ones in a separate node, write new ones in the next

- **Context:** a table of cards with an index "one active per (camera, view)"
  (`UNIQUE ... WHERE status IN ('new','in_work','normal')`). A single run needs to both close
  a stale card and create a new one with the same key.
- **Gotcha:** in one query with a data-modifying CTE (`WITH fade AS (UPDATE …), ins AS
  (INSERT …)`) the execution order isn't guaranteed, and the index is checked against a
  snapshot taken before the changes — the INSERT hits `duplicate key`. Also, results from such
  CTEs are only visible through `RETURNING`: a final `SELECT … FROM <table>` would show the
  rows as they were before the edits.
- **Solution:** two consecutive Postgres nodes: "Fade out" (`UPDATE … RETURNING`) → "Write"
  (`INSERT … RETURNING *` + `UPDATE … RETURNING p.*` combined via `UNION ALL` of the
  RETURNING outputs). Operations are passed as a single `$1::jsonb` parameter and unpacked
  with `jsonb_to_recordset($1) AS o(op text, id bigint, …)` — one query per batch, no
  per-element loops. Flags not present in the table come back via a `JOIN ops` on the insert
  key.

Status: ✅ Confirmed — the feature was tested on a phone (7 cards).

### PG-E17. UPDATE/DELETE affecting no rows — the Postgres node returns `{success: true}`, which looks like a result row

- **Symptom:** a repeated confirmation action on an already-confirmed record replies
  "approved: true", a repeated "revoke" replies "revoked: true", and the handler reading
  `rows[0].hypothesis` crashes with `undefined`. Nothing actually changed in the database.
- **Cause:** for `UPDATE … RETURNING` / `DELETE … RETURNING` that affected zero rows, the
  Postgres node (v2.6, n8n 2.37) emits a single item `{ "success": true }` instead of an empty
  output. For `SELECT` (and for `WITH … SELECT`) with no rows there's no output at all — hence
  neighboring actions of the same API behave differently.
- **Solution:** in the shared response node, filter out pseudo-rows:
  `rows = items.filter(j => j && Object.keys(j).length && !(Object.keys(j).length === 1 && j.success === true))`.
  Then "nothing changed" honestly returns `false` / 404.
- **How it was caught:** an automated test for "repeat action" (a second "revoke", confirming
  a record that isn't pending) — that's exactly what found it.

Status: ⚠️ Error recorded; fix verified by tests (67/67 after the change).

### PG-E18. A final SELECT in the same query as an UPDATE-CTE sees the row BEFORE the update

- **Symptom:** after the "Acknowledged" button, the bot writes `ack_by` to the database but
  edits the message as if nothing was acknowledged: no "Acknowledged: …" line, both buttons
  still there. A second tap already says "Already acknowledged". Same with "Close": the
  keyboard doesn't go away.
- **Cause:** `WITH a AS (UPDATE … RETURNING …) SELECT … FROM events e WHERE …` — data changed
  inside a data-modifying CTE is **not visible** to the rest of the query (Postgres executes
  it against the snapshot taken before the changes). The result: stale `ack_at` / `closed_at`.
- **Solution:** take the fresh values from the CTE's own `RETURNING` and blend them in via
  `COALESCE(a.ack_at, e.ack_at)`; and **always return a row**
  (`FROM (SELECT 1) one LEFT JOIN events e ON …`), otherwise for a non-existent id the
  permission flag disappears too → "no access" instead of "not found".

Status: ⚠️ Error recorded; fix verified by automated tests (29/29).
