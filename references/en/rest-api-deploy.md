# REST API deployment

Deploying and editing workflows through the REST Public API of a self-hosted n8n.

> Placeholders: `<N8N_BASE_URL>` — your instance domain; `<workflow_id>` — workflow id;
> `<ERROR_WORKFLOW_ID>` — id of the shared Error Workflow; `<*_CRED_ID>` — credential ids.

---

## ✅ Solutions

### DEP-R1. Two kinds of JWT tokens — do not mix them up

A server may have two different tokens for two different APIs. Mix them up and you get 401 or 404.

| Type | Audience | Header | Endpoint | Used for |
|---|---|---|---|---|
| MCP Server API | `mcp-server-api` | `Authorization: Bearer <token>` | `/mcp-server/http` | SDK / MCP operations |
| REST Public API | `public-api` | `X-N8N-API-KEY: <token>` | `/api/v1/...` | GET/PUT workflows, executions |

Status: ✅ Confirmed in production.

### DEP-R2. Deployment order via PUT without losing credentials

**Context:** `PUT /api/v1/workflows/{id}` fully replaces all nodes. GET **does not return**
the `credentials` field (stored separately in the DB). If you don't add credentials manually before PUT,
all bindings will be lost.

**Mandatory order:**
1. `GET /api/v1/workflows/{id}` — get the current JSON.
2. Surgical edit in Python, **preserve all node IDs** from the source.
3. Add `credentials` to each node by type (credential map below).
4. `PUT` with minimal settings (see DEP-R3).
5. `GET /api/v1/executions?workflowId={id}&limit=3` — verify there are no credential errors.

**Credential map template** (substitute your own ids and names for the node types you use):

```python
telegram_types = ["telegramTrigger", "telegram"]
postgres_types = ["postgres", "memoryPostgresChat"]
openai_types   = ["lmChatOpenAi", "n8n-nodes-base.openAi", "@n8n/n8n-nodes-langchain.openAi"]

CREDS = {
    "telegram":  {"telegramApi": {"id": "<TELEGRAM_CRED_ID>", "name": "<credential name>"}},
    "postgres":  {"postgres":    {"id": "<POSTGRES_CRED_ID>", "name": "<credential name>"}},
    "openai":    {"openAiApi":   {"id": "<OPENAI_CRED_ID>",   "name": "<credential name>"}},
}
# If different workflows use different bots/DBs — keep a separate entry per
# credential and substitute the right one based on the workflow's context.
```

To find the real credential ids: `GET /api/v1/credentials` (or from the UI). The credential type
(`telegramApi`, `postgres`, `openAiApi`) must match what the node expects.

Status: ✅ Confirmed in production.

### DEP-R3. Minimal settings for PUT

On PUT, strip out non-standard settings fields (`availableInMCP`, `binaryMode`, `timeSavedMode`,
etc.) — otherwise validation complains (see DEP-E7). Pass only the basic set:

```json
{
  "executionOrder": "v1",
  "callerPolicy": "workflowsFromSameOwner",
  "errorWorkflow": "<ERROR_WORKFLOW_ID>"
}
```

The server preserves the existing additional settings on its own — after PUT, a GET will again show
`binaryMode` and the like. Losing them is not a concern.

Status: ✅ Confirmed.

### DEP-R4. The new workflow's ID — take it ONLY from the server response

On `POST /api/v1/workflows` (creation) the server returns the real `id`. That, and only that,
should be used in all subsequent calls (activate / PUT / executions / delete).

**Rule:** don't guess, don't "remember", don't substitute an id that merely looks similar. Right after POST,
extract the `id` from the JSON response into a variable. After creating/editing — cross-check against the list by
**name**: `GET /api/v1/workflows?limit=200` → find your `name`.

Related to DEP-E5. Status: ✅ Rule.

### DEP-R6. Rotating a service secret — read from service_tokens at runtime, do not self-update the credential

- **Context:** a long-lived external-service token (OAuth/Bearer, Meta/IG, etc.)
  is periodically renewed by a workflow. The temptation is — after renewal, write it into the n8n credential
  via `PATCH /api/v1/credentials/{id}` (DEP-R5), so that HTTP nodes pull from the credential.
  In practice this PATCH fails in production (`invalid syntax`, see DEP-R5) — the Public API
  is not designed for writing credential secrets.
