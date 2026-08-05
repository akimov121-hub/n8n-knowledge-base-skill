# Telegram

Telegram nodes: Trigger, sendMessage, sendPhoto, inline buttons, receiving voice/video notes.

---

## ✅ Solutions

### TG-R1. Always disable the n8n attribution caption

- **Context:** by default Telegram nodes append "This message was sent automatically
  with n8n".
- **Solution:** set `appendAttribution: false` on every Telegram node.

Status: ✅ Project rule.

### TG-R2. Default parse_mode for all text nodes — `HTML` + escaping `<>&`

- **Context:** any sendMessage / sendPhoto where the text or caption is built dynamically
  (from an LLM, from a user, from an API) needs safe formatting. Markdown (the default)
  breaks on stray `_ * [ \``; leaving parse_mode unset also falls back to Markdown
  (see TG-E1).
- **Solution:**
  1. Set `parse_mode: "HTML"` (snake_case!) on ALL Telegram nodes that send text or a caption.
  2. Anywhere text/caption may contain user input or an LLM reply — **always** run it through
     an escape function:
  ```js
  const esc = s => String(s||'').replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;');
  ```
  3. If the LLM itself writes HTML markup, require this explicitly in the system prompt:
     only `<b>`/`<i>`/`<u>`/`<s>`/`<a href="...">`/`<code>`/`<pre>`/
     `<tg-spoiler>`/`<blockquote>` are allowed; tags must be balanced; plain text must use
     `&amp;`/`&lt;`/`&gt;`.
- **What this covers:** `_` and `*` inside `@username`, `@channel_name` won't break a post;
  arbitrary user-supplied text like `text < 10` becomes `text &lt; 10` after `esc()`;
  an LLM-generated post with `<b>title</b>` renders bold.

Status: ✅ Confirmed in production.

### TG-R3. "Typing…" indicator (sendChatAction) — wire it so you don't lose data

- **Context:** show the user "typing…" while the bot is thinking (especially ahead of a long
  LLM node).
- **Node setup:** Telegram node, `operation: sendChatAction`, `chatId: ={{ $json.chat_id }}`
  (from a normalizing Set node right before it).
- **Wiring (KEY POINT):** the `sendChatAction` node **overwrites the current item** with
  Telegram's API response (see TG-E4). Two working approaches:
  1. Every node **after** Send Typing reads data not from `$json` but from the upstream node
     by name: `$('Normalize').first().json.<field>`.
  2. Or place Send Typing on a **side (dead-end) branch** so the main data flow never passes
     through it.
- **Note (Telegram API behavior):** the "typing…" status lasts about 5 seconds and clears on
  its own. For long-running responses you'd need to resend it.

Status: ✅ Confirmed.

### TG-R4. Inline buttons and forceReply in the Telegram node (sendMessage)

- **Context:** attach an inline keyboard (callback or URL buttons), or request a reply via
  ForceReply.
- **Solution:**
  - Inline keyboard: `parameters.replyMarkup = "inlineKeyboard"`, then
    `parameters.inlineKeyboard.rows = [{ "row": { "buttons": [ ... ] } }]`. Each button:
    `{ "text": "...", "additionalFields": { ... } }`.
    - callback button: `additionalFields.callback_data` (**snake_case!**).
    - URL button (including a deep link `t.me/<bot>?start=<payload>`):
      `additionalFields.url`.
  - ForceReply: `parameters.replyMarkup = "forceReply"`,
    `parameters.forceReply = { "force_reply": true }`.
  - camelCase (`callbackData`, `forceReply:true`) is silently dropped by n8n → the button
    never appears (DEP-E4). After a PUT, always do a GET readback to confirm the keys made
    it through.
  - answerCallbackQuery: `resource:"callback"`, `operation:"answerQuery"`, `queryId` from
    `$('<node with the update>').first().json.<cb_id>`.

Status: ✅ Structure verified via GET (camelCase is dropped, snake_case persists).

### TG-R4*. Approval loop (preview with buttons + text/voice edits) via a Postgres state machine

**Context.** Needs human-in-the-loop: the bot shows a result, the owner taps buttons
(approve/revise), and edits can be sent as text OR voice. The "Send and Wait" node doesn't
fit: it accepts free text only through an n8n web form, not a chat message, and doesn't
accept voice at all.

