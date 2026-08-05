# HTTP requests to external APIs

Calling third-party HTTP APIs from n8n directly (not through a ready-made integration node): the HTTP Request node, multipart/form-data, working with binary data, Code node limitations (no authenticated helpers, no Web API), timeouts on long synchronous responses, and the asynchronous polling pattern.

---

## ✅ Solutions

### HTTP-R1 — ~~Comma-separated `inputDataFieldName` for N files~~ REFUTED
**Status:** ⚠️ Refuted (the actual solution is HTTP-R2)
**Symptom / context:** need to send a multipart request to an external API with several files under one field name (`image[]`), the number of files is not known in advance (1–4).
**What was believed to be the solution:** `parameterType: formBinaryData`, `inputDataFieldName: "={{ $json.refKeys }}"` with a comma-separated list of names ("n8n splits the comma-separated list itself").
**Why this is wrong:** n8n does NOT split a comma-separated list. In the source (`HttpRequestV3.node.ts`, checked on 2.26.8 and master), `assertBinaryData(itemIndex, cur.inputDataFieldName)` receives the string as **a single field name** → with two or more references the node fails with `The item has no binary field 'ref0,ref1'`. The original entry had apparently only been confirmed with single-reference runs (`refKeys="ref0"` — no comma), where the scheme degenerates into an ordinary single field.
**Source:** the image-generation service; incident (the first real call with 2 references).

### HTTP-R2 — Sending N files (dynamic count) in multipart: Switch + K copies of the HTTP node
**Status:** ✅ Confirmed (multi-ref and text mode verified by live runs; the result assembled by this scheme was approved by the client)
**Symptom / context:** same as HTTP-R1: multipart with 1–4 files under one field name `image[]`, dynamic file count. Platform limitations (verified against n8n 2.26.8 source):
- comma-separated values in `inputDataFieldName` are not split (see HTTP-R1);
- `formBinaryData` with an empty/nonexistent field name at `typeVersion ≥ 4.2` crashes the node with `Cannot read properties of undefined (reading 'value')` (`prepareRequestBody`: `entry[key].value` when the parameter is missing) — the "empty name = skip the parameter" trick does not work;
- assembling a multipart Buffer by hand and passing it as raw binary is not possible — see HTTP-E2 (`source.on is not a function`): the core's legacy adapter rebuilds the body via object spread when the `multipart/form-data` header is present.
**Solution (works):**
1. **Code node** decodes the files into binary properties `ref0…refN-1` (`prepareBinaryData`) and outputs `refCount` in the json.
2. **Switch node** branches on `{{ $json.refCount }}` (1/2/3/4) into…
3. …**K copies of the HTTP Request node** (`contentType: multipart-form-data`): in the copy for K files there are exactly K static `formBinaryData` parameters with `name: "image[]"` and `inputDataFieldName: ref0…refK-1`. Duplicate `image[]` names are allowed — the FormData path does a separate `append` for each parameter. All copies converge into a single result-decoder node.
4. Authorization in each copy: `authentication: predefinedCredentialType`, `nodeCredentialType: openAiApi` + explicit `credentials` on PUT (see rest-api-deploy.md).
**Example:** the image-generation service, reference branch: Switch "How many references?" → `OpenAI edits 1/2/3/4 refs` → "PNG from base64".
**Side note:** gpt-image `/edits` returns `b64_json` → a decoder Code node runs before Respond (as before).
**Source:** fixing the image-generation service (incident: the first call with 2 references broke the workflow).

