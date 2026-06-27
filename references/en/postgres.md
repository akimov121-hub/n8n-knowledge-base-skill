# Postgres

Postgres nodes: `executeQuery`, Postgres Chat Memory.

---

## ✅ Solutions

### PG-R1. Controlling a value's type in SQL — embed `{{ Number(...) }}` directly in the query text

- **Context:** you need to substitute a numeric value (id, message_id) into a query so that it
  reaches the DB as a **number**, not a string.
- **Fact (verified):** cross-node references in `queryReplacement` **are evaluated** —
  `$('Node').first().json.field` resolves correctly. So passing a value via
  `queryReplacement` with a cross-node ref is a valid approach.
- **Caveat:** `queryReplacement` passes the value as parameter `$1`, and the evaluated
  result often goes through as a **string**. If the target column is `bigint`/`numeric` this
  produces an error (see PG-E1). To guarantee the value goes through as a number, two paths:
  1. Wrap it in `Number(...)` directly in the `queryReplacement` expression.
  2. Embed the numeric literal directly in the SQL text:

```sql
VALUES ({{ Number($('Telegram Trigger').first().json.message.message_id) }})
```

Status: ✅ Confirmed.

### PG-R2. Arbitrary SQL / DDL via a temporary webhook workflow

- **Context:** the public n8n REST API cannot execute arbitrary SQL; the Postgres
  Chat Memory nodes create tables lazily (on the first message); you need to create a table
  ahead of time, inspect the schema, run a "dry" test before a `DELETE`, or selectively
  populate a lookup table.
- **Solution:** a one-off workflow `Webhook (POST) → Postgres executeQuery → Respond to
  Webhook (allIncomingItems)`. Create it via `POST /api/v1/workflows`, activate it, hit it
  with curl, read the result from the response, then **deactivate and delete it** (don't leave
  an active public webhook around — a small risk).
- **Use cases:** `CREATE TABLE`, schema verification (`SELECT … FROM information_schema.columns`),
  a dry run of a cleanup (`SELECT count(*)` instead of `DELETE` — to see what would be deleted),
  populating lookup tables with an `INSERT` using dollar-quoting (`$Q$text$Q$`) for texts with
  quotes.
- **Caveat:** the table name can be substituted with an expression directly in the
  `executeQuery` text (`{{ $json.table }}`) — n8n resolves the expression before sending to
  the DB. This is safe as long as the name comes from your own controlled list, not from user
  input.

Status: ✅ Confirmed.

### PG-R3. Inserting into the lowest free `id` (gap-fill, reusing numbers)

- **Context:** the record's `id` is shown to the user as a "number" (bot memory, reminders,
  any personal list). With a plain `serial`, deletions leave gaps and new records go to the
  end. You want numbers to run without gaps and for **the freed-up number to be taken by the
  next new record**.
- **Solution:** don't touch the schema or recreate the PK. Instead of `serial`, insert an
  **explicit `id` = the lowest free one** via `generate_series` + an anti-join. The real PK
  then equals the displayed number — `delete`/`update` address it directly.

```sql
INSERT INTO <table> (id, ...) VALUES (
  (SELECT COALESCE(MIN(s.n),1)
     FROM generate_series(1, COALESCE((SELECT MAX(id) FROM <table>),0)+1) s(n)
     LEFT JOIN <table> m ON m.id = s.n
    WHERE m.id IS NULL),
  ...
) RETURNING id;
```

- **Behavior:** empty → id 1; have {2,3,4} → a new one takes 1, then 5; deleted a middle one →
  the next new one takes it. Existing numbers don't change.