- **Solution (reliable):** **do not use a credential for the rotated token at all.**
  The source of truth is the mirror table `service_tokens(service, token, expires_at)`.
  1. Renewal workflow: `refresh → UPDATE service_tokens` (and that's all, with no credential-update
     node and no alert about its failure).
  2. Consumer workflow: at the start of the branch — a Postgres node
     `SELECT token FROM service_tokens WHERE service='<svc>'`; HTTP nodes are switched from
     `authentication: genericCredentialType` to `none` + a manual `Authorization` header
     = `={{ "Bearer " + $('<token-node>').first().json.token }}`. Nodes that read input by
     name (`$('<input-node>')...`) are not broken by inserting the token node into the line — data
     is pulled by name, not from `$json`.
- **Why it's better:** no fragile PATCH, no manual UI updates, no false alerts;
  rotation = just a DB write, the consumer always takes the fresh value.
- **Deployment:** change the active sub-workflow via REST PUT (DEP-R2/R3), having taken a JSON backup;
  the token node = a clone of an existing Postgres node (same credential). Verification without publishing:
  hit a read endpoint of the service with the token from `service_tokens` (200 = token is valid); then
  one real run.
- **Status:** ✅ Confirmed in practice (a real publish went through).

### DEP-R7. Deploying scenarios via a script through the Public API: push by name, auto-binding the Error Workflow, tags, activation, GET cross-check

- **Why:** a stream of small scenarios on a contour without a UI — each one must come out with an error
  handler and a tag, and nobody should have to remember this by hand.
- **Deploy script** (stdlib, the Public API key is pulled from a protected secrets store):
  the "deploy workflow file" command substitutes the notification recipient in place of the placeholder
  in the JSON → looks up the workflow by name (`GET /workflows?limit=250`) → `PUT` or `POST` with only
  `{name, nodes, connections, settings}` and the settings whitelist (see DEP-E7) →
  `settings.errorWorkflow` = the id of the shared handler found by name (no handler found —
  stop, except when deploying the handler itself) → tags via `POST /tags` + `PUT /workflows/{id}/tags
  [{id}]` → `POST /activate` → **GET cross-check** (name, node count, errorWorkflow, active). Before
  every PUT — a snapshot of the current version into a backup folder (in `.gitignore`). A separate command —
  bind the shared error handler to already-deployed scenarios, preserving nodes and activation state
  (cross-check afterward).
- **Credentials:** the Public API does not list them; the workflow JSON only carries `credentials: {type: {id,
  name}}`. Get the id at creation time (`POST /credentials` returns the id) or read-only from the instance
  DB: `select id, name, type from credentials_entity`.
- **Updating an active workflow via PUT** re-registers triggers on its own (Schedule/Webhook
  kept working without a separate deactivate/activate).
- **Addendum — folders.** The n8n 2.37 Public API neither returns nor accepts a scenario's folder:
  `GET /workflows/{id}` has no `parentFolderId` or similar field, even though the instance's internal DB
  does have the `workflow_entity."parentFolderId"` column (folders live in the `folder` table). Read the
  folder only from the DB, read-only: `select w.id, coalesce(f.name,'') from workflow_entity w left
  join folder f on f.id = w."parentFolderId"`. The deploy script checks the folder before and after PUT
  and warns if the scenario dropped out of its folder. A new scenario (POST) lands in the root.
  ✅ **PUT via the Public API preserves the folder** (a body without `parentFolderId` doesn't touch the
  column) — verified with a no-op deploy of a scenario from an existing folder.
- **Status:** ✅ Confirmed (several scenarios deployed via the script, a receiving integration
  rebound to the shared error handler without losses).

---

## ⚠️ Errors

### DEP-E1. `Node does not have any credentials set` on the first node after PUT