**Solution** (✅ confirmed in production):

1. **State lives in a Postgres table** (`<bot>_drafts`, keyed by `chat_id`, with a `stage`
   column plus draft data). Every message/button press is a new workflow run; the stage
   determines the route. Survives n8n restarts, waits are unbounded, and executions stay
   short.
2. **Trigger updates = `["message","callback_query"]`**. Access check and chatId via
   `COALESCE`: `message?.from?.id || callback_query?.from?.id`.
3. **State load always returns exactly 1 row** (otherwise a branch with no items silently
   fails to fire):
   `SELECT COALESCE(d.stage,'') ... FROM (SELECT 1) x LEFT JOIN <bot>_drafts d ON d.chat_id = {{ Number(...) }}`.
4. **Router (Switch, loose v2)**: first match button callback_data (string equals), then
   `/start`, then input-waiting stages (regex on stage + presence of a message), then a
   fresh request (text/caption not empty), fallback — a polite decline.
5. **Buttons** — a plain sendMessage with `replyMarkup: inlineKeyboard`. Every callback
   branch starts with `resource: callback` (answerQuery, `queryId = callback_query.id`),
   otherwise the button keeps spinning.
6. **Voice only at wait points**: IF `!!message?.voice` → Telegram `resource: file`
   (voice.file_id) → openAi `resource: audio, transcribe, language: ru` → a shared "Input
   text" Code node → Switch on stage. Outside a wait stage, voice is rejected.
7. **Text values in SQL** — inline literals with quote escaping:
   `'{{ expr.replace(/'/g, "''") }}'` (queryReplacement breaks on commas inside text). All
   UPDATEs use `RETURNING *`.
8. Completion is not a DELETE but `stage='published'`: the next request overwrites the row
   via UPSERT.

### TG-R5. Accepting video notes (video_note) alongside voice messages

**Context.** The bot needs to accept not only voice but also video notes (video_note) and
treat them as voice input (transcribing the audio track).

**Solution** (✅ confirmed in production):

1. **Detection** — check both types in an IF/Switch:
   `!!($json.message?.voice || $json.message?.video_note)`.
2. **Download** — Telegram node `resource: file`,
   `fileId = {{ message.voice?.file_id || message.video_note?.file_id }}` (one node handles
   both).
3. **Transcription** — openAi `resource: audio, operation: transcribe, language: ru` accepts
   the mp4 video note **directly**; no need to extract the audio track separately.
4. From there, reuse the shared "Input text" code, same as for voice (TG-R4* step 6).

**Protecting the draft against unsolicited voice/video notes.** If a voice message/video note
can arrive both as a "new intent" (no active draft) and as an edit (during a wait stage), add
a second condition to the "new intent" rule based on stage: only fire when there is no draft
(`stage` is empty/`published`/`cancelled`). Otherwise a stray voice message during a ready
preview will clobber the draft.

### TG-R6. Retry On Fail on outbound Telegram nodes (protection against transient failures)

**Context.** Outbound Telegram calls (sendMessage / sendPhoto / sendChatAction / getFile)
occasionally fail transiently — e.g. a sporadic "chat not found" with a fully valid config
(TG-E7) or a network blip. One such failure aborts the whole run and loses a message /
notification / lead.

**Solution.** Enable **node-level** retry on the sending/downloading Telegram nodes:
`retryOnFail: true`, `maxTries: 3`, `waitBetweenTries: 5000` (these are fields of the node
itself, not inside `parameters`).

**⚠️ Exception — public posting.** Retry gives you **at-least-once** semantics: if Telegram
delivered the message but the response never reached n8n (a drop exactly between delivery
and the ack), the retry creates a **duplicate**. A duplicate is tolerable for acks / replies
to the user / owner notifications / file downloads. For nodes posting to a **public
channel/feed** (e.g. the "Publish to Telegram" node), do **NOT** enable retry — a duplicate
there means a double public post.

**Deployment.** Set the three fields on the node, PUT, then GET readback to confirm
`retryOnFail` persisted. Credential bindings are untouched in the process (on the current
instance, GET returns `credentials` — just pass them through unchanged).

**Source.** Rollout of retry across 8 active workflows (52 outbound Telegram nodes; public
posting excluded). Original case: TG-E7.

