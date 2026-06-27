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