- **Symptom:** immediately after PUT the workflow fails on the very first node.
- **Cause:** PUT overwrote the nodes without the `credentials` field (GET doesn't return it).
- **Fix:** redo the PUT, adding credentials to each node per the credential map (DEP-R2).

### DEP-E2. SDK `update_workflow` via MCP breaks the workflow on topological changes

- **Symptom:** after editing via the MCP `update_workflow`, credentials are lost and/or node
  behavior changes.
- **Cause:** it regenerates node IDs → loses the credential binding; it auto-bumps
  node typeVersion (Set 3.2→3.4, Merge 3.1→3.2) → changes behavior.
- **How to avoid:** for any topological changes, deploy via REST PUT (DEP-R2),
  not via the SDK `update_workflow`.

### DEP-E3. A request to n8n from an isolated browser context → 401

- **Symptom:** `fetch()` to the n8n API from a browser extension returns 401, even though in the
  browser itself the session is active.
- **Cause:** the extension runs in an isolated context and does not share the n8n session.
- **How to avoid:** don't use browser `fetch()` for n8n API operations — only the
  REST API with a token.

### DEP-E4. n8n silently drops unknown parameters keys

- **Symptom:** a node parameter was passed via the API, but the behavior didn't change.
- **Cause:** on save, n8n silently discards unknown/misnamed `parameters` keys,
  without raising an error. A common trap is camelCase instead of snake_case
  (`callbackData` instead of `callback_data`), or a wrong parameter name.
- **How to avoid:** after PUT, always do a **GET cross-check** — verify that the parameter
  actually persisted in the JSON. Check the exact names (see the examples in telegram.md: `file` instead
  of `photo`, parse_mode in snake_case).

### DEP-E5. A cascade of 404s "You do not have permission to ... this workflow"

- **Symptom:** after creating a workflow, all activate/PUT/delete/GET calls return
  `HTTP 404` with text about permissions ("Ask the owner to share it with you") — it looks like
  an access problem.
- **Cause (the real one):** the request is going to a **non-existent id** — the id was made up or
  mixed up, rather than taken from the POST response. n8n responds to someone else's/non-existent id with a 404 worded as
  a permissions issue, which is confusing.
- **Fix:** take the real id from the POST response (or from `GET /workflows?limit=200` by
  `name`). See DEP-R4.

### DEP-E6. After PUT the workflow stays deactivated; an immediate `/activate` returns 400

- **Symptom:** a `PUT` on an active workflow publishes a new version, and `deactivated` appears in publishHistory.
  Immediately after, `POST /activate` sometimes returns **HTTP 400** and does NOT
  activate — the workflow stays off (the bot "goes silent").
- **Cause:** a race — PUT itself triggers deactivation/activation, and the parallel `/activate`
  arrives at a bad moment.
- **Fix:** after PUT, **don't trust a single `/activate`** — do a GET, check `active`,
  and if `false` — retry `/activate` with a pause (a retry loop until `active:true`).
- **Prevention:** in the deploy script — a mandatory post-check of activity with a retry (2–3
  attempts, ~2 sec pause), separate from the PUT.

### DEP-E7. PUT rejects "extra" settings keys (400), but the live workflow keeps them

- **Symptom:** `PUT` → **HTTP 400** `"request/body/settings must NOT have additional properties"`.
- **Cause:** the Public API schema accepts a limited set of `settings` (`executionOrder`,
  `errorWorkflow`, `callerPolicy`, `saveManualExecutions`, `timezone`,
  `saveExecutionProgress`, …). Keys set by the UI (`binaryMode`, `timeSavedMode`,
  `availableInMCP`) are returned by GET, but PUT fails with them present.
- **Fix:** in PUT, pass only the minimal settings (DEP-R3). The server preserves
  the existing additional settings on its own.

### DEP-E8. PUT of the main workflow fails with 400 until the sub-workflow is "published"

- **Symptom:** `PUT /api/v1/workflows/{id}` with an Execute Workflow node returns
  `400: Node "..." references workflow ... which is not published. Please publish all
  referenced sub-workflows first.`
- **Cause:** n8n requires that a sub-workflow referenced by an active workflow be
  published (activate) before the referencing one is deployed.
- **Fix:** deployment order for the pair: POST the sub → `POST /workflows/{subId}/activate` →
  PUT the main one. A sub-workflow with an Execute Workflow Trigger activates without webhooks — that's
  safe.
- **Status:** ⚠️ Confirmed in practice.

### DEP-E9. The generator has drifted from production: deploying wipes out manual edits made in n8n

- **Symptom:** before rolling out a change, the workflow generator's output was diffed against the live
  workflow, revealing that the deploy would have destroyed working things: several nodes entirely (the
  branch that receives content into a separate inbox and the branch that handles the menu command),
  the routing rule for that command, the credential binding for third-party image hosting (imgBB — present
  in production, missing in the generator, so images would have stopped loading), and an already-fixed
  parser for a reasoning model's response in one of the nodes (fixed in production, but the generator still
  had the old version — the old bug would have come back).