**Status:** ✅ Confirmed in production.

### TG-R7. Send images to Telegram as binary (download → sendPhoto binaryData), not by URL

**Context.** The image is hosted at a public URL (imgBB — needed for the Instagram Graph
API). Delivering it to Telegram by link is unreliable (TG-E8) and capped at 5 MB.

**Solution.** Insert an HTTP Request node before the Telegram node that downloads the URL as
binary, and send it via sendPhoto as binary:
- HTTP Request (GET, `url={{ image_url }}`),
  `options.response.response = { responseFormat: "file", outputPropertyName: "data" }`;
  `retryOnFail` on the download is fine (idempotent GET).
- Telegram sendPhoto: `binaryData: true`, `binaryPropertyName: "data"`, leave the `file`
  field unset. Pull `caption`/`parse_mode` from a named node
  (`$('Load Draft').first().json.body_text`), since after the HTTP node `$json` holds binary
  data, not text.

Eliminates both classes of TG-E8 errors and the 5 MB limit (binary photo goes up to 10 MB).
imgBB stays in use for Instagram — the Graph API pulls the URL on its own side.

**⚠️ Do NOT enable retry on the actual sendPhoto node publishing to the channel**
(duplicate risk, TG-R6) — retry is only acceptable on the download node.

**Source.** Publisher product, "Download Cover" → "Final Preview: photo" and
"Download Cover (publish)" → "Publish to Telegram".

**Status:** ✅ Confirmed in production.

### TG-R8. "Private chat only" filter at the bot entry point

**Context.** A bot for personal use (owner in a DM). Updates from channels/groups are noise
and a source of errors (TG-E9).

**Solution.** Right after the Telegram Trigger (before the access check and any replies) —
an IF on chat type:
```
chat_type = {{ ($('Telegram Trigger').item.json.message || $('Telegram Trigger').item.json.edited_message || $('Telegram Trigger').item.json.callback_query?.message || {}).chat?.type || '' }}
```
`chat_type === 'private'` → proceed; otherwise → NoOp (silently ignore, no reply). Covers
message, edited_message, and callback_query all at once.

**Source.** Publisher product, "Private chat?".

**Status:** ✅ Confirmed in production.

### TG-R9. Treating an edit to the user's own message (edited_message) as fresh verbatim input

**Context.** The user needs to be able to correct already-sent text not by sending a new
message but by editing their own (a "fix it in place" UX).

**Solution.**
1. Add the `edited_message` update type to the Telegram Trigger (alongside `message`,
   `callback_query`).
2. Everywhere incoming data is read, support both branches:
   `message?.… || edited_message?.…` (chat.id, from.id, text — in CHAT/CHATNUM, Check
   Access, save nodes).
3. In the router: a rule "there is an `edited_message` AND stage = `<manual input mode>`" →
   save `edited_message.text` verbatim → show the preview. Any other `edited_message` (not
   in the right stage) → silently ignore it (otherwise editing an old message would falsely
   trigger the flow).
4. Limitation: the bot can only catch edits to the user's **own** messages; the first input
   still has to arrive as a regular message, only subsequent corrections come as edits.
   Telegram only surfaces edited_message for about ~48 hours.

**Source.** Publisher product, "Save texts (message edit)" + the `edit_text_manual` rule.

**Status:** ✅ Confirmed in production.

### TG-R10. Gracefully handling broken user-supplied HTML on send (onError → hint)

**Context.** If the text to send is authored by the user (manual input, their own finished
post), its HTML may be invalid (`<b>…<b>` instead of `</b>`). sendMessage/sendPhoto with
`parse_mode: HTML` fails on this (`can't find end tag …`) — the node turns red, and the user
gets no feedback.