- **Caveats:** the scalar subquery in `VALUES` sees the table BEFORE the insert — correct.
  Since we always insert an explicit `id`, the `serial` counter isn't used → no collisions
  (it's important that ALL insert paths go through this pattern). For multi-user, add
  `WHERE telegram_id=...` to both parts of the subquery.
- **Related:** editing a record "in place" is a separate `UPDATE` by id, not delete+insert
  (otherwise the number moves to the end).

Status: ✅ Confirmed in production.

### PG-R4. Idempotency of an action via atomic status capture (protection against a double button press)

- **Context:** an inline button that triggers an irreversible action (publishing, charging,
  sending), when pressed twice, spawns **two parallel runs**. Both read the "ready to act"
  status before the first one changes it → the action runs twice (a double post, etc.).
- **Solution:** not "SELECT status → IF", but an **atomic UPDATE with a condition** on the
  status row:

```sql
UPDATE drafts SET stage='processing', updated_at=now()
WHERE chat_id=<chat_id> AND stage='await_review'
RETURNING <needed columns>;
```

  Postgres serializes concurrent UPDATEs on a row: the first flips
  `await_review→processing` and returns the row, the second no longer finds `await_review` →
  0 rows. **Important (see PG-E2):** on 0 rows the node returns NOT 0 items, but a service item
  `{success:true}` — so the protection only works in combination with an IF on the status/key
  field (`stage='processing'` / `chat_id notEmpty`) right after the UPDATE. No timers/debounces —
  the guarantee is at the DB level + a mandatory IF.
- **Caveat:** if the action can partially fail after the capture — the status stays
  `processing`; provide a rollback or drive it to a final status in every branch
  (onError continue).

Status: ✅ Confirmed.

### PG-R5. Text parameters via queryReplacement as an array-expression

- **Pattern:** pass user text into SQL NOT by concatenation with `'`→`''` escaping,
  but as parameters `$1..$n`: in the node options `queryReplacement` = `={{ [expression1, expression2, ...] }}`
  (a single expression returning an array).
- **Why an array:** the comma-separated string format of queryReplacement breaks on commas
  inside a value; an array-expression carries commas, quotes, HTML, and line breaks without
  loss.
- **Verified live:** `SELECT $1::text...` + `={{ [$json.body.a, ...] }}` — values with
  commas/quotes/tags came back 1:1.
- **Limitation:** values go through as strings — for `bigint`/`numeric` columns still
  embed `{{ Number(...) }}` as a literal in the SQL (PG-R1, PG-E1).
- **Use case:** the publisher product — all writes of post text/instructions/caption.

Status: ✅ Confirmed.

### PG-R6. Webhook response as a list/object — `json_agg`/scalar-subquery, guaranteed 1 row

- **Context:** in the Mini App API (webhook + Respond), actions like `list`/`get` read from
  the DB. If a SELECT returns 0 rows — the branch may not continue, and the `Respond to Webhook`
  node won't fire → **the request hangs** on the frontend (see also the ambiguity of PG-E2:
  0 rows → sometimes `{success:true}`, sometimes empty).
- **Solution:** wrap the query so the node **always returns exactly 1 row**:
  - list → `SELECT COALESCE(json_agg(x), '[]'::json) AS __list FROM (SELECT ... ) x;` —
    empty → `[]`, otherwise an array.
  - object → `SELECT COALESCE((SELECT to_jsonb(t) FROM (SELECT ...) t), '{}'::jsonb) AS __obj;` —
    no record → `{}`.

  Then a Code node normalizes (`if(typeof v==='string') JSON.parse`), and Respond returns the
  array/object. Bonus: `bigint` inside `json_agg`/`to_jsonb` is serialized as a **number** (not
  a string) — this fixes epoch-ms times for the frontend.

Status: ✅ Confirmed.

---

## ⚠️ Errors

### PG-E1. `invalid input syntax for type bigint`

- **Symptom:** Postgres executeQuery fails with `invalid input syntax for type bigint`.
- **Cause:** `queryReplacement` passes the evaluated value as parameter `$1`
  **as a string**; if the column is `bigint`/`numeric` and the string is empty or non-numeric,
  Postgres won't coerce it and fails. (The cause is precisely the value's type, not that the
  cross-node expression "isn't evaluated" — it is evaluated, see PG-R1.)
- **Fix:** wrap the value in `Number(...)` in `queryReplacement`, or embed the numeric
  literal directly in the SQL via `{{ Number(...) }}`. Also ensure the value is non-empty.

### PG-E2. executeQuery on 0 rows returns an item {success:true}, not 0 items