- **Cause:** edits were made directly in n8n (fast, and often justified), but were never carried back
  into the generator. Meanwhile the generator keeps being treated as the "source of truth" — and a month
  later silently stops being one. An extra tell: the generator's built-in self-check was failing on a stale
  credential check — a sure sign that the generator hadn't been run in a while and the drift had been
  accumulating.
- **Solution:** before every `PUT` from the generator — a **machine cross-check against the live
  workflow**: nodes present only in live, nodes present only in gen, a `parameters` and `credentials`
  diff for every shared node, a `connections` diff. Everything that exists only in live gets carried into
  the generator BEFORE deploying. Every discrepancy gets explained by name: "this is my edit" / "this is
  an n8n default" / "this is a loss."
- **A separate gotcha:** the diff also catches losses in the other direction. The generator was setting
  `maxTokens` on every AI node, but in production the reasoning nodes had no limit. For reasoning models,
  the reasoning itself consumes the same budget, so a hard `maxTokens` on a generative node would have
  truncated the actual answer — an empty result instead of content. The limit is now set only on
  non-reasoning models.
- **General rule:** "generate → deploy" is safe exactly until the first manual edit in the
  UI. After that, the only protection is a diff before rollout; a backup saves you after an incident,
  a diff saves you before one.
- **Cross-check script:** compare a fresh production GET against the generated JSON over the sets of
  node names, `parameters`, `credentials`, and `connections`; filter out noise from n8n defaults
  (`temperature: 1`, `method: GET`, `outputPropertyName`, `sendQuery: false`, leading newlines in
  `jsCode`) by eye — it's harmless.
- **Status:** ✅ Confirmed — the discrepancies were carried into the generators, the workflows were
  deployed, production is working.

### DEP-E10. n8n 2.x only writes files under ~/.n8n-files, the Execute Command node is gone

- **Task:** move generated files (e.g., cover images) off third-party hosting onto our own
  server — n8n needs to write the file into the web root, with the web server serving it as ordinary
  static content.
- **Gotcha 1 — the n8n container has nowhere to write.** Neither the main container nor the worker
  has a mount into the host's web root. Add it as a single line to the shared docker-compose anchor
  (both services inherit it), then `docker compose up -d` recreates the containers — took about
  30 seconds, webhooks and bots survived without loss.
- **Gotcha 2 — "The file … is not writable" on a fully writable directory.** Writes
  failed both to the mounted directory and even to `/tmp`, even though `docker exec … touch`
  worked from both containers. Cause: in n8n 2.x, `SecurityConfig.restrictFileAccessTo`
  defaults to **`~/.n8n-files`** (`@n8n/config/dist/configs/security.config.js`), meaning
  file operations are only allowed there. The right fix is to mount the target directory
  exactly there: `- <web-root>/<folder>:/home/node/.n8n-files/<folder>`. That way there's no need
  to widen `N8N_RESTRICT_FILE_ACCESS_TO` and relax the setting for the whole instance.
- **Gotcha 3 — the Execute Command node isn't in the build.** Activating the workflow fails
  with `Unrecognized node type: n8n-nodes-base.executeCommand` (even though `NODES_EXCLUDE` is empty).
  Anything that needs a shell — file cleanup, `find`, `rm` — has to be moved outside n8n.
- **How this reshapes cleanup of old files:** the actual `rm` is done by a cron script
  on the host, while n8n hands out the "what must not be deleted" list via a secret webhook. This way
  production DB credentials stay inside n8n, the SQL ends up in version snapshots, and the host holds
  no passwords at all. The script exits silently if the webhook didn't answer or answered with a
  refusal: an empty list would otherwise mean "delete everything."
