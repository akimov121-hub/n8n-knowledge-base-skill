# HTTP requests to external APIs

Calling third-party HTTP APIs from n8n directly (not through a ready-made integration node): the HTTP Request node, multipart/form-data, working with binary data, Code node limitations (no authenticated helpers, no Web APIs), timeouts on long synchronous responses, and the asynchronous polling pattern.

---

## ✅ Solutions

### HTTP-R1 — ~~Comma-separated `inputDataFieldName` for N files~~ REFUTED
**Status:** ⚠️ Refuted (the actual solution is HTTP-R2)
**Symptom / context:** need to send a multipart request to an external API with several files under one field name (`image[]`), with the number of files unknown in advance (1–4).
**What was believed to be the solution:** `parameterType: formBinaryData`, `inputDataFieldName: "={{ $json.refKeys }}"` with a comma-separated list of names ("n8n splits comma-separated lists on its own").
**Why this is wrong:** n8n does NOT split comma-separated lists. In the source (`HttpRequestV3.node.ts`, checked on 2.26.8 and master), `assertBinaryData(itemIndex, cur.inputDataFieldName)` receives the string as **a single field name** → with two or more references the node fails with `The item has no binary field 'ref0,ref1'`. The original entry was apparently confirmed only by single-reference runs (`refKeys="ref0"` — no comma), where the scheme degenerates into an ordinary single field.
**Source:** image generation service; incident (first real call with 2 references).

### HTTP-R2 — Sending N files (dynamic count) in multipart: Switch + K copies of the HTTP node
**Status:** ✅ Confirmed (multi-reference and text mode verified by live runs; the result produced by this scheme was approved by the client)
**Symptom / context:** same as HTTP-R1: multipart with 1–4 files under one field name `image[]`, dynamic file count. Platform limitations (checked against n8n 2.26.8 source):
- comma-separated values in `inputDataFieldName` are not split (see HTTP-R1);
- `formBinaryData` with an empty/nonexistent field name at `typeVersion ≥ 4.2` crashes the node with `Cannot read properties of undefined (reading 'value')` (`prepareRequestBody`: `entry[key].value` on a skipped parameter) — the "empty name = skip parameter" trick does not work;
- assembling a multipart Buffer by hand and passing it as raw binary is not possible — see HTTP-E2 (`source.on is not a function`): the core's legacy adapter rebuilds the body via object spread when the `multipart/form-data` header is set.
**Solution (works):**
1. **Code node** decodes the files into binary properties `ref0…refN-1` (`prepareBinaryData`) and returns `refCount` in json.
2. **Switch node** branches on `{{ $json.refCount }}` (1/2/3/4) into…
3. …**K copies of the HTTP Request node** (`contentType: multipart-form-data`): in the copy for K files, there are exactly K static `formBinaryData` parameters with `name: "image[]"` and `inputDataFieldName: ref0…refK-1`. Duplicate `image[]` names are fine — the FormData path calls `append` for each parameter separately. All copies converge into a single result-decoding node.
4. Authentication in each copy: `authentication: predefinedCredentialType`, `nodeCredentialType: openAiApi` + explicit `credentials` on PUT (see rest-api-deploy.md).
**Example:** image generation service, reference branch: Switch "How many references?" → `OpenAI edits 1/2/3/4 refs` → "PNG from base64".
**Side note:** the gpt-image `/edits` endpoint returns `b64_json` → decoded by a Code node before Respond (as before).
**Source:** fixing the image generation service (incident: the first call with 2 references crashed the workflow).