**Solution.** On the preview node (the first one that sends the user's text with parse_mode
HTML), set `onError: continueErrorOutput`. The error output (index 1) → a separate static
hint node (with no tags in its own text, so it can't fail itself): "broken markup, closing
tag is `&lt;/b&gt;`, fix it and resend." The preview acts as a "gate": as long as the markup
is broken, publishing is unreachable, and the user sees a clear reason. Keep the stage in
"verbatim" mode so a resend stays verbatim.

**Source.** Publisher product, "Text preview" (error output) → "Preview: broken markup".

**Status:** ✅ Confirmed in production.

### TG-R11. Dynamic list of cards with inline buttons (id in callback_data via expression)

- **Context:** need to show the user a list of database records (a post queue, tasks), each
  with its own action buttons.
- **Solution:** don't build one shared keyboard — send a **card per item**: a Postgres node
  returns N rows → the Telegram node runs once per item. In the button's callback_data, use
  an expression with the record id: `"=q_now_{{ $json.id }}"`. The callback handler in the
  router matches by regex (`^q_now_`) and extracts the id via `split('_')[2]`.
- **Advantages:** no need to dynamically assemble a reply_markup JSON blob; each card is
  self-contained; actions are atomic by id (`UPDATE ... WHERE id AND status` — protects
  against stale buttons).
- **Empty list:** Postgres returns `{success:true}` on 0 rows (PG-E2) — an IF guard on a
  required field before the card is mandatory, or you'll send a card full of garbage.
- **Source:** publisher product, `/queue` command (queue cards with 🚀/🕒/❌).
- **Status:** ✅ Confirmed in production.

### TG-R12. Bot menu button = Web App (Mini App) instead of a command list

- **Goal:** make the bot's menu button (bottom-left of the chat) open the Mini App instead
  of showing a command list.
- **Solution (Bot API, no BotFather needed):**
  `POST https://api.telegram.org/bot<token>/setChatMenuButton`
  with body `{"menu_button":{"type":"web_app","text":"Open","web_app":{"url":"https://<app-url>/"}}}`.
  Omit `chat_id` for a default that applies to everyone. The URL must be HTTPS. To revert:
  `{"menu_button":{"type":"commands"}}`.
- **⚠️ Readback gotcha:** right after `set` returns (ok:true), `getChatMenuButton` may still
  return the old value (`{type:'commands'}`) — this is **Telegram's propagation delay**, not
  an error. Recheck a minute later: the default reads back as `{type:'web_app',...}`, and a
  specific chat shows `{type:'default'}` (inheriting the default). Don't "fix" it by
  re-issuing `set` based on the first read.
- **Token:** pull it from `*_config`/execution data, server-side only — never print it to
  chat or logs.
- **Source:** publisher bot.
- **Status:** ✅ Confirmed in production (button opens the app).

### TG-R13. Forwarded message/link → Mini App inbox: detect and gate before the router

- **Goal:** the bridge bot should drop a forwarded news item into `pub_inbox` (→ the
  "Forwarded" block in the Mini App), rather than process it in-chat.
- **Gotcha 1 — detecting only via `forward_date`.** The "Forwarded?" node checked
  `!!message.forward_date`. But when a user **shares a link** (via the share sheet or a
  pasted URL), Telegram sends it as a **plain message with no forward fields whatsoever**
  (`message` = message_id/from/chat/date/text/entities — neither `forward_date` nor
  `forward_origin`). So link-based news wasn't recognized as "forwarded." Fix: broaden the
  condition —
  `!!m.forward_date || /^https?:\/\//i.test((m.text||m.caption||'').trim())`. (Also worth
  remembering: Telegram is gradually replacing the legacy `forward_from*`/`forward_date`
  fields with `forward_origin` — for genuine forwards, check both.)
- **Gotcha 2 — the detector node ran IN PARALLEL with the router.**
  `Load Draft → [Router, Forwarded?]` fired simultaneously: even a recognized forward still
  flowed into the router too → with an active draft, the URL got "swallowed" as a text edit.
  Fix: turn the detector into a **gate BEFORE the router** —
  `Load Draft → Forwarded?`; true → inbox; **false (output 1) → Router**. The IF node passes
  the same item through, so the router still gets identical data. Now forwards/links only go
  to the inbox.
- **Gotcha 3 — silent write.** After the INSERT into `pub_inbox`, the bot stayed silent →
  it looked like "nothing happened." Add a confirmation reply ("🔗 Added to the app → open
  it, see the 'Forwarded' block").
- **The inbox node is resilient to missing forward fields:** build `src` with fallbacks
  (`forward_origin.chat.title || sender_user || forward_from... || ''` → "Repost"), `text` =
  `message.text||caption`. For a bare link, src="Repost", text=URL — it doesn't break.
- **Source:** publisher bot (`<workflow_id>`).
- **Status:** ✅ Confirmed in production.

### TG-R14. Signing into the web app via Telegram Login Widget → session HMAC token (secret=SHA256(token), ≠ initData)

- **Goal:** authenticate a user in the **browser-based** web version of a Telegram product
  (not the Mini App) on the n8n backend, then reuse the same data endpoints as the Mini App.
- **Key difference from the Mini App signature.** Telegram exposes **two distinct** signing
  schemes that are easy to mix up:
  - **Login Widget** (the "Log in with Telegram" widget on a website):
    `secret = SHA256(bot_token)` (binary digest), `data_check_string` = every widget field
    **except `hash`**, sorted by key, `key=value`, joined with `\n`;
    `check = HMAC_SHA256(dcs, secret)` in hex must equal `hash`. Additionally,
    `auth_date` must be ≤ 86400 s old (anti-replay).
  - **Mini App initData** (see TG-E11): `secret = HMAC_SHA256("WebAppData", bot_token)`.
    **Not interchangeable** with the widget scheme.
- **Session instead of re-validating.** The widget's signature is valid only once (the
  widget isn't resent on every request). After a successful `auth/telegram`, the backend
  issues a **self-contained session token** with no server-side session table:
  `base64url(JSON{telegram_id,exp}) + "." + base64url(HMAC_SHA256(payload, SESSION_SECRET))`.
  Expiry (`exp`) — e.g. 30 days. From then on, every request carries the token; the backend
  verifies the signature + `exp` and takes `user_id` **from the token, never from the
  payload** (protects against spoofing).
- **`SESSION_SECRET` lives in the DB config** (table `*_config`, its own key), read by the
  `Load Config` node at the start of the workflow; never appears in code/responses/logs.
  One secret per instance, generated once.
- **Reusing queries.** After the token gate, the same Postgres queries used by the Mini App
  (profile, facts, history, journal, awards) serve the web client too — isolation is achieved
  by **duplicating the SQL in a separate workflow** (`<workflow_name>`), with no shared nodes
  with the miniapp/main bot.
- **Access gate** — the shared `bot_access(bot='<slug>', active=true)` check keyed on
  `telegram_id` from the verified signature; no access → no token is issued (empty string),
  and the frontend shows "no access."
- **Source:** coaching product, `<workflow_name>` (webhook `<webhook_path>`, id
  `<workflow_id>`).
- **Status:** ✅ Confirmed in production (valid hash → token; forged/expired → 401).

### TG-R15. Secure Telegram avatar proxy — signed URL, bot token never on the client

- **Goal:** show a user's real Telegram avatar in the web app/Mini App without **exposing
  the bot token** to the client (Telegram's file URL contains the token in the path).
- **Solution — a dedicated proxy workflow with a signed URL:**
  1. Endpoint `GET /webhook/<webhook_path>?uid=<telegram_id>&sig=<signature>`, where
     `sig = HMAC_SHA256(String(uid), AVATAR_SECRET)` in hex. An invalid/missing signature →
     **403**. The signature is generated by the main API when it issues `photo_url` — the
     client can't forge `uid`.
  2. Given a valid signature: Bot API `getUserProfilePhotos(user_id=uid, limit=1)` → take the
     largest size (the last element of `photos[0]`) → `file_id` → `getFile` → `file_path` →
     download `https://api.telegram.org/file/bot<token>/<file_path>` (HTTP node,
     response=file) → **return the raw bytes** (`image/jpeg`).
  3. Caching: respond with `Cache-Control: public, max-age=86400`. No photo / privacy
     restriction / Bot API error → **404** (the frontend shows a placeholder via `onerror`).
- **Security:** `bot_token` and `AVATAR_SECRET` come from `*_config` (the `Load Config`
  node); they never appear in the response/redirect/headers/logs. Downloading happens
  entirely on the n8n side → only the image bytes leave, never a URL containing the token.
  HTTP nodes calling the Bot API: `retryOnFail` 3×5s, `getUserProfilePhotos`/`getFile` with
  `onError: continueRegularOutput` (graceful degradation to 404).
- **Consistency:** the Mini App should also use the proxy URL rather than
  `initData.photo_url` — one code path on the frontend, and the image is always fresh and
  cacheable.
- **Source:** coaching product, `<workflow_name>` (webhook `<webhook_path>`, GET, id
  `<workflow_id>`).
- **Status:** ✅ Confirmed in production (valid signature → 200 image/jpeg 640×640; broken
  signature → 403; no token ever appears in the response).

### TG-R16. Channel mirroring — an outbound sendMessage from the web branch into the Telegram conversation doesn't create a loop

- **Goal:** a user's conversation with the agent **from the website** should also be visible
  in their Telegram chat with the bot (a single conversation across two channels).
- **Solution:** once the web branch has produced a reply, **in parallel** with the web
  response, send two `sendMessage` calls to Telegram as the bot: an echo of the user's
  message (tagged with the channel, e.g. `🌐 <Name> (from the website):\n<text>`) plus the
  agent's reply. The name comes from `profile.name` keyed on `telegram_id` (via the same
  query that loads the prompt/profile — no extra node needed).
- **Why there's no loop:** an outbound `sendMessage` **from the bot** doesn't generate an
  inbound update in the Telegram Trigger (the bot doesn't "hear" itself) → the main workflow
  never fires. Mirroring is safely one-directional.
- **Reliability (important):** the mirror branch runs **in parallel** with the web-response
  node (`Respond to Webhook` replies first, independently). The Telegram HTTP nodes use
  `onError: continueRegularOutput` + `retryOnFail` 3×5s: **a Telegram failure never breaks
  the web response** and never fails the execution. Conversation memory is **not
  duplicated** — the agent's memory node writes it during the main pass; the mirror only
  sends to Telegram.
- **Formatting is reused** from the main bot (`stripLeak` + converting the
  `⟪PROGRAM⟫` sentinel → `<blockquote expandable>`, `parse_mode=HTML` only when a block is
  present). Use the **raw agent output**, not the version already "cleaned" for the web (in
  case that cleanup strips the sentinels). No token/`user_id` → the node returns `[]` (mirror
  is skipped).
- **Source:** coaching product, `<workflow_name>` branch `chat/send`
  (`Web Mirror Format`/`Mirror Echo`/`Mirror Reply`).
- **Status:** ✅ Accepted in production (logic verified; actual delivery is tracked from a
  live web session, at the owner's discretion).

### TG-R17. Opening the Mini App on a specific screen from a bot message — inline web_app button + deep link via query

- **Goal:** in a bot message (e.g. a weekly ping), give a button instead of a bare link —
  one that opens the Mini App directly on the target screen.
- **Solution:** add a `reply_markup` with an inline `web_app` button to `sendMessage`:
  `reply_markup: {inline_keyboard:[[{text:'🗓 Open Summary', web_app:{url:'https://domain/app?screen=weekly'}}]]}`.
  The domain must match the Mini App's domain (the one registered in BotFather). The button
  opens a webview right inside Telegram. It's also worth adding a text hint in the message
  itself ("check your Journal"), not just the button.
- **Frontend-side deep link:** on startup, read the screen from
  `new URLSearchParams(location.search).get('screen')`, and in Telegram mode also check
  `tg.initDataUnsafe.start_param` (fallback). Use the value to set the initial tab before the
  first render.
- **n8n nuance:** in the httpRequest node, build `jsonBody` via
  `JSON.stringify({chat_id, text, reply_markup})`, preparing the `reply_markup` object in a
  preceding Code node (to avoid an unwieldy inline expression).
- **Source:** coaching product, `<workflow_name>` Ping + web frontend boot.
- **Status:** ✅ Confirmed in production (button delivered, opens the correct screen).

---

## ⚠️ Errors

### TG-E1. `parse_mode: ""` doesn't disable Markdown

- **Symptom:** you want to send "raw" text, you set `parse_mode: ""` — but Telegram still
  parses it as Markdown, and the text arrives formatted/broken on special characters.
- **Cause:** an empty string isn't treated as "no formatting."
- **Fix:** set `parse_mode: "HTML"` (not an empty string, not Markdown). Uncovered after
  4 debugging pings via the sandbox workbench (see debugging-sandbox.md DBG-R1).

### TG-E2. sendPhoto — the parameter is `file`, not `photo`; parse_mode is snake_case

- **Symptom:** you configure a photo send via the API, and the photo doesn't go out / the
  parameter is ignored.
- **Cause:** in n8n's Telegram node, the photo parameter is named `file`, not `photo`.
  Names are case-sensitive: `parse_mode` must be snake_case. n8n silently drops an incorrect
  name (see rest-api-deploy.md DEP-E4).
- **Fix:** use `file`; parse_mode in snake_case; always GET readback after a PUT.

### TG-E3. "sent automatically with n8n" caption on messages

- **Symptom:** bot messages have an n8n attribution line at the bottom.
- **Cause:** `appendAttribution: false` wasn't set.
- **Fix:** see TG-R1.

### TG-E4. After sendChatAction ("Send Typing"), downstream nodes lose chat_id/message_id/text

- **Symptom:** after a Telegram `sendChatAction` node, the following nodes receive
  `undefined` instead of `chat_id` / `message_id` / `text`. DB writes end up null, replies
  go to the wrong place.
- **Cause:** the Telegram node (like most action nodes in n8n) **replaces the current item
  with its own API response** — afterward `$json` is roughly `{ "result": true }`, not the
  data from the normalizing node.
- **Fix:** in nodes after Send Typing, reference the upstream node by name —
  `$('Normalize').first().json.<field>`, not `$json.<field>`. Or move Send Typing to a side
  branch. Full pattern: TG-R3.

### TG-E5. `{{ }}` placeholders in a node's text arrive as raw literal text

- **Symptom:** the bot's message shows the literal `{{ $json.foo }}` instead of a value.
  Worse, this breaks downstream logic that searches for the substituted marker.
- **Cause:** n8n only treats a string field as an **expression** if it starts with `=`. Text
  containing `{{ }}` without a leading `=` is sent literally. Easy to miss on static text
  where the substitution is only in the middle.
- **Fix:** any field (`text`, `chatId`, `callback_data`, `url`) that contains `{{` must start
  with `=`. In a generator, use a helper:
  `'=' + v if '{{' in v and not v.startswith('=') else v`. GET readback after PUT.

### TG-E6. "Bad Request: chat not found" on sendMessage + the bot-selection rule

- **Symptom.** The Telegram sendMessage node fails with
  `Bad request - please check your parameters`, with `Bad Request: chat not found` in the
  `description`. The credential is valid (there's no "Credential … does not exist" error).
- **Cause.** The bot tied to the credential physically cannot message this `chatId`, because
  the user has **never pressed Start / never messaged that bot**. A private chat with a bot
  only exists after the person messages it first. This is NOT a parameter error — Telegram
  simply doesn't know of that chat for that bot. Each bot has its own separate private chat.
- **Fix.** Open the correct bot in Telegram and send it any message (Start). After that,
  sendMessage to that chatId succeeds. Make sure you message the exact bot whose credential
  is set on the node.
- **Related rule.** During a build, Claude once wired notifications to the wrong bot (a
  consultant bot) on its own initiative — this must not happen. **Before wiring ANY
  bot/credential into a workflow — always ask which bot to use. Never guess, and never reuse
  someone else's bot.** One bot = one Telegram Trigger (webhook): reusing a trigger bot in a
  different scenario breaks the original one.
- **Status:** ⚠️ Documented.

### TG-E7. Sporadic "chat not found" with a fully valid config (Telegram transient)

- **Symptom.** sendMessage fails with `Bad Request: chat not found`, but the config is
  correct and **the same bot successfully messages the same chat on neighboring runs**.
  Different from TG-E6: there the bot was never started by the user — it fails
  deterministically every time; here the failure is a one-off.
- **Sign of a transient.** 2 out of 3 identical runs (same `chatId`, same bot, same node, the
  only difference being the inline button's URL, which Telegram doesn't check for existence)
  succeeded, one failed.
- **Fix.** Don't hunt for a "parameter bug" — there isn't one. Enable node-level Retry On
  Fail (see TG-R6). A single transient failure shouldn't take down the whole workflow and
  lose a lead/notification.
- **Status:** ⚠️ Documented.

### TG-E8. sendPhoto by URL with imgBB: "failed to get HTTP URL content" / "wrong type of the web page content"

- **Symptom.** A Telegram sendPhoto node with `file = {{ image_url }}` (a link to
  `i.ibb.co/.../*.png`) fails: either `Bad Request: failed to get HTTP URL content`
  (Telegram couldn't download it) or `Bad Request: wrong type of the web page content`
  (downloaded the wrong thing). Sporadic: one generation goes through, the next doesn't.
- **Cause.** Delivering the image **by link** offloads the download to Telegram. The imgBB
  CDN is either "cold" right after upload (first error class) or serves content in a way
  Telegram doesn't recognize as an image (second error class). On top of that, photo-by-URL
  is capped at 5 MB — a heavy PNG can exceed it.
- **Fix.** Don't let Telegram pull the URL — download the image in the workflow and send it
  as **binary** (multipart). See TG-R7. Retrying the by-URL approach doesn't fix "wrong
  type" (it's not a cold-CDN issue but the delivery method itself).
- **Source.** Publisher product, "Final Preview: photo" / "Publish to Telegram".
- **Status:** ⚠️ Documented.

### TG-E9. Update from a channel/supergroup → "channel direct messages topic must be specified"

- **Symptom.** A bot built for DMs receives an update with a negative `chat.id`
  (channel/supergroup). The reply branch (sendMessage to that chat) fails: `Bad Request:
  channel direct messages topic must be specified` (a newer Telegram "channel direct
  messages" feature).
- **Cause.** The trigger catches messages from more than just DMs. Subscribing to
  `edited_message` / the bot being added to a group widens the inbound update stream.
  Telegram rejects a reply into a channel without a specified topic.
- **Fix.** Filter incoming updates by chat type. See TG-R8.
- **Source.** Publisher product, "Access denied".
- **Status:** ⚠️ Documented.

### TG-E10. `require('crypto')` is blocked in Code nodes on this instance — HMAC/SHA256 must be pure JS

- **Symptom:** a Code node doing signing/verification (HMAC-SHA256, SHA256 — Login Widget/
  initData validation, proxy URL signing, session tokens) fails on `require('crypto')` (the
  module isn't available in this n8n's Code node sandbox). Any crypto logic relying on the
  standard `crypto` module won't run.
- **Cause:** this instance's settings don't allow built-in Node modules inside Code nodes
  (`NODE_FUNCTION_ALLOW_BUILTIN` doesn't include `crypto`).
- **Fix:** implement SHA-256 and HMAC-SHA256 in **pure JS** (no `require`) and reuse the
  same block across every crypto node (signature validation, token issuance/verification,
  avatar URL signing). In the coaching product, the pure-JS implementation was carried over
  by copying it between the initData workflow and the web workflow (Login Widget + session +
  URL signing).
- **Prevention:** for any new crypto task inside a Code node, immediately reuse the ready-made
  pure-JS implementation from a neighboring workflow rather than writing
  `require('crypto')` "the way you would on a backend." Verify by executing the node (the
  error shows up at runtime, not in config validation).
- **Source:** coaching product, `Auth`/signing nodes (carried across workflows).
- **Status:** ⚠️ Confirmed in production (pure JS works, `require('crypto')` doesn't).

### TG-E11. Signing Mini App initData for testing: compact JSON + `%20`, not `+`

- **Context:** to test a Mini App webhook backend without Telegram (a curl request with a
  forged but validly signed `initData`), the signature is computed as a double
  HMAC-SHA256: `secret = HMAC("WebAppData", bot_token)`,
  `hash = HMAC(secret, data_check_string)`, where `data_check_string` is `k=v` pairs
  (excluding `hash`), sorted, joined with `\n`, with values **URL-decoded**.
- **Gotchas:** (1) serialize the `user` field as **compact** JSON with no spaces
  (`separators=(",",":")`) — otherwise the string won't match what n8n gets after
  decodeURIComponent; (2) when building the query string, encode spaces as `%20`,
  **not `+`** (`urlencode(..., quote_via=quote)`): n8n uses `decodeURIComponent`, which
  doesn't turn `+` into a space → `data_check_string` diverges → hash mismatch → 401.
- **Also:** check the "admin only" gate against the `id` from the **signed**
  `initData.user`, never from the raw payload; `auth_date` must be no older than 24h.
- **Source:** access-control product, Mini App API — 401/200 smoke test.
- **Status:** ⚠️ Confirmed (signed requests → 200; foreign/broken/empty → 401).