- **Writing the file:** the `n8n-nodes-base.readWriteFile` node (v1), `operation: 'write'`,
  `fileName` — an absolute path, `dataPropertyName: 'data'`. The input `json` passes straight through
  (`Object.assign(newItem.json, item.json)`), so it's enough to compute the filename and public URL
  in the preceding Code node — the next node will pick them up from `$json`.
- **Status:** ✅ Confirmed in production. Verified with both branches for receiving the file (from
  base64 and from binary) — the file gets written, the web server serves it byte-for-byte; the cron
  cleanup ran as expected; publishing picked up the file from our own server and succeeded.

### DEP-E11. Public API: a tag name longer than 24 characters → `409 Tag already exists`

- **Symptom:** `POST /api/v1/tags` with a name longer than 24 characters (e.g., "A corporate
  client — Module" — the name looks short at a glance, but the limit is already exceeded) responds with
  `409 Tag already exists`, even though no such tag exists in `GET /tags` or in the tags table. The deploy
  script, trusting the response code, fails.
- **Rule:** a tag name must be no longer than 24 characters (an n8n limit; the error message is
  misleading). When checking whether a tag exists, read the list with `?limit=250` so a truncated
  list isn't mistaken for the tag's absence.
- **Status:** ✅ Confirmed.

### DEP-E12. `n8n import:workflow` via the CLI requires an `id` field in the JSON

- **Symptom:** `docker exec n8n n8n import:workflow --input=wf.json` → `null value in column
  "id" of relation "workflow_entity"`. The Public API generates the id itself; the CLI doesn't.
- **Rule:** when importing via the CLI, set `"id"` in the JSON (Latin letters/digits, up to
  16 characters); `import:credentials` via the same path accepts `data` in plaintext and encrypts it
  with the instance key. If a Public API key is available, deploy through it instead (see DEP-R7);
  keep the CLI for instances without a key.
- **Status:** ✅ Confirmed.

### DEP-E13. A parent with Execute Workflow won't save until the sub-scenario is published

- **Symptom:** `PUT /workflows/{id}` on the parent → 400 `Cannot publish workflow: Node "…"
  references workflow X which is not published. Please publish all referenced sub-workflows
  first`. A sub-scenario with a single Execute Workflow Trigger, meanwhile, **can** be activated
  (on n8n 2.37, `POST /workflows/{id}/activate` succeeds).
- **Rule:** deployment order for the pair — first the sub-scenario and its activation, then
  the parent. A separate "don't activate" flag for sub-scenarios is no longer needed.
- **Status:** ✅ Confirmed.

---

## 🟡 Fragile / not recommended

### DEP-R5. PATCH /api/v1/credentials/{id} — credential self-update (unstable in production)

- **Pattern:** n8n has `PATCH /api/v1/credentials/{id}` with a body
  `{"data": {...full credential data...}}` — in theory this lets a workflow
  self-update secrets (token rotation) without manually logging into the UI.
- **Caveats:** GET does not return the credential data — the current value must be kept in a mirror
  (the `service_tokens` table); for httpHeaderAuth, the data must pass `name`, `value`, and
  `allowedHttpRequestDomains` (`"all"` or `"domains"`+`allowedDomains` — the create schema
  requires consistency); a PATCH with an empty `{}` is harmless (200, doesn't touch the data).
- **Security:** keep the Public API key for such calls in a separate credential
  (httpHeaderAuth `X-N8N-API-KEY`), restricted to the instance domain.
- **⚠️ Production experience — the pattern is unreliable, replaced.** In a manual test PATCH passed, but the very first
  scheduled auto-run failed: the credential-update node went to the error output with
  `error: "invalid syntax"` (reproducibility in production was not confirmed; you can't tune a working
  body against a live secret). **The correct solution is to NOT self-update the credential, but to
  read the secret from `service_tokens` at runtime: see DEP-R6.** This entry is kept as a
  "known-fragile" path.
- **Status:** 🟡 Passes in a manual test, but unstable in production → superseded by DEP-R6.