### HTTP-R3 — Asynchronous webhook (job_id + polling) for operations longer than 30 s
**Status:** ✅ Confirmed (`1024x1536`+`high` → a 1.2 MB PNG in 90 s from a client network where synchronous mode failed 100% of the time)
**Task:** deliver the result of an operation that takes 60–150 s to the client, under a hard ~30 s limit on HTTP response duration (see HTTP-E5). The limit cannot be raised — it's outside your infrastructure.
**Solution — don't hold the connection open:** instead of one long request, two short ones (about 1 s each).
```
Webhook → IF "Router" (body.action == 'get')
  ├─yes─→ SELECT job → Switch on status
  │        done    → Code(base64→binary) → Respond binary
  │        error   → Respond JSON {status:'error'}
  │        (fallback) → Respond JSON {status:'pending'}
  └─no──→ IF "has prompt?" → INSERT job (pending) RETURNING job_id
           → IF "async?" ─yes→ Respond 202 {job_id} ─┐  (execution CONTINUES)
                          └no───────────────────────┤
                                                    ↓
                          [long-running work] → Code(binary→base64) → IF "async?"
                                              yes → UPDATE job SET status='done', result_b64
                                              no  → Respond binary (old behavior)
```
**Key points:**
- the `Respond to Webhook` node does **not** end the workflow — execution continues after it, which is what "respond right away, finish later" relies on;
- after `Respond`/`INSERT`, `$json` already holds different data → all expressions in the long branch must be rebound to the original request: `$('Webhook').item.json.body.*`;
- job storage is an ordinary table (`job_id text PK, status, result_b64 text, …`), the result is stored as base64 text; old rows are auto-purged via a CTE query on INSERT: `WITH del AS (DELETE FROM t WHERE created_at < now() - interval '1 day' RETURNING 1) INSERT …`;
- the `async` flag in the body keeps backward compatibility (without it — the old synchronous path);
- SQL parameters go through `queryReplacement` as an array expression (see postgres.md PG-R5).
**Example:** the image-generation service (`<workflow_id>`), the `img_jobs` table.
**Source:** reworking the image-generation service.

---

## ⚠️ Errors

### HTTP-E1 — A Code node cannot make an authenticated HTTP request
**Status:** ⚠️ Error (platform limitation)
**Symptom:** in a Code node, `this.helpers.httpRequestWithAuthentication(...)` fails with `The function "helpers.httpRequestWithAuthentication" is not supported in the Code Node`; `requestWithAuthentication` is missing entirely. They formally show up in `Object.keys(this.helpers)`, but are blocked inside the sandbox.
**Cause:** the Code node sandbox forbids authenticated HTTP helpers (credentials cannot be extracted). Only the **unauthenticated** `this.helpers.httpRequest` / `request` are available.
**Fix:** any call to an external API that requires n8n credentials must be made from a **separate HTTP Request node** with `authentication: predefinedCredentialType` + `nodeCredentialType`. The secret stays inside n8n and is not leaked into the exported workflow.json. The Code node is only for assembling/parsing data.
**Source:** the image-generation service (reference branch).

### HTTP-E2 — A pre-built multipart buffer sent to the HTTP node as raw binary → `source.on is not a function`
**Status:** ⚠️ Error
**Symptom:** built a multipart/form-data body as a Buffer in a Code node, passed it to the HTTP Request node via `contentType: binaryData` (raw binary) — the node fails with `source.on is not a function`.
**Cause:** the binary-sending mode expects a stream-like source, not an in-memory Buffer.
**Fix:** don't assemble multipart by hand. Use the HTTP node's **native multipart** support (see HTTP-R2).
**Source:** the image-generation service.

### HTTP-E3 — Web APIs (`URLSearchParams` etc.) are unavailable in the Code node — query/form strings must be built by hand
**Status:** ⚠️ Error (platform limitation) + ✅ solution confirmed (a live payment went through)
**Symptom:** the Code node fails with `URLSearchParams is not defined (ReferenceError)`. The insidious part: the node was written "by eye" and failed on EVERY call, but from the outside it looked like "Invalid server response" to the client (a webhook with a responseNode returns non-JSON when the node fails), and workaround tests (building the URL in Python) masked the bug — the live end-to-end path was first exercised only days later.
**Cause:** the n8n Code node sandbox doesn't provide browser Web APIs: `URLSearchParams`, `fetch`, `TextEncoder`… (the same class of restriction as the `require('crypto')` ban — see telegram.md TG-E10). Only plain JS plus `encodeURIComponent`/`Buffer` are available.
**Fix:** build the query string and form-urlencoded body by hand:
```js
const pairs=[['MerchantLogin',ML],['OutSum',amount],['InvId',String(inv)]];
const qs=pairs.map(p=>encodeURIComponent(p[0])+'='+encodeURIComponent(p[1])).join('&');
```
Difference from `URLSearchParams`: a space is encoded as `%20` rather than `+` — semantically equivalent, both are accepted by parsers.
**Prevention:** (1) run every new Code node through a live call before calling it "done" — "the node is written" ≠ "the node has actually executed"; (2) on "Invalid server response" / non-JSON from a webhook, check the workflow's executions first (`status=error`, `resultData.error.message`), not the network/DPI; (3) grep `URLSearchParams|fetch\(|TextEncoder` across all workflows when in doubt — usually turns up a single node.
**Source:** `<workflow_name>` of the coaching product, the Robokassa payment link, the "nothing works" incident; several executions in a row failed.

