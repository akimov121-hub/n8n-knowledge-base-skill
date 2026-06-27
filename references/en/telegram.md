# Telegram

Telegram nodes: Trigger, sendMessage, sendPhoto, inline buttons, receiving voice messages / video notes.

---

## ✅ Solutions

### TG-R1. Always disable the n8n attribution

- **Context:** by default, Telegram nodes append "This message was sent automatically
  with n8n".
- **Solution:** set `appendAttribution: false` on all Telegram nodes.

Status: ✅ Project rule.

### TG-R2. Default parse_mode for all text nodes — `HTML` + escape `<>&`

- **Context:** in any sendMessage / sendPhoto where the text or caption is built
  dynamically (from an LLM, from a user, from an API), you need safe formatting. Markdown
  (the default) breaks on a single `_ * [ \``; the absence of a parse_mode also yields Markdown
  (see TG-E1).
- **Solution:**
  1. Set `parse_mode: "HTML"` (snake_case!) on ALL Telegram nodes with text or caption.
  2. Anywhere user input or an LLM response can end up in text/caption —
     **always** run it through an escape:
  ```js
  const esc = s => String(s||'').replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;');
  ```
  3. If the LLM itself writes text with HTML markup — explicitly require in the system prompt:
     only `<b>`/`<i>`/`<u>`/`<s>`/`<a href="...">`/`<code>`/`<pre>`/
     `<tg-spoiler>`/`<blockquote>` are allowed; tags must be balanced; in plain text use
     `&amp;`/`&lt;`/`&gt;`.
- **What it covers:** `_` and `*` in `@username`, `@channel_name` won't break the post; an arbitrary
  user-supplied topic `text < 10` after `esc()` goes out as `text &lt; 10`;
  an LLM-generated post with `<b>heading</b>` renders in bold.

Status: ✅ Confirmed in production.

### TG-R3. The "typing…" indicator (sendChatAction) — wire it up so you don't lose data

- **Context:** show the user "typing…" while the bot is thinking (especially before a long
  LLM node).
- **Node setup:** Telegram node, `operation: sendChatAction`, `chatId: ={{ $json.chat_id }}`
  (from the normalizing Set node right before it).
- **Wiring (THE KEY POINT):** the `sendChatAction` node **overwrites the current item** with the Telegram API response
  (see TG-E4). Two working approaches:
  1. All nodes **after** Send Typing pull data not from `$json`, but from an upstream node by name:
     `$('Normalize').first().json.<field>`.
  2. Or place Send Typing in a **side (dead-end) branch**, so the main data flow
     doesn't pass through it.
- **Note (Telegram API behavior):** the "typing…" status lasts ~5 seconds and clears
  on its own. For long responses you'd have to resend it repeatedly.

Status: ✅ Confirmed.

### TG-R4. Inline buttons and forceReply in the Telegram node (sendMessage)

- **Context:** attach an inline keyboard (callback or URL buttons) or request a reply
  via ForceReply.
- **Solution:**
  - Inline keyboard: `parameters.replyMarkup = "inlineKeyboard"`, then
    `parameters.inlineKeyboard.rows = [{ "row": { "buttons": [ ... ] } }]`. Each button:
    `{ "text": "...", "additionalFields": { ... } }`.
    - callback button: `additionalFields.callback_data` (**snake_case!**).
    - URL button (including a deep-link `t.me/<bot>?start=<payload>`): `additionalFields.url`.
  - ForceReply: `parameters.replyMarkup = "forceReply"`,
    `parameters.forceReply = { "force_reply": true }`.
  - camelCase (`callbackData`, `forceReply:true`) is silently swallowed by n8n → the button won't appear
    (DEP-E4). After the PUT — do a GET to verify the keys are in place.
  - answerCallbackQuery: `resource:"callback"`, `operation:"answerQuery"`, `queryId` from
    `$('<node with the update>').first().json.<cb_id>`.

Status: ✅ Structure verified via GET (camelCase is dropped, snake_case is preserved).