### HTTP-R3 — Asynchronous webhook (job_id + polling) for operations longer than 30 s
**Status:** ✅ Confirmed (`1024x1536`+`high` → 1.2 MB PNG in 90 s from the client's network, where synchronous mode failed 100% of the time)
**Task:** deliver the result of an operation that takes 60–150 s to a client, under a hard ~30 s limit on HTTP response duration (see HTTP-E5). The limit cannot be raised — it's outside your infrastructure.
**Solution — don't hold the connection open:** instead of one long request, use two short ones (each ~1 s).
```
Webhook → IF "Router" (body.action == 'get')
  ├─yes─→ SELECT job → Switch on status
  │        done    → Code(base64→binary) → Respond binary
  │        error   → Respond JSON {status:'error'}
  │        (fallback) → Respond JSON {status:'pending'}
  └─no→ IF "has prompt?" → INSERT job (pending) RETURNING job_id
           → IF "async?" ─yes→ Respond 202 {job_id} ─┐  (execution CONTINUES)
                          └no───────────────────────┤
                                                    ↓
                          [long task] → Code(binary→base64) → IF "async?"
                                              yes → UPDATE job SET status='done', result_b64
                                              no  → Respond binary (old behavior)
```
**Key points:**
- the `Respond to Webhook` node does **not** end the workflow — execution continues after it, which is what makes "respond immediately, finish later" possible;
- after `Respond`/`INSERT`, `$json` holds different data → all expressions in the long branch must be re-bound to the original request: `$('Webhook').item.json.body.*`;
- the job store is a plain table (`job_id text PK, status, result_b64 text, …`), the result is stored as base64 text; old rows are auto-cleaned via a CTE query on INSERT: `WITH del AS (DELETE FROM t WHERE created_at < now() - interval '1 day' RETURNING 1) INSERT …`;
- the `async` flag in the body keeps backward compatibility (without it, the old synchronous path runs);
- SQL parameters go through `queryReplacement` as an array expression (see postgres.md PG-R5).
**Example:** image generation service (`<workflow_id>`), table `img_jobs`.
**Source:** rework of the image generation service.

### HTTP-R4 — Public temporary file hosting as a bridge between external APIs that need a URL, not a binary
**Status:** 🟡 Draft — a pattern from a GORA course training blueprint, not yet run on our own infrastructure.
**Task:** chain together third-party APIs (in the case at hand, HeyGen for avatar video and DashScope/Alibaba for voice cloning), where one service returns a binary and the next one in the chain accepts **only a public URL**, not a file in the request body.
**Pattern:** between two such services, insert an upload to an anonymous public temporary host (`litterbox.catbox.moe` — POST with the file, response is a direct link to the file with a limited lifetime), and pass that link to the next API as a parameter. Cheap (free, no keys) and avoids running your own file server just for one intermediate step.
**Handle with caution:**
- The same class of risk already confirmed in production in HTTP-E6: free external hosting is a cache, not storage. The link can die before the consuming service picks it up (overload, geo restriction, expiry). For an intermediate step within a single workflow run the risk is small (seconds to minutes between upload and use), but not for anything that needs to be reused later.
- 152-FZ / privacy matters more here than in the image-for-social-post case. This flow sends a **person's voice and video** (voice cloning, avatar video) through an anonymous public service — potentially far more sensitive than a post cover image. The link to the file on such a service is accessible to **anyone** who gets hold of it until it expires. Acceptable for your own experiments; **for client projects where a real person's voice/face is the input, this combination should be discussed separately first** rather than reused as-is — either use your own temporary server with TTL cleanup (the pattern from HTTP-E6, mitigation #2), or a service with private presigned URLs instead of anonymous public hosting.
- The asynchronous polling for video generation status (HeyGen `getVideoStatus` in an `If`/`Wait` loop) in this blueprint is the same class of task already solved more reliably by HTTP-R3 (job_id + short requests instead of holding a connection open); for your own pipelines, follow HTTP-R3 rather than copying the polling loop as-is.
**Source:** GORA course training blueprint ("content factory - video avatar").

### HTTP-R5 — "Python computes, n8n delivers": broadcasting from a static JSON behind Basic Auth
**Status:** ✅ Confirmed.
**When:** the needed logic already exists in another tool (the dashboard-page builder parses dates and splits stages into daily and weekly buckets). Duplicating it in a Code node means having two sources of truth that will drift apart within a week.
**Scheme:** on every build, the page builder writes a machine-readable JSON next to the page (a plan for N days ahead, keyed by ISO date); deployment publishes it to the same host behind the same Basic Auth; n8n: Schedule → HTTP Request (`genericCredentialType` / `httpBasicAuth`, Retry On Fail 3×5 s) → Code takes the key `$now.setZone('Europe/Moscow').toISODate()` and formats it → Telegram (HTML).
**Data honesty:** freshness = last deployment, so the message includes the data date (`updated`), with a warning if it's older than 7 days. If a day's key is missing in the file — `throw` (goes to the error channel); an empty day is still sent — **silence = breakage**.
**Changing the Basic Auth password** on the host breaks the n8n credential; the scenario will fail and report it on its own — no separate monitoring needed.
**Source:** a corporate client, `<workflow_name>` (morning task digest) + the dashboard build script → `agenda.json`.

---

## ⚠️ Errors

### HTTP-E1 — Code node cannot make an authenticated HTTP request
**Status:** ⚠️ Error (platform limitation)
**Symptom:** in a Code node, `this.helpers.httpRequestWithAuthentication(...)` fails with `The function "helpers.httpRequestWithAuthentication" is not supported in the Code Node`; `requestWithAuthentication` is absent entirely. They formally show up in `Object.keys(this.helpers)`, but are blocked inside the sandbox.
**Cause:** the Code node sandbox forbids authenticated HTTP helpers (credentials must not be extractable). Only the **unauthenticated** `this.helpers.httpRequest` / `request` is available.
**Solution:** any external API call that needs n8n credentials must be a **separate HTTP Request node** with `authentication: predefinedCredentialType` + `nodeCredentialType`. The secret stays inside n8n and doesn't leak into the exported workflow.json. The Code node is only for assembling/parsing data.
**Source:** image generation service (reference branch).

### HTTP-E2 — A pre-built multipart buffer sent to the HTTP node as raw binary → `source.on is not a function`
**Status:** ⚠️ Error
**Symptom:** assembled the multipart/form-data body as a Buffer in a Code node, passed it to the HTTP Request node via `contentType: binaryData` (raw binary) — the node fails with `source.on is not a function`.
**Cause:** binary send mode expects a stream-like source, not an in-memory Buffer.
**Solution:** don't build multipart by hand. Use the HTTP node's **native multipart** support (see HTTP-R2).
**Source:** image generation service.

### HTTP-E3 — Web APIs (`URLSearchParams` etc.) are unavailable in the Code node — query/form strings must be built by hand
**Status:** ⚠️ Error (platform limitation) + ✅ solution confirmed (payment went through live)
**Symptom:** the Code node fails with `URLSearchParams is not defined (ReferenceError)`. The tricky part: the node was written "by eye" and failed on EVERY call, but from the outside it looked like "Invalid server response" to the client (a webhook with a responseNode returns non-JSON when the node crashes), and workaround tests (building the URL in Python) masked the bug — the live end-to-end path was first exercised only days later.
**Cause:** the n8n Code node sandbox doesn't provide browser Web APIs: `URLSearchParams`, `fetch`, `TextEncoder`… (the same class of restriction as the ban on `require('crypto')` — see telegram.md TG-E10). Only plain JS plus `encodeURIComponent`/`Buffer` are available.
**Solution:** build query strings and form-urlencoded bodies by hand:
```js
const pairs=[['MerchantLogin',ML],['OutSum',amount],['InvId',String(inv)]];
const qs=pairs.map(p=>encodeURIComponent(p[0])+'='+encodeURIComponent(p[1])).join('&');
```
Difference from `URLSearchParams`: a space is encoded as `%20` instead of `+` — semantically equivalent, and parsers accept both.
**Prevention:** (1) run every new Code node through a live call before calling it "done" — "the node is written" ≠ "the node has been executed"; (2) on "Invalid server response" / non-JSON from a webhook, check the workflow's executions first (`status=error`, `resultData.error.message`) rather than the network/DPI; (3) grep `URLSearchParams|fetch\(|TextEncoder` across all workflows when in doubt — usually finds a single offending node.
**Source:** `<workflow_name>` of a coaching product, the Robokassa payment link, "nothing works" incident; several executions in a row failed with an error.

### HTTP-E4 — Robokassa: the Receipt goes into the signature WITHOUT URL encoding (the docs are wrong)
**Status:** ✅ Confirmed (production code sends the link, the form accepts it: no errors, `isRecurring: true`, receipt line item present)
**Symptom:** added a receipt item (`Receipt`) to the payment request — the payment form started returning `error.code = 29`, the `isRecurring` field reset to `false`, and the receipt contents came back empty. Without `Receipt`, the same request was accepted.
**What the docs say** (docs.robokassa.ru/ru/fiscalization): "The value is included in the signature: `MerchantLogin:OutSum:InvId:Receipt:Password#1`. The Receipt value must be URL-encoded before being added to the signature string."
**What actually happens:** the production form accepts a signature computed over the **raw JSON, unencoded**. With `encodeURIComponent` applied — error code 29. Verified by trying five variants (encoded/unencoded signature, `tax: none` vs `vat0`, amount as number vs string, with and without `Recurring`) — exactly one flag made the difference.
**Working formula:** `md5(MerchantLogin : OutSum : InvId : <JSON Receipt as-is> : Password#1 : Shp_… in alphabetical order)`. In the request itself, `Receipt` is encoded normally when building the query string.
**Why Receipt is needed at all:** without it, the receipt **isn't generated at all** (violates 54-FZ), and some payment methods don't show up on the form. This shows up in the form response as `"receipt":{"items":[]}`.
**How to test without spending money:** build the link to `/Merchant/Index.aspx` and curl it — the payment form returns a JSON status showing `error`, `isRecurring`, and the receipt contents. The invoice is created, but no charge occurs.
**Receipt contents for a self-employed payee:** `{"items":[{"name":"…","quantity":1,"sum":<amount>,"payment_method":"full_payment","payment_object":"service","tax":"none"}]}`. The `sno` field is omitted — it's taken from the merchant account.
**Source:** coaching product, `<workflow_name>` (payment) + `<workflow_name>` (recurring payments).

### HTTP-E5 — A long synchronous webhook response is cut off at ~30 s OUTSIDE the server
**Status:** ⚠️ Error (network, outside the instance's infrastructure) + ✅ solution: see HTTP-R3
**Symptom:** a request to a webhook whose response takes longer than ~30 s to prepare consistently fails with `curl exit 56` / `Recv failure: Connection reset by peer` at 29–30 s. Meanwhile the workflow in n8n **completes successfully** in 60–120 s and returns a valid result (visible in executions). The same root cause also causes **SSH** disconnects on long-running commands (`Connection reset by peer`, `timeout during banner exchange`).
**False hypothesis (cost time to rule out):** "a short proxy read/response timeout after the server move." Wrong — the proxy is not at fault.
**How to test correctly (cheaply, without spending on paid APIs):** a temporary probe webhook `Webhook → Code(await setTimeout(delay)) → Respond`, measured from three vantage points:
```
external  45 s → ❌ cut off at 30.2 s   external 25 s → ✅ 200
server    45 s → ✅ 200 (n8n directly)  server   45 s → ✅ 200 (via proxy)
```
If it succeeds **from the server** but fails externally, the culprit is a node on the network path (ISP/DPI/NAT/VPN), not the proxy. No need to touch server configs.
**Prevention:** TCP keepalive (`curl --keepalive-time`) does NOT help — the cutoff happens at the application layer. For SSH, application-layer keepalive helps:
```
Host <host>
    ServerAliveInterval 15
    ServerAliveCountMax 10
```
**Source:** image generation service, incident (after moving n8n to `<server_ip>`); showed up with `1024x1536`+`medium/high`.

### HTTP-E6 — External image hosting: the link lives its own life, and gets checked at the worst possible moment
**Status:** ✅ Confirmed — the fix has been rolled out to production.
**Symptom:** a post with a finished cover image failed to publish. The scenario failed at the "Download cover" node: `The resource you are requesting could not be found` — imgBB returned a **404 and a 1031-byte placeholder PNG**. Of three cover images from the same series, uploaded in one batch, **two** disappeared within a day.
**Investigation:** the uploads had succeeded (`success: true`, each assigned its own URL, `expiration: 1209600` = 14 days, i.e. not yet expired). The `ibb.co/<id>` pages were still alive and showed the same addresses — yet the `i.ibb.co` CDN returned 404 for **all** path variants. Size was not the factor: cover images from other series at 1.8–2.3 MB were still alive, while the missing files were 307 and 273 KB. The cause is on imgBB's side and can't be reproduced or prevented from our end.
**What's actually on us:** **days** pass between upload and publication, and during all that time "the cover exists" only means "there's a row in the DB with an address." The check happens exactly at the moment of publication — the one moment when it's too late to fix.
**Recovery:** the source files were found in the ingestion webhook's execution data — the request body with `image_base64` is stored in n8n's execution history. They were re-uploaded via the standard "queue_id + image_base64 → update cover" branch; the series didn't need to be rebuilt. **Takeaway: n8n's execution history is a working binary archive, as long as execution retention hasn't expired.** For screenshots and manual graphics it's the only source — they can't be regenerated.
**Mitigations (in increasing order of reliability):** (1) check `image_url` availability ahead of time — an hour before publication, so there's time to fix it; (2) store cover images on your own server, keeping imgBB only as a public mirror for Instagram; (3) at minimum, don't leave the record stuck after such a failure.
**General rule:** free external hosting is a cache, not storage. Anything you'll need in the future should live where you control the lifetime; an external link should be checked ahead of time, not at the moment the result depends on it.
**Source:** a publishing product, `<workflow_name>` → "Download cover."
**How it was resolved (owner's decision — yes to both points):** cover images were moved entirely to our own server (`/var/www/pub-media` → served by nginx at `/pub-media/`), imgBB removed from the pipeline in all three workflows; a pre-flight check was added an hour before publication, notifying the owner in the internal work bot; file lifetime is now managed by us — a cron job deletes covers older than a week, except those still needed by the queue or drafts.
**Status (after the fix):** ✅ Confirmed in production: the post went out on time, the scenario downloaded the cover from our own server (`<N8N_BASE_URL>/pub-media/...`), and Telegram and Instagram both accepted it (permalink obtained). The pre-flight check ran exactly once before publication, found the cover alive, and stayed silent. The root cause of the disappearing images remains on imgBB's side, but there's no longer any dependency on it.