### HTTP-E4 — Robokassa: the Receipt goes into the signature WITHOUT URL-encoding (the documentation is wrong)
**Status:** ✅ Confirmed (the production code returns the link, the form accepts it: no errors, `isRecurring: true`, the receipt line item is present)
**Symptom:** added the receipt line-item data (`Receipt`) to the payment request — the payment form started returning `error.code = 29`, the `isRecurring` field reset to `false`, and the receipt composition came back empty. Without `Receipt`, the same request was accepted.
**What the documentation says** (docs.robokassa.ru/ru/fiscalization): "The value is included in the signature: `MerchantLogin:OutSum:InvId:Receipt:Password#1`. Before adding it to the signature string, the Receipt value must be URL-encoded."
**What actually happens:** the production form accepts a signature computed from the **raw JSON, unencoded**. With `encodeURIComponent` — error code 29. Verified by trying five variants (encoded/unencoded signature, `tax: none` vs `vat0`, amount as number vs string, with and without `Recurring`) — exactly one factor made the difference.
**Working formula:** `md5(MerchantLogin : OutSum : InvId : <JSON Receipt as-is> : Password#1 : Shp_… in alphabetical order)`. In the request itself, `Receipt` is encoded normally when building the query string.
**Why Receipt is needed at all:** without it, the receipt is **not generated at all** (a violation of Russian Federal Law 54-FZ), and some payment methods don't show up on the form. In the form's response this shows up as `"receipt":{"items":[]}`.
**How to test without spending money:** build the link to `/Merchant/Index.aspx` and curl it — the payment form returns a JSON status object showing `error`, `isRecurring`, and the receipt composition. An invoice is created, but no charge occurs.
**Composition for a self-employed merchant (Robochecks SMZ):** `{"items":[{"name":"…","quantity":1,"sum":<amount>,"payment_method":"full_payment","payment_object":"service","tax":"none"}]}`. The `sno` field is not passed — it's taken from the merchant's personal account settings.
**Source:** the coaching product, `<workflow_name>` (payment) + `<workflow_name>` (recurring payments).

### HTTP-E5 — A long synchronous webhook response gets cut off at ~30 s OUTSIDE the server
**Status:** ⚠️ Error (network, outside the instance's infrastructure) + ✅ solution: see HTTP-R3
**Symptom:** a request to a webhook whose response takes longer than ~30 s to prepare reliably fails with `curl exit 56` / `Recv failure: Connection reset by peer` at 29–30 s. Meanwhile the workflow in n8n **completes successfully** in 60–120 s and produces a valid result (visible in executions). The same root cause also drops **SSH** connections on long-running commands (`Connection reset by peer`, `timeout during banner exchange`).
**False hypothesis (cost time to rule out):** "a short proxy read/response timeout after the server move." Wrong — the proxy isn't at fault.
**How to test correctly (cheap, without spending on paid APIs):** a temporary probe webhook `Webhook → Code(await setTimeout(delay)) → Respond`, measured from three points:
```
externally  45 s → ❌ cut off at 30.2 s      externally 25 s → ✅ 200
from server 45 s → ✅ 200 (n8n directly)     from server 45 s → ✅ 200 (through proxy)
```
If it succeeds **from the server** but is cut off from outside — some node on the network path (ISP/DPI/NAT/VPN) is at fault, not the proxy. There is NO need to touch the server config.
**Prevention:** TCP keepalive (`curl --keepalive-time`) does NOT help — the cut happens at the application layer. For SSH, application-layer keepalive helps:
```
Host <host>
    ServerAliveInterval 15
    ServerAliveCountMax 10
```
**Source:** the image-generation service, incident (after n8n moved to `<server_ip>`); it showed up on `1024x1536`+`medium/high`.