- **Symptom:** downstream nodes after a Postgres node execute even though the query
  (UPDATE...RETURNING / SELECT) returned no rows; the item contains `{"success": true}`
  without the query's fields. If an Execute Workflow node sits after it, the sub-workflow is
  called with this junk item (`image_url=undefined`, etc.).
- **Cause:** the n8n Postgres executeQuery, on an empty result, emits a service item
  `{success:true}` — you can't rely on "0 rows → the branch won't execute".
- **Fix:** after every query that "may return 0 rows" — an IF-guard on a key field
  (`{{ $json.chat_id }}` notEmpty or a status check), and only then the action.
- **Status:** ⚠️ Confirmed in practice.

### PG-E3. Dollar-quoting `$tag$…$tag$` with a numeric value breaks n8n's parameter scanner

- **Symptom:** the Postgres node fails with `Variable $<large number> exceeds supported maximum of $100000`,
  even though there's no such parameter number in the query. Example: inserting a Telegram bot token
  `<BOT_TOKEN>` via `VALUES ('bot_token', $tok$<BOT_TOKEN>$tok$)`.
- **Cause:** n8n scans the SQL for `$N` placeholders (queryReplacement) **before** sending to
  Postgres and doesn't understand dollar-quoting. The closing `$` of the tag + the value's
  digits (`…$tok$<token digits>`) are read as `$<number>` → a "parameter number" out of range.
- **Fix:** **never** insert values with dollar-quoting — pass them as a native parameter
  `$1` via `options.queryReplacement` (see PG-R5). This is especially critical for values
  starting with digits (ids, tokens).
- **Status:** ⚠️ Confirmed in practice.

### PG-E4. Multi-statement SQL as an n8n expression (with `{{ }}`) → "invalid syntax"

- **Symptom:** a Postgres node with a multi-statement query (several `;`), where values are
  embedded via `{{ … }}` (i.e. the entire query text became an n8n expression with a leading `=`),
  fails with `invalid syntax` before even reaching Postgres. A single statement with the same
  `{{ }}` works fine.
- **Cause:** when the entire SQL is one n8n expression, the expression parser trips on the
  complex multi-statement text (especially with regex/special characters inside `{{ }}`). This is
  a bug in the n8n expression engine, not Postgres.
- **Fix:** **one statement — one node** + values as native parameters `$1`
  (queryReplacement); leave `{{ }}` only in separate fields (params / chatId / text of the
  Telegram node), but NOT in the SQL text. Decompose a multi-step operation (UPSERT + flag + log +
  query) into a chain of nodes. For regexes/sanitizing — a separate Code node (Prep), and the SQL
  references the ready, clean values.
- **Status:** ✅ Confirmed (after splitting into single statements everything worked;
  the grant/revoke/add/decline smoke cycle is green).

### PG-E5. executeQuery with multi-statement SQL returns the result of the FIRST statement, not the last

- **Symptom:** a Postgres node has one query made of several statements
  (`INSERT…; UPDATE…; SELECT…;`). All statements **execute** (the writes go through), but the
  node's output item gets the result of the **first** statement. A node downstream that reads
  `$('This node').first().json.<field from the last SELECT>` gets `undefined`. Example: a
  grant node (3 writes + a final `SELECT bot_name`), and the message renders as «to the bot
  "undefined"».
- **Test (read-only):** `SELECT 'a' AS first_stmt; SELECT 'b' AS second;` → the node returned
  `{first_stmt:'a'}`.
- **Fix:** fetch the value needed downstream with a **separate node** (a single
  SELECT) — its result is correct. Leave multi-statement writes as is if their result isn't
  read. In the Mini App API this is already split across nodes (Grant SQL separate from
  Grant BotName).
- **Related:** reinforces PG-R6 (one node = one needed result) and PG-E4.
- **Status:** ✅ Confirmed (read-only test + splitting across nodes).

---

## ✅ Solutions (continued)

### PG-R7. "One row per user" draft — on starting a new item, reset carried-over fields