### TG-R4*. Approval loop (preview with buttons + edits by text/voice) via a Postgres state machine

**Context.** You need a human-in-the-loop: the bot shows the result, the owner taps buttons
(approve/revise), and edits can be sent by text OR voice message. The "Send and Wait" node is not
suitable: it accepts free text through an n8n web form, not via a chat message, and doesn't accept
voice at all.

**Solution** (✅ confirmed in production):

1. **State — a table in Postgres** (`<bot>_drafts`, key `chat_id`, a `stage` column +
   draft data). Each message/button = a new workflow run; the stage determines the
   route. It survives n8n restarts, the wait is indefinite, and executions are short.
2. **Trigger updates = `["message","callback_query"]`**. Check Access and chatId — via
   `COALESCE`: `message?.from?.id || callback_query?.from?.id`.
3. **Always load state as exactly 1 row** (otherwise a branch with no items won't fire):
   `SELECT COALESCE(d.stage,'') ... FROM (SELECT 1) x LEFT JOIN <bot>_drafts d ON d.chat_id = {{ Number(...) }}`.
4. **Router (Switch, loose v2):** first the buttons' callback_data (string equals), then
   `/start`, then the input-waiting stages (regex on stage + presence of a message), then a new
   request (text/caption not empty), fallback — a polite refusal.
5. **Buttons** — a regular sendMessage with `replyMarkup: inlineKeyboard`. On each callback branch
   the first node must be `resource: callback` (answerQuery, `queryId = callback_query.id`), otherwise the
   button keeps spinning.
6. **Voice only at the wait points:** IF `!!message?.voice` → Telegram `resource: file`
   (voice.file_id) → openAi `resource: audio, transcribe, language: ru` → shared Code "Input
   text" → Switch by stage. Outside a stage, voice is rejected.
7. **Text values in SQL** — as an inline literal with quote escaping:
   `'{{ expr.replace(/'/g, "''") }}'` (queryReplacement breaks on commas in the text). All
   UPDATEs use `RETURNING *`.
8. Completion — not a DELETE, but `stage='published'`: the next request overwrites the row via
   an UPSERT.

### TG-R5. Accepting video notes (video_note) alongside voice messages

**Context.** The bot must accept not only voice but also video notes (video_note) and
use them as voice input (transcribing the audio track).

**Solution** (✅ confirmed in production):

1. **Detect** — in the IF/Switch check both types:
   `!!($json.message?.voice || $json.message?.video_note)`.
2. **Download** — Telegram node `resource: file`,
   `fileId = {{ message.voice?.file_id || message.video_note?.file_id }}` (one node for both).
3. **Transcribe** — openAi `resource: audio, operation: transcribe, language: ru` accepts
   the mp4 video note **directly**, no need to extract the audio track separately.
4. Then — the shared "Input text" code, same as for voice (TG-R4* step 6).

**Protecting the draft against free-form voice/note.** If a voice message/video note can arrive both as a "new
intent" (with no active draft) and as an edit (in a wait stage) — add a second condition by stage to the
"new intent" rule: fire only when there is no draft (`stage`
empty/`published`/`cancelled`). Otherwise a stray voice message while a preview is ready will overwrite the draft.

### TG-R6. Retry On Fail on Telegram sending nodes (protection against transient failures)

**Context.** External Telegram calls (sendMessage / sendPhoto / sendChatAction / getFile)
occasionally fail transiently — for example, a sporadic "chat not found" with a valid config
(TG-E7) or a network blip. One such failure crashes the whole run and loses the message / notification /
lead.

**Solution.** On Telegram sending and downloading nodes, enable **node-level** retry:
`retryOnFail: true`, `maxTries: 3`, `waitBetweenTries: 5000` (these are fields on the node itself, not in
`parameters`).

**⚠️ Exception — public publishing.** Retry gives **at-least-once** semantics: if Telegram
delivered the message but the response didn't reach n8n (a break exactly between delivery and response), a retry
will create a **duplicate**. For an ack / replies to the user / notifications to the owner / file downloads a duplicate
is tolerable. For nodes that post to a **public channel/feed** (e.g. the "Publish to Telegram" node),
do **NOT** enable retry — a duplicate = a double public post.

**Deploy.** Set the three fields on the node, PUT, then GET to verify `retryOnFail` was
saved. The credential bindings are not touched in the process (on the current instance, GET returns `credentials` —
it's enough to pass them through unchanged).

**Source.** Rolling retry out across 8 active workflows (52 Telegram sending nodes; public
publishing excluded). The original case — TG-E7.

**Status:** ✅ Confirmed in production.

### TG-R7. Send the image to Telegram as binary (download → sendPhoto binaryData), not by-URL

**Context.** The image is stored at a public URL (imgBB — needed for the Instagram Graph API).
For Telegram, delivery by link is unreliable (TG-E8) and capped at 5 MB.

**Solution.** Before the Telegram node, insert an HTTP Request that downloads the URL into a binary, and
send sendPhoto as binary:
- HTTP Request (GET, `url={{ image_url }}`),
  `options.response.response = { responseFormat: "file", outputPropertyName: "data" }`;
  `retryOnFail` on the download is fine (idempotent GET).
- Telegram sendPhoto: `binaryData: true`, `binaryPropertyName: "data"`, do NOT set the `file` field.
  Take `caption`/`parse_mode` from a named node
  (`$('Load draft').first().json.body_text`), because after the HTTP node `$json` holds the binary,
  not the text.

This removes both error classes of TG-E8 and the 5 MB limit (a binary photo can go up to 10 MB). imgBB stays for
Instagram — the Graph API pulls the URL on its side.

**⚠️ Do NOT set retry on the actual channel-publishing sendPhoto** (duplicate, TG-R6) — retry is allowed
only on the download node.

**Source.** A publishing product, "Download cover" → "Final preview: photo" and
"Download cover (publish)" → "Publish to Telegram".

**Status:** ✅ Confirmed in production.

### TG-R8. A "private chat only" filter at the bot's entry

**Context.** A bot for personal use (the owner in a DM). Updates from channels/groups are junk
and a source of errors (TG-E9).

**Solution.** Right after the Telegram Trigger (before the access check and any replies) — an IF on the chat
type:
```
chat_type = {{ ($('Telegram Trigger').item.json.message || $('Telegram Trigger').item.json.edited_message || $('Telegram Trigger').item.json.callback_query?.message || {}).chat?.type || '' }}
```
`chat_type === 'private'` → continue; otherwise → NoOp (silently ignore, no reply). It covers
message, edited_message and callback_query at once.

**Source.** A publishing product, "Private chat?".

**Status:** ✅ Confirmed in production.

### TG-R9. Reacting to an edit of the user's own message (edited_message) as new verbatim input

**Context.** You want to let the user edit already-sent text not with a new message, but by
editing their own (the "fixed it in place" UX).

**Solution.**
1. In the Telegram Trigger, add the `edited_message` update type (alongside `message`,
   `callback_query`).
2. Everywhere incoming data is read, support both branches: `message?.… || edited_message?.…`
   (chat.id, from.id, text — in CHAT/CHATNUM, Check Access, save nodes).
3. In the router: the rule "there is an `edited_message` AND stage = `<manual input mode>`" →
   save `edited_message.text` verbatim → show the preview. Any other `edited_message`
   (not in the right stage) → silently ignore (otherwise editing an old message falsely triggers it).
4. Limitation: the bot can catch edits only to the user's **own** messages; the first input
   still arrives as a regular message, and after that — edits. Telegram delivers edited_message
   only for ~48h.

**Source.** A publishing product, "Save texts (message edit)" + the
`edit_text_manual` rule.

**Status:** ✅ Confirmed in production.

### TG-R10. Graceful handling of broken user HTML in send (onError → a hint)

**Context.** If the text to send is produced by the user (manual input, their own ready-made post),
its HTML may be invalid (`<b>…<b>` instead of `</b>`). sendMessage/sendPhoto with
`parse_mode: HTML` fails on that (`can't find end tag …`) — the node goes red, the user gets no
feedback.

**Solution.** On the preview node (which is the first to send the user's text with parse_mode HTML)
set `onError: continueErrorOutput`. The error output (index 1) → a separate static
hint node (with no tags in the text, so it doesn't fail itself): "broken markup, the closing tag is
`&lt;/b&gt;`, fix it and send again". The preview acts as a "gate": as long as the markup is broken,
publishing is unreachable, and the user sees a clear reason. Keep the stage in "verbatim" mode,
so the re-sent text stays verbatim.

**Source.** A publishing product, "Text preview" (error output) → "Preview: broken markup".

**Status:** ✅ Confirmed in production.

### TG-R11. A dynamic card list with inline buttons (id in callback_data via an expression)

- **Context:** you need to show the user a list of records from the DB (a post queue, tasks),
  each with its own action buttons.
- **Solution:** don't build one shared keyboard, but send **a card per item**:
  the Postgres node returns N rows → the Telegram node runs per item. In the buttons' callback_data —
  an expression with the record id: `"=q_now_{{ $json.id }}"`. The callback handler in the router catches
  it by regex (`^q_now_`), and extracts the id with `split('_')[2]`.
- **Pros:** no fiddling with dynamically assembling the reply_markup JSON; each card is
  self-contained; actions are atomic by id (UPDATE ... WHERE id AND status — protection against
  stale buttons).
- **Empty list:** Postgres returns `{success:true}` on 0 rows (PG-E2) — an IF guard
  on a key field is required before the card, otherwise a junk card goes out.
- **Source:** a publishing product, the /queue command (queue cards with 🚀/🕒/❌).
- **Status:** ✅ Confirmed in production.

### TG-R12. The bot's menu button = a Web App (mini app) instead of a command list

- **Task:** make the bot's menu button (bottom-left in the chat) open a Mini App rather than showing a
  command list.
- **Solution (Bot API, without BotFather):** `POST https://api.telegram.org/bot<token>/setChatMenuButton`
  with body `{"menu_button":{"type":"web_app","text":"Open","web_app":{"url":"https://<app-url>/"}}}`.
  Without `chat_id` — the default for everyone. The URL must be HTTPS. To revert:
  `{"menu_button":{"type":"commands"}}`.
- **⚠️ Readback gotcha:** right after `set` (ok:true), `getChatMenuButton` may still return
  the old value (`{type:'commands'}`) — a **Telegram propagation delay**, not an error. Re-check
  a minute later: the default reads back as `{type:'web_app',...}`, and for a specific chat —
  `{type:'default'}` (inherits the default). Don't "fix" it with repeated `set`s based on the first read.
- **Token:** take it from `*_config`/the execution server-side, don't print it to the chat/logs.
- **Source:** the publishing bot.
- **Status:** ✅ Confirmed in production (the button opens the app).

### TG-R13. Forward / link → Mini App inbox: detect and gate before the router

- **Task:** the bridge bot must put a forwarded news item into `pub_inbox` (→ the "From
  forwards" block in the Mini App), not handle it in the chat.
- **Gotcha 1 — detecting only by `forward_date`.** The "Forwarded?" node checked
  `!!message.forward_date`. But when the user **shares a link** (share-sheet / pasted URL),
  Telegram sends it as **a plain message with no forward field at all** (`message` =
  message_id/from/chat/date/text/entities — neither `forward_date` nor `forward_origin`). So a
  link-news item wasn't recognized as "forwarded". Fix: broaden the condition —
  `!!m.forward_date || /^https?:\/\//i.test((m.text||m.caption||'').trim())`. (Also keep in mind:
  Telegram is gradually replacing legacy `forward_from*`/`forward_date` with `forward_origin` — for
  real forwards check both.)
- **Gotcha 2 — the detect node ran IN PARALLEL with the router.**
  `Load draft → [Router, Forwarded?]` simultaneously: even a recognized
  forward still went into the router too → with an active draft the URL was "eaten" as a text
  edit. Fix: make the detect a **gate BEFORE the router** —
  `Load draft → Forwarded?`; true → inbox; **false (output 1) → Router**. The IF
  passes the same item through, so the router gets the same data. Now forwards/links go only into
  the inbox.
- **Gotcha 3 — a silent write.** After the INSERT into `pub_inbox` the bot stayed silent → "nothing
  happened". Add a confirmation reply ("🔗 Added to the app → open it, the "From forwards" block").
- **The inbox node is resilient to missing forward fields:** build `src` with fallbacks
  (`forward_origin.chat.title || sender_user || forward_from... || ''` → "Repost"), `text` =
  `message.text||caption`. For a bare link, src="Repost", text=URL — it doesn't break.
- **Source:** the publishing bot (`<workflow_id>`).
- **Status:** ✅ Confirmed in production.

### TG-R14. Web app login via the Telegram Login Widget → a session HMAC token (secret=SHA256(token), ≠ initData)

- **Task:** authenticate the user in the **browser** web version of a Telegram product
  (not the Mini App) on the n8n backend, then reuse the same data endpoints as in the Mini App.
- **The key difference in the signature from the Mini App.** Telegram provides **two different** schemes, which are easy
  to mix up:
  - **Login Widget** (the "Log in with Telegram" widget on a site): `secret = SHA256(bot_token)`
    (binary digest), `data_check_string` = all widget fields **except `hash`**, sorted
    by key, `key=value`, joined with `\n`; `check = HMAC_SHA256(dcs, secret)` in hex == `hash`.
    Additionally `auth_date` ≤ 86400 s (anti-replay).
  - **Mini App initData** (see TG-E11): `secret = HMAC_SHA256("WebAppData", bot_token)`.
    **Not interchangeable.**
- **A session instead of repeated validation.** The widget signature is valid once (the widget is not
  sent on every request). After a successful `auth/telegram` the backend issues a
  **self-contained session token** without a server-side session table:
  `base64url(JSON{telegram_id,exp}) + "." + base64url(HMAC_SHA256(payload, SESSION_SECRET))`.
  The lifetime (`exp`) — e.g. 30 days. After that every request carries the token; the backend checks the signature +
  `exp` and takes `user_id` **from the token, never from the payload** (protection against spoofing).
- **`SESSION_SECRET` — in the DB config** (the `*_config` table, a separate key), read by the
  `Load Config` node at the start of the workflow; it never reaches code/responses/logs. One secret per instance,
  generated once.
- **Reusing the queries.** After the token gate, the same Postgres queries as in the Mini
  App (profile, facts, history, journal, rewards) serve the web — isolation is achieved by
  **duplicating the SQL in a separate workflow** (`<workflow_name>`), with no shared nodes with
  the miniapp/main bot.
- **Access gate** — the shared `bot_access(bot='<slug>', active=true)` by `telegram_id` from
  the verified signature; no access → no token is issued (an empty string), the frontend shows "no
  access".
- **Source:** a coaching product, `<workflow_name>` (webhook `<webhook_path>`, id `<workflow_id>`).
- **Status:** ✅ Confirmed in production (valid hash → token; forged/expired → 401).

### TG-R15. A secure Telegram avatar proxy — a signed URL, the bot token never on the client

- **Task:** show the user's real Telegram avatar in web/Mini App, **without
  exposing the bot token** to the client (the Telegram file URL contains the token in the path).
- **Solution — a separate proxy workflow with a signed URL:**
  1. Endpoint `GET /webhook/<webhook_path>?uid=<telegram_id>&sig=<signature>`, where
     `sig = HMAC_SHA256(String(uid), AVATAR_SECRET)` in hex. An invalid/missing signature →
     **403**. The signature is generated by the main API when it issues `photo_url` — the client can't forge
     `uid`.
  2. On a valid signature: Bot API `getUserProfilePhotos(user_id=uid, limit=1)` → take the largest
     size (the last element of `photos[0]`) → `file_id` → `getFile` → `file_path` →
     download `https://api.telegram.org/file/bot<token>/<file_path>` (HTTP node, response=file)
     → **return the bytes** (`image/jpeg`).
  3. Cache: response with `Cache-Control: public, max-age=86400`. No photo / privacy / Bot
     API error → **404** (the frontend shows a placeholder via `onerror`).
- **Security:** `bot_token` and `AVATAR_SECRET` — from `*_config` (the `Load Config` node), they never
  reach the response/redirect/headers/logs. The full download happens on the n8n side → only the image
  bytes leave, not a URL with the token. HTTP nodes to the Bot API: `retryOnFail` 3×5s,
  `getUserProfilePhotos`/`getFile` with `onError: continueRegularOutput` (graceful degradation to 404).
- **Consistency:** in the Mini App also return the proxy URL, not `initData.photo_url` — one
  code path on the frontend + always a fresh, cacheable image.
- **Source:** a coaching product, `<workflow_name>` (webhook `<webhook_path>`, GET, id
  `<workflow_id>`).
- **Status:** ✅ Confirmed in production (valid signature → 200 image/jpeg 640×640; broken → 403;
  no token in the response).

### TG-R16. Channel mirroring — an outbound sendMessage from the web branch into the Telegram dialog doesn't create a loop

- **Task:** the user's conversation with the agent **from the website** should also be visible in their
  Telegram chat with the bot (a single dialog across two channels).
- **Solution:** after the web branch has built the reply, **in parallel** with the web response, send two
  `sendMessage` calls from the bot to Telegram: an echo of the user's message (with a channel tag, e.g.
  `🌐 <Name> (from the site):\n<text>`) + the agent's reply. The name — from `profile.name` by `telegram_id` (with the
  same query that loads the prompt/profile — without a separate node).
- **Why there's no loop:** an outbound `sendMessage` from the **bot** does not produce an incoming update in the Telegram
  Trigger (the bot doesn't "hear" itself) → the main workflow doesn't run. Mirroring is
  safely one-way.
- **Reliability (important):** the mirror branch runs **in parallel** with the web-response node
  (`Respond to Webhook` answers first and independently). The Telegram HTTP nodes use
  `onError: continueRegularOutput` + `retryOnFail` 3×5s: **a Telegram failure doesn't break the web response** and
  doesn't crash the execution. The dialog memory is **not duplicated** — it's written by the agent's memory node on the
  main pass; the mirror only sends to Telegram.
- **Formatting is reused** from the main bot (`stripLeak` + converting the sentinel
  `⟪PROGRAM⟫`→`<blockquote expandable>`, `parse_mode=HTML` only when a block is present). Take the
  **raw agent output**, not the one already "cleaned" for the web (if the cleaning strips the sentinels). No
  token/`user_id` → the node returns `[]` (the mirror is skipped).
- **Source:** a coaching product, `<workflow_name>` the `chat/send` branch
  (`Web Mirror Format`/`Mirror Echo`/`Mirror Reply`).
- **Status:** ✅ Accepted in production (the logic is verified; actual delivery — from a live web session, on the
  owner).

---

## ⚠️ Errors

### TG-E1. `parse_mode: ""` does not disable Markdown

- **Symptom:** you want to send "raw" text, you set `parse_mode: ""` — and Telegram still
  parses Markdown, the text arrives with markup / breaks on special characters.
- **Cause:** an empty string is not interpreted as "no formatting".
- **Fix:** set `parse_mode: "HTML"` (not an empty string and not Markdown). Uncovered over
  4 pings through the sandbox workbench (see debugging-sandbox.md DBG-R1).

### TG-E2. sendPhoto — the parameter is `file`, not `photo`; parse_mode in snake_case

- **Symptom:** you configure sending a photo via the API, the photo doesn't go out / the parameter is ignored.
- **Cause:** in the n8n Telegram node, the parameter for the photo is named `file`, not `photo`. Names are
  case-sensitive: `parse_mode` is written in snake_case. An incorrect name is silently dropped by n8n
  (see rest-api-deploy.md DEP-E4).
- **Fix:** use `file`; parse_mode — snake_case; do a GET to verify after the PUT.

### TG-E3. The "sent automatically with n8n" attribution in messages

- **Symptom:** the bot's messages have an n8n note at the bottom.
- **Cause:** `appendAttribution: false` is not set.
- **Fix:** see TG-R1.

### TG-E4. After sendChatAction ("Send Typing") downstream loses chat_id/message_id/text

- **Symptom:** after the Telegram `sendChatAction` node, the following nodes get `undefined`
  instead of `chat_id` / `message_id` / `text`. DB queries write null, replies go to the wrong place.
- **Cause:** the Telegram node (like most n8n action nodes) **replaces the current item with its
  API response** — after it, `$json` is roughly `{ "result": true }`, not the data from the
  normalizing node.
- **Fix:** in nodes after Send Typing, reference upstream by name —
  `$('Normalize').first().json.<field>`, not `$json.<field>`. Or move Send Typing into a
  side branch. The full pattern — TG-R3.

### TG-E5. A `{{ }}` substitution in a node's text comes through as raw text

- **Symptom:** in the bot's message, you literally see `{{ $json.foo }}` instead of the value. Worse —
  it breaks downstream logic that looks for the substituted marker.
- **Cause:** n8n treats a string field as an **expression** only if it starts with `=`.
  Text with `{{ }}` without a leading `=` goes out as a literal. Easy to forget on static texts where
  the substitution is only in the middle.
- **Fix:** any field (`text`, `chatId`, `callback_data`, `url`) containing `{{` must
  start with `=`. In the generator — a helper:
  `'=' + v if '{{' in v and not v.startswith('=') else v`. Do a GET to verify after the PUT.

### TG-E6. "Bad Request: chat not found" on sendMessage + the bot-selection rule

- **Symptom.** The Telegram sendMessage node fails with `Bad request - please check your parameters`,
  with `Bad Request: chat not found` in the `description`. The credential is valid (no
  "Credential … does not exist" error).
- **Cause.** The bot the credential belongs to physically cannot message this `chatId`,
  because the user **never tapped Start / never wrote to this bot**. A private chat with a bot
  arises only after the person writes to the bot first. This is NOT a parameter error —
  Telegram simply doesn't know such a chat for this bot. Each bot has its own separate private
  chat.
- **Fix.** Open the right bot in Telegram and send it any message (Start). After
  that, sendMessage to this chatId goes through. You must launch exactly the bot whose credential
  is set in the node.
- **Related rule.** During a build, Claude on its own initiative connected someone else's bot
  (a consultant bot) to notifications — that's not allowed. **Before connecting ANY bot/credential to
  a workflow — always ask which bot exactly to use. Don't pick by guesswork and don't
  reuse someone else's bot.** One bot = one Telegram Trigger (webhook): reusing the
  trigger bot in another scenario breaks the original.
- **Status:** ⚠️ Recorded.

### TG-E7. A sporadic "chat not found" with a fully valid config (a Telegram transient)

- **Symptom.** sendMessage fails with `Bad Request: chat not found`, but the config is fine and **the same
  bot writes successfully to the same chat in neighboring runs**. Different from TG-E6: there the bot wasn't
  started by the user — it fails deterministically every time; here the failure is one-off.
- **Sign of a transient.** 2 of 3 identical runs (same `chatId`, same bot, same node,
  differing only in the inline button's URL, which Telegram doesn't check for existence) — successful,
  one failed.
- **Fix.** Don't go looking for a "parameter bug" — there isn't one. Enable node-level Retry On Fail (see
  TG-R6). One transient failure shouldn't crash the whole workflow and lose a lead/notification.
- **Status:** ⚠️ Recorded.

### TG-E8. sendPhoto by-URL with imgBB: "failed to get HTTP URL content" / "wrong type of the web page content"

- **Symptom.** The Telegram sendPhoto node with `file = {{ image_url }}` (a link to `i.ibb.co/.../*.png`)
  fails: either `Bad Request: failed to get HTTP URL content` (Telegram couldn't download), or
  `Bad Request: wrong type of the web page content` (downloaded the wrong thing). Sporadically: one generation
  goes through, the next doesn't.
- **Cause.** Delivering the image **by link** offloads the download to Telegram. The imgBB CDN
  is "cold" right after upload (yields the first error class) or serves the content in a way that
  Telegram doesn't recognize as an image type (the second class). Plus photo-by-URL has a 5 MB limit — a heavy
  PNG breaks it.
- **Fix.** Don't let Telegram pull the URL — download the image in the workflow and send it as **binary**
  (multipart). See TG-R7. Retry by-URL doesn't fix "wrong type" (it's not a CDN warm-up, but the delivery
  method itself).
- **Source.** A publishing product, "Final preview: photo" / "Publish to Telegram".
- **Status:** ⚠️ Recorded.

### TG-E9. An update from a channel/supergroup → "channel direct messages topic must be specified"

- **Symptom.** A bot meant for DMs receives an update with a negative `chat.id`
  (channel/supergroup). The reply branch (sendMessage to this chat) fails: `Bad Request: channel
  direct messages topic must be specified` (Telegram's new "channel direct messages" feature).
- **Cause.** The trigger catches messages from more than just DMs. Subscribing to `edited_message`/
  the bot being in a group widens the incoming stream. Telegram rejects a reply to a channel without a topic
  specified.
- **Fix.** Filter the input by chat type. See TG-R8.
- **Source.** A publishing product, "Access denied".
- **Status:** ⚠️ Recorded.

### TG-E10. `require('crypto')` is forbidden in this instance's Code nodes — HMAC/SHA256 only in pure JS

- **Symptom:** a Code node doing a signature/verification (HMAC-SHA256, SHA256 — validating the Login
  Widget/initData, signing the proxy URL, the session token) fails on `require('crypto')` (the module
  isn't available in this n8n's Code node sandbox). Any crypto logic via the standard `crypto`
  won't run.
- **Cause:** in this instance's settings, built-in Node modules aren't allowed in Code nodes
  (`NODE_FUNCTION_ALLOW_BUILTIN` doesn't include `crypto`).
- **Fix:** implement SHA-256 and HMAC-SHA256 in **pure JS** (no `require`) and reuse
  the same block in all crypto nodes (signature validation, token issuance/verification, signing the
  avatar URL). In the coaching product, the pure-JS implementation was carried between the workflow (initData)
  and the web workflow (Login Widget + session + URL signing) by copying.
- **Prevention:** for any new crypto task in a Code node — immediately take a ready pure-JS
  implementation from a neighboring workflow, don't write `require('crypto')` "like on a backend". Check at
  execution (the error shows up at runtime, not in the config).
- **Source:** a coaching product, the `Auth`/signing nodes (carried between workflows).
- **Status:** ⚠️ Confirmed in production (pure JS works, `require('crypto')` doesn't).

### TG-E11. Signing Mini App initData for a test: compact JSON + `%20`, not `+`

- **Context:** to test the Mini App webhook backend without Telegram (curl with a forged but
  validly signed `initData`), the signature is computed as a double HMAC-SHA256:
  `secret = HMAC("WebAppData", bot_token)`, `hash = HMAC(secret, data_check_string)`, where
  `data_check_string` — `k=v` pairs (without `hash`), sorted, joined with `\n`, values **in
  URL-decoded form**.
- **Gotchas:** (1) serialize the `user` field as **compact** JSON with no spaces
  (`separators=(",",":")`) — otherwise the string won't match what n8n gets after
  decodeURIComponent; (2) when building the query, encode spaces as `%20`, **not `+`**
  (`urlencode(..., quote_via=quote)`): n8n uses `decodeURIComponent`, which doesn't turn `+` into a space
  → the `data_check_string` diverges → hash mismatch → 401.
- **Also:** the "admin only" gate must check the `id` from the **signed** `initData.user`, not from the
  payload; `auth_date` no older than 24h.
- **Source:** an access-control product, the Mini App API — the 401/200 smoke test.
- **Status:** ⚠️ Confirmed (signed requests → 200, foreign/broken/empty → 401).