- **Context:** wizard state (a post draft) is stored as a single `pub_drafts` row per
  `user_id` (UPSERT `ON CONFLICT (user_id)`). The same row is reused for the next post.
- **Pitfall:** when generating the text of a NEW post, the UPSERT updated
  `mode/idea/header/post/stage` but **didn't touch the derived fields of the previous post**
  (`image_url`, `ig_caption`). Result: the new post inherited the old cover/caption; the
  frontend (polling `draft/get`) picked up the old image and offered "Regenerate" instead of
  creating a new one.
- **Solution:** in `ON CONFLICT DO UPDATE` for a new item, **explicitly reset all
  carried-over/derived fields**:
  `... SET mode=$1, idea=$2, header=$3, post=$4, stage='text', image_url=NULL, ig_caption=NULL, updated_at=now()`.
  The general rule: when reusing a state row, a new "lifecycle" must reset everything that
  belonged to the previous one, not just the fields being overwritten.
- **Client-side protection (extra):** also clear the mirror on the frontend at start
  (`S.draft.imageUrl=''; igCaption=''`), otherwise a persistent polling watcher will re-pick
  up the stale data.

Status: ✅ Confirmed.

### PG-R8. Free JSONB merge (`data || EXCLUDED.data`) — new frontend metrics without schema migrations

- **Context:** the frontend (the "Diary" Mini App) writes daily metrics; the set of keys grows
  from version to version (added `stress`, `anxiety`, `focus`, `sleep_quality`,
  `meditation_min`, a `reflection` object, `nutrition` became a number). Storing each key as a
  separate column → a migration for every new metric.
- **Solution:** keep the metrics in a single JSONB column `data` and do a **partial merge** in
  the UPSERT: `INSERT … ON CONFLICT (telegram_id, day) DO UPDATE SET data = daily_checkin.data || EXCLUDED.data`.
  The `||` operator for jsonb is a shallow merge: new/changed keys are overwritten, old ones
  are preserved. `checkin/history` returns the entire `data` as-is → new keys come back to the
  frontend without backend changes.
- **What it gives:** adding a metric on the frontend does NOT require a DB migration or a
  workflow change — only testing. Key-name validation via a light regex (`^[a-z][a-z0-9_]{0,40}$`),
  without a strict whitelist. Values of any type (number/scalar/nested object) pass through.
- **Boundaries:** `||` merges only the top level — a nested object (`reflection`) is
  overwritten entirely, not merged key-by-key. If you need a deep merge — `jsonb_set` or a
  `jsonb_deep_merge` function. The downside of this approach is the lack of a schema at the DB
  level (types/required keys aren't guaranteed), so keep critical/indexable fields as separate
  columns, and put only the "extensible tail" of metrics in JSONB.

Status: ✅ Confirmed.

### PG-R9. Idempotency of scheduling: a stage filter when inserting from a reusable draft row

- **Symptom:** the same post appeared in the queue twice (two `publisher_queue` rows, identical
  content and `publish_at`).
- **Cause:** the `publish/schedule` action inserted a row by reading the draft
  `... FROM pub_drafts WHERE post<>'' AND image_url<>''` — **without checking the stage**. After
  the first scheduling the stage → `queued`, but `post`/`image_url` remain → a repeat call
  (a double tap on the time button / a retry / a webhook redelivery) passes the filter again and
  inserts a duplicate. One call = one row (`INSERT ... SELECT FROM src`), so a duplicate = two
  calls.
- **Solution:** add an active-stage filter to `src` — insert only from a NON-finished
  draft: `... AND stage IN ('idea','text','image','preview')`. After the first scheduling
  (`stage='queued'`) a repeat call finds 0 rows in `src` → `INSERT ... SELECT` inserts nothing
  → idempotent. The general rule: **any "one-time" insert from a state row that keeps living
  must be gated by a stage/"already processed" flag**, otherwise a repeated request breeds
  duplicates.
- **Extra client-side protection:** an in-flight flag against a double tap — kills the
  duplicate before the backend even. Full protection against the race is also provided by a
  unique index, but the stage filter covers the real case (a sequential double tap).

Status: ✅ Confirmed.
