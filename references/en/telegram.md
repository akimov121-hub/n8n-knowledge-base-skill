# Telegram

Telegram nodes: Trigger, sendMessage, sendPhoto, inline buttons, receiving voice messages/video notes.

---

## ✅ Solutions

### TG-R1. Always disable the n8n attribution

- **Context:** by default, Telegram nodes append "This message was sent automatically
  with n8n".
- **Solution:** set `appendAttribution: false` on all Telegram nodes.

Status: ✅ Project rule.

### TG-R2. Default `parse_mode` for all text nodes — `HTML` + escaping `<>&`

- **Context:** any sendMessage / sendPhoto where the text or caption is built dynamically
  (from an LLM, from the user, from an API) needs safe formatting. Markdown (the default)
  breaks on stray `_ * [ \``; leaving `parse_mode` unset also defaults to Markdown
  (see TG-E1).
- **Solution:**
  1. Set `parse_mode: "HTML"` (snake_case!) on ALL Telegram nodes with text or a caption.
  2. Anywhere user input or an LLM reply can end up in text/caption — **always** run it
     through an escape function:
  ```js
  const esc = s => String(s||'').replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;');
  ```
  3. If the LLM itself writes text with HTML markup — explicitly require in the system
     prompt: only `<b>`/`<i>`/`<u>`/`<s>`/`<a href="...">`/`<code>`/`<pre>`/
     `<tg-spoiler>`/`<blockquote>` are allowed; tags must be balanced; in plain text —
     `&amp;`/`&lt;`/`&gt;`.
- **What this covers:** `_` and `*` in `@username`, `@channel_name` won't break a post; an
  arbitrary user-supplied topic like `what < 10` becomes `what &lt; 10` after `esc()`;
  an LLM-generated post with `<b>heading</b>` renders bold.

Status: ✅ Confirmed in production.

### TG-R3. "Typing…" indicator (sendChatAction) — set it up without losing data

- **Context:** show the user "typing…" while the bot is thinking (especially before a
  long LLM node).
- **Node setup:** Telegram node, `operation: sendChatAction`, `chatId: ={{ $json.chat_id }}`
  (from a normalizing Set node placed right before it).
- **Wiring (KEY POINT):** the `sendChatAction` node **overwrites the current item** with
  the Telegram API response (see TG-E4). Two working approaches:
  1. All nodes **after** Send Typing read data not from `$json` but from the upstream node
     by name: `$('Normalize').first().json.<field>`.
  2. Or place Send Typing on a **side (dead-end) branch** so the main data flow doesn't
     pass through it.
- **Note (Telegram API behavior):** the "typing…" status lasts ~5 seconds and clears
  itself. For long responses you'd need to resend it repeatedly.

Status: ✅ Confirmed.

### TG-R4. Inline buttons and forceReply in the Telegram node (sendMessage)

- **Context:** attach an inline keyboard (callback or URL buttons) or request a reply via
  ForceReply.
- **Solution:**
  - Inline keyboard: `parameters.replyMarkup = "inlineKeyboard"`, then
    `parameters.inlineKeyboard.rows = [{ "row": { "buttons": [ ... ] } }]`. Each button:
    `{ "text": "...", "additionalFields": { ... } }`.
    - callback button: `additionalFields.callback_data` (**snake_case!**).
    - URL button (including a `t.me/<bot>?start=<payload>` deep link): `additionalFields.url`.
  - ForceReply: `parameters.replyMarkup = "forceReply"`,
    `parameters.forceReply = { "force_reply": true }`.
  - camelCase (`callbackData`, `forceReply:true`) is silently dropped by n8n → the button
    won't appear (DEP-E4). After a PUT — verify with a GET that the keys are intact.
  - answerCallbackQuery: `resource:"callback"`, `operation:"answerQuery"`, `queryId` from
    `$('<node with the update>').first().json.<cb_id>`.

Status: ✅ Structure verified via GET (camelCase is dropped, snake_case is preserved).

### TG-R4*. Approval loop (preview with buttons + edits via text/voice) via a Postgres state machine

**Context.** Need human-in-the-loop: the bot shows a result, the owner presses buttons
(approve/revise), and edits can arrive as text OR voice. The "Send and Wait" node doesn't
fit: it accepts free text only via an n8n web form, not as a chat message, and doesn't
accept voice at all.

**Solution** (✅ confirmed in production):

1. **State lives in a Postgres table** (`<bot>_drafts`, keyed by `chat_id`, plus a `stage`
   column and the draft data). Each message/button press = a new workflow run; the stage
   determines the route. Survives n8n restarts, waiting is indefinite, executions are
   short.
2. **Trigger updates = `["message","callback_query"]`**. Check Access and chatId — via
   `COALESCE`: `message?.from?.id || callback_query?.from?.id`.
3. **Always load state as a single row** (otherwise a branch with no items won't fire):
   `SELECT COALESCE(d.stage,'') ... FROM (SELECT 1) x LEFT JOIN <bot>_drafts d ON d.chat_id = {{ Number(...) }}`.
4. **Router (Switch, loose v2)**: first the buttons' callback_data (string equals), then
   `/start`, then the input-waiting stages (regex on stage + presence of a message), then a
   new request (text/caption not empty), fallback — a polite refusal.
5. **Buttons** — a plain sendMessage with `replyMarkup: inlineKeyboard`. Every callback
   branch starts with a `resource: callback` node (answerQuery, `queryId = callback_query.id`),
   otherwise the button keeps its spinner.
6. **Voice only accepted at waiting points**: IF `!!message?.voice` → Telegram `resource: file`
   (voice.file_id) → openAi `resource: audio, transcribe, language: ru` → a shared "Input
   text" Code node → Switch on stage. Outside the waiting stages, voice is rejected.
7. **Text values in SQL** — as an inline literal with quotes escaped:
   `'{{ expr.replace(/'/g, "''") }}'` (queryReplacement breaks on commas inside the text).
   All UPDATEs use `RETURNING *`.
8. Completion is not a DELETE but `stage='published'`: the next request overwrites the row
   via UPSERT.

### TG-R5. Accepting video notes (video_note) the same way as voice messages

**Context.** The bot must accept not only voice messages but also video notes
(video_note), and treat them as voice input (transcribe the audio track).

**Solution** (✅ confirmed in production):

1. **Detection** — check both types in an IF/Switch:
   `!!($json.message?.voice || $json.message?.video_note)`.
2. **Download** — Telegram node `resource: file`,
   `fileId = {{ message.voice?.file_id || message.video_note?.file_id }}` (one node covers both).
3. **Transcription** — openAi `resource: audio, operation: transcribe, language: ru` accepts
   the mp4 video note **directly** — no need to extract the audio track separately.
4. From there — the shared "Input text" code, same as for voice (TG-R4*, step 6).

**Protecting the draft against unsolicited voice/video notes.** If a voice message or video
note can arrive both as a "new intent" (no active draft) and as an edit (in a waiting
stage) — add a second condition on stage to the "new intent" rule: fire only when there is
no draft (`stage` empty/`published`/`cancelled`). Otherwise a stray voice message while a
preview is ready will clobber the draft.

### TG-R6. Retry On Fail on outgoing Telegram nodes (protection against transient failures)

**Context.** Outgoing Telegram calls (sendMessage / sendPhoto / sendChatAction / getFile)
occasionally fail transiently — for example a sporadic "chat not found" with a valid
config (TG-E7), or a network blip. A single such failure kills the whole run and loses a
message / notification / lead.

**Solution.** Enable **node-level** retry on outgoing/downloading Telegram nodes:
`retryOnFail: true`, `maxTries: 3`, `waitBetweenTries: 5000` (these are fields on the node
itself, not inside `parameters`).

**⚠️ Exception — public publishing.** Retry gives **at-least-once** semantics: if Telegram
delivered the message but the response never reached n8n (a disconnect exactly between
delivery and the acknowledgment), a retry creates a **duplicate**. For an ack / a reply to
the user / a notification to the owner / file downloads, a duplicate is tolerable. For
nodes that post to a **public channel/feed** (e.g. the "Publish to Telegram" node), do
**NOT** enable retry — a duplicate there means a double public post.

**Deployment.** Set the three fields on the node, PUT, then verify with a GET that
`retryOnFail` was saved. Credential bindings are untouched in this process (on the current
instance, GET returns `credentials` — it's enough to pass them through unchanged).

**Source.** Rolled out retry across 8 active workflows (52 outgoing Telegram nodes; public
publishing excluded). The original case was TG-E7.

**Status:** ✅ Confirmed in production.

### TG-R7. Send images to Telegram as binary (download → sendPhoto binaryData), not by URL

**Context.** The image is stored at a public URL (imgBB — needed for the Instagram Graph
API). Delivering it to Telegram by link is unreliable (TG-E8) and capped at 5 MB.

**Solution.** Before the Telegram node, insert an HTTP Request that downloads the URL into
binary data, and have sendPhoto send the binary:
- HTTP Request (GET, `url={{ image_url }}`),
  `options.response.response = { responseFormat: "file", outputPropertyName: "data" }`;
  `retryOnFail` on the download is fine (idempotent GET).
- Telegram sendPhoto: `binaryData: true`, `binaryPropertyName: "data"`, leave the `file`
  field unset. Take `caption`/`parse_mode` from a named node
  (`$('Load draft').first().json.body_text`), since after the HTTP node `$json` holds
  binary data, not text.

This removes both classes of TG-E8 errors and the 5 MB limit (binary photo up to 10 MB).
imgBB stays in use for Instagram — the Graph API pulls the URL server-side.

**⚠️ Do not add retry on the sendPhoto node that publishes to a channel** (duplicate,
TG-R6) — retry is only appropriate on the download node.

**Source.** A publishing product, "Download cover" → "Final preview: photo" and
"Download cover (publish)" → "Publish to Telegram".

**Status:** ✅ Confirmed in production.

### TG-R8. "Private chat only" filter at the bot's entry point

**Context.** A bot for personal use (owner in DM). Updates from channels/groups are noise
and a source of errors (TG-E9).

**Solution.** Right after the Telegram Trigger (before the access check and any replies) —
an IF on chat type:
```
chat_type = {{ ($('Telegram Trigger').item.json.message || $('Telegram Trigger').item.json.edited_message || $('Telegram Trigger').item.json.callback_query?.message || {}).chat?.type || '' }}
```
`chat_type === 'private'` → continue; otherwise → NoOp (silently ignore, no reply). Covers
message, edited_message and callback_query at once.

**Source.** A publishing product, "Private chat?".

**Status:** ✅ Confirmed in production.

### TG-R9. Treat an edit to the user's own message (edited_message) as a fresh verbatim input

**Context.** Need to let the user revise text they already sent by editing their own
message rather than sending a new one (UX: "correct it in place").

**Solution.**
1. Add the `edited_message` update type to the Telegram Trigger (alongside `message`,
   `callback_query`).
2. Everywhere incoming data is read, support both branches: `message?.… || edited_message?.…`
   (chat.id, from.id, text — in CHAT/CHATNUM, Check Access, save nodes).
3. In the router: the rule "there is an `edited_message` AND stage = `<manual-input mode>`"
   → save `edited_message.text` verbatim → show the preview. Any other `edited_message`
   (not in the right stage) → silently ignore (otherwise editing an old message would
   falsely trigger the flow).
4. Limitation: the bot can only catch edits to the user's **own** messages; the first input
   still has to arrive as a regular message, and only edits after that. Telegram only
   exposes edited_message for ~48 hours.

**Source.** A publishing product, "Save texts (message edit)" + the `edit_text_manual`
rule.

**Status:** ✅ Confirmed in production.

### TG-R10. Gracefully handle broken user-supplied HTML on send (onError → hint)

**Context.** If the text to send is authored by the user (manual input, their own
ready-made post), its HTML may be invalid (`<b>…<b>` instead of `</b>`). sendMessage/
sendPhoto with `parse_mode: HTML` fails on that (`can't find end tag …`) — the node turns
red and the user gets no feedback.

**Solution.** On the preview node (the first one to send the user's text with
`parse_mode: HTML`), set `onError: continueErrorOutput`. The error output (index 1) →
a separate static hint node (with no tags in its own text, so it can't fail itself):
"broken markup, the closing tag is `&lt;/b&gt;`, fix it and resend". The preview node acts
as a "gate": as long as the markup is broken, publishing is unreachable, and the user sees
a clear reason. Keep the stage in "verbatim" mode so a resend stays verbatim.

**Source.** A publishing product, "Text preview" (error output) → "Preview: broken markup".

**Status:** ✅ Confirmed in production.

### TG-R11. Dynamic list of cards with inline buttons (id in callback_data via expression)

- **Context:** need to show the user a list of database records (a post queue, tasks),
  each with its own action buttons.
- **Solution:** instead of building one shared keyboard, send **one card per item**: a
  Postgres node returns N rows → the Telegram node runs once per item. In the buttons'
  callback_data — an expression with the record id: `"=q_now_{{ $json.id }}"`. The
  callback handler in the router catches it by regex (`^q_now_`), extracting the id via
  `split('_')[2]`.
- **Advantages:** no hassle building a dynamic reply_markup JSON; each card is
  self-contained; actions are atomic by id (`UPDATE ... WHERE id AND status` — protects
  against stale buttons).
- **Empty list:** Postgres returns `{success:true}` on 0 rows (PG-E2) — an IF guard on a
  key field before the card is mandatory, otherwise a garbage card goes out.
- **Source:** a publishing product, the /queue command (queue cards with 🚀/🕒/❌).
- **Status:** ✅ Confirmed in production.

### TG-R12. Bot menu button = Web App (Mini App) instead of a command list

- **Task:** make the bot's menu button (bottom-left in the chat) open a Mini App instead
  of showing a command list.
- **Solution (Bot API, no BotFather needed):** `POST https://api.telegram.org/bot<token>/setChatMenuButton`
  with body `{"menu_button":{"type":"web_app","text":"Open","web_app":{"url":"https://<app-url>/"}}}`.
  Without `chat_id` — this sets the default for everyone. The URL must be HTTPS. To revert:
  `{"menu_button":{"type":"commands"}}`.
- **⚠️ Readback gotcha:** right after `set` (ok:true), `getChatMenuButton` may still return
  the old value (`{type:'commands'}`) — this is Telegram's **propagation delay**, not an
  error. Re-check after a minute: the default reads as `{type:'web_app',...}`, and a
  specific chat reads `{type:'default'}` (inheriting the default). Don't "fix" it by
  re-issuing `set` based on the first read.
- **Token:** pull it from `*_config`/the execution server-side; never output it to chat or
  logs.
- **Source:** a publishing-bot product.
- **Status:** ✅ Confirmed in production (the button opens the app).

### TG-R13. Forwarded message/link → Mini App inbox: detection and gate before the router

- **Task:** a bridge bot must put a forwarded news item into `pub_inbox` (→ the "From
  forwards" block in the Mini App), rather than processing it in chat.
- **Gotcha 1 — detection based only on `forward_date`.** The "Forwarded?" node checked
  `!!message.forward_date`. But when a user **shares a link** (via the share sheet or by
  pasting a URL), Telegram sends it as a **plain message with no forward field at all**
  (`message` = message_id/from/chat/date/text/entities — neither `forward_date` nor
  `forward_origin`). So a link-based news item wasn't recognized as "forwarded". Fix:
  broaden the condition — `!!m.forward_date || /^https?:\/\//i.test((m.text||m.caption||'').trim())`.
  (Also worth remembering: legacy `forward_from*`/`forward_date` are being phased out by
  Telegram in favor of `forward_origin` — check both for genuine forwards.)
- **Gotcha 2 — the detection node ran IN PARALLEL with the router.**
  `Load draft → [Router, Forwarded?]` ran simultaneously: even a recognized forward still
  went into the router too → with an active draft, the URL got "eaten" as a text edit. Fix:
  turn the detector into a **gate BEFORE the router** —
  `Load draft → Forwarded?`; true → inbox; **false (output 1) → Router**. The IF node
  passes the same item through, so the router gets the same data. Now a forward/link only
  goes to the inbox.
- **Gotcha 3 — silent write.** After the INSERT into `pub_inbox`, the bot stayed silent →
  "nothing happened" from the user's perspective. Added a confirmation reply
  ("🔗 Added to the app → open it, 'From forwards' block").
- **The inbox node is resilient to missing forward fields:** build `src` with fallbacks
  (`forward_origin.chat.title || sender_user || forward_from... || ''` → "Repost"); `text` =
  `message.text||caption`. For a bare link, src="Repost", text=URL — nothing breaks.
- **Source:** a publishing bot (`<workflow_id>`).
- **Status:** ✅ Confirmed in production.

### TG-R14. Logging into the web app via Telegram Login Widget → a session HMAC token (secret=SHA256(token), ≠ initData)

- **Task:** authenticate a user in the **browser-based** web version of a Telegram product
  (not a Mini App) against an n8n backend, then reuse the same data endpoints as the Mini
  App.
- **Key difference from Mini App signing.** Telegram provides **two different** schemes
  that are easy to confuse:
  - **Login Widget** (the "Log in with Telegram" widget on a website): `secret = SHA256(bot_token)`
    (binary digest), `data_check_string` = all widget fields **except `hash`**, sorted by
    key, `key=value`, joined with `\n`; `check = HMAC_SHA256(dcs, secret)` in hex must equal
    `hash`. Also check `auth_date` ≤ 86400s (anti-replay).
  - **Mini App initData** (see TG-E11): `secret = HMAC_SHA256("WebAppData", bot_token)`.
    **Not interchangeable** with the above.
- **Session instead of re-validating each time.** The widget's signature is only valid
  once (the widget isn't resent on every request). After a successful `auth/telegram`, the
  backend issues a **self-contained session token** with no server-side session table:
  `base64url(JSON{telegram_id,exp}) + "." + base64url(HMAC_SHA256(payload, SESSION_SECRET))`.
  Lifetime (`exp`) — e.g. 30 days. From then on, every request carries the token; the
  backend verifies the signature + `exp` and takes `user_id` **from the token, never from
  the payload** (protects against tampering).
- **`SESSION_SECRET` lives in the DB config** (table `*_config`, its own key), read by a
  `Load Config` node at the start of the workflow; never ends up in code/responses/logs.
  One secret per instance, generated once.
- **Reusing existing queries.** After the token gate, the same Postgres queries used by the
  Mini App (profile, facts, history, diary, achievements) also serve the web client —
  isolation is achieved by **duplicating the SQL in a separate workflow**
  (`<workflow_name>`), with no shared nodes with the miniapp/main bot.
- **Access gate** — the shared `bot_access(bot='<slug>', active=true)` check keyed on the
  `telegram_id` from the verified signature; no access → no token is issued (empty
  string), and the frontend shows "no access".
- **Source:** a coaching product, `<workflow_name>` (webhook `<webhook_path>`, id
  `<workflow_id>`).
- **Status:** ✅ Confirmed in production (a valid hash → token; a forged/expired one → 401).

### TG-R15. Secure Telegram avatar proxy — a signed URL, bot token never reaches the client

- **Task:** show the user's real Telegram avatar in the web app/Mini App **without exposing
  the bot token** to the client (Telegram's file URL contains the token in its path).
- **Solution — a dedicated proxy workflow with a signed URL:**
  1. Endpoint `GET /webhook/<webhook_path>?uid=<telegram_id>&sig=<signature>`, where
     `sig = HMAC_SHA256(String(uid), AVATAR_SECRET)` in hex. An invalid/missing signature →
     **403**. The signature is generated by the main API when it hands out `photo_url` —
     the client cannot forge `uid`.
  2. On a valid signature: Bot API `getUserProfilePhotos(user_id=uid, limit=1)` → take the
     largest size (the last element of `photos[0]`) → `file_id` → `getFile` → `file_path` →
     download `https://api.telegram.org/file/bot<token>/<file_path>` (HTTP node,
     response=file) → **return the raw bytes** (`image/jpeg`).
  3. Cache: respond with `Cache-Control: public, max-age=86400`. No photo / privacy
     restriction / Bot API error → **404** (the frontend shows a placeholder via `onerror`).
- **Security:** `bot_token` and `AVATAR_SECRET` come from `*_config` (a `Load Config`
  node) and never end up in the response/redirect/headers/logs. Downloading happens
  entirely on the n8n side → only image bytes leave the system, never a URL containing the
  token. HTTP nodes calling the Bot API: `retryOnFail` 3×5s,
  `getUserProfilePhotos`/`getFile` with `onError: continueRegularOutput` (graceful
  degradation to 404).
- **Consistency:** the Mini App should also use the proxy URL rather than
  `initData.photo_url` — one code path on the frontend, and the picture is always fresh
  and cacheable.
- **Source:** a coaching product, `<workflow_name>` (webhook `<webhook_path>`, GET, id
  `<workflow_id>`).
- **Status:** ✅ Confirmed in production (valid signature → 200 image/jpeg 640×640;
  malformed → 403; no token ever appears in the response).

### TG-R16. Mirroring a channel — an outgoing sendMessage from the web branch into the Telegram chat does not create a loop

- **Task:** a conversation the user has with the agent **from the website** should also be
  visible in their Telegram chat with the bot (a single conversation across two channels).
- **Solution:** after the web branch builds a reply, **in parallel** with the web response,
  send two `sendMessage` calls to Telegram on the bot's behalf: an echo of the user's
  message (tagged with the channel, e.g. `🌐 <Name> (from the website):\n<text>`) plus the
  agent's reply. The name comes from `profile.name` keyed by `telegram_id` (the same query
  that loads the prompt/profile — no extra node needed).
- **Why there's no loop:** the bot's own outgoing `sendMessage` does not produce an incoming
  update in the Telegram Trigger (the bot doesn't "hear" itself) → the main workflow never
  fires from it. Mirroring is safely one-directional.
- **Reliability (important):** the mirror branch runs **in parallel** with the web-response
  node (`Respond to Webhook` replies first and independently). The Telegram HTTP nodes use
  `onError: continueRegularOutput` + `retryOnFail` 3×5s: **a Telegram failure never breaks
  the web response** and never fails the execution. Conversation memory is **not
  duplicated** — the agent's memory node writes it during the main pass; the mirror only
  sends to Telegram.
- **Formatting is reused** from the main bot (`stripLeak` + converting the
  `⟪PROGRAM⟫` sentinel → `<blockquote expandable>`, `parse_mode=HTML` only when such a
  block is present). Use the **raw agent output**, not the version already "cleaned" for
  the web (in case that cleanup strips the sentinels). No token/`user_id` → the node
  returns `[]` (mirroring is skipped).
- **Source:** a coaching product, `<workflow_name>` branch `chat/send`
  (`Web Mirror Format`/`Mirror Echo`/`Mirror Reply`).
- **Status:** ✅ Accepted in production (logic verified; actual delivery from a live web
  session is left to the owner to confirm).

### TG-R17. Opening the Mini App to a specific screen from a bot message — an inline web_app button + a query-based deep link

- **Task:** in a bot message (e.g. a weekly ping), give a button — not a bare link — that
  opens the Mini App directly on the right screen.
- **Solution:** add a `reply_markup` with an inline `web_app` button to `sendMessage`:
  `reply_markup: {inline_keyboard:[[{text:'🗓 Open summary', web_app:{url:'https://domain/app?screen=weekly'}}]]}`.
  The domain must match the Mini App's domain (the one set in BotFather). The button opens
  a webview directly inside Telegram. It's also worth adding a text hint in the message
  itself ("see the Diary"), not just the button.
- **Deep link on the frontend side:** on startup, read the screen from
  `new URLSearchParams(location.search).get('screen')`, and in Telegram mode also check
  `tg.initDataUnsafe.start_param` (as a fallback). Use the value to set the initial tab
  before the first render.
- **n8n quirk:** in the httpRequest node, build `jsonBody` via
  `JSON.stringify({chat_id, text, reply_markup})`, preparing the `reply_markup` object in a
  preceding Code node (to avoid an unwieldy inline expression).
- **Source:** a coaching product, `<workflow_name>` Ping + the web frontend boot logic.
- **Status:** ✅ Confirmed in production (the button is delivered and opens the screen).

### TG-R18. Everything that isn't text — stickers, video notes, attachments — must not reach the agent as empty

- **Symptom:** a user sends a sticker — the bot replies "Looks like the message arrived
  empty. Resend it as text or voice." It looks like the agent is broken, but the agent
  actually did its job honestly: it really did receive an empty string.
- **Cause:** the typical normalization `type = voice ? 'voice' : photo ? 'photo' : 'text'`
  and `text = message.text || message.caption || ''`. A sticker has neither `text` nor
  `caption` — it falls into the "text" branch with empty content. Video notes, GIFs,
  videos, documents, locations, contacts, and polls fall into the same trap.
- **A key fact that saves work:** stickers don't need to be "recognized" — **Telegram sends
  its emoji directly in the message**: `message.sticker.emoji`. A sticker arrives with
  `emoji: '👍'`, and the set name is in `set_name`. So supporting stickers is one
  expression, not image parsing.
- **Solution:** extend the normalization node's fallback chain. Substitute the sticker's
  emoji as regular text (the LLM understands emoji-only messages fine — verified), and give
  every other type an honest short label:
```
text = message.text || message.caption
     || (voice||video_note||photo ? "" : (
          sticker ? (sticker.emoji || "[sticker]") : ""
       || video_note ? "[video message]" : ""      // if not transcribing — see below
       || animation  ? "[GIF]" : ""
       || video      ? "[video]" : ""
       || document   ? "[file]" : ""
       || location   ? "[location]" : ""
       || "[attachment]"))
```
- **Two gotchas, both catchable only by reading the neighboring branches:**
  1. **Voice and photo must be excluded from this chain.** They have their own branches,
     and `text` there is used as "the user's caption" next to the transcription/
     description. The shared fallback would tag every uncaptioned photo as "[attachment]".
  2. **Check order — specific before general.** A GIF's message has both `animation` and
     `document`; a video note has `video_note` alongside `video`. Checking the general case
     first yields "file" instead of "GIF".
- **A video note can skip labeling and be transcribed instead.** Telegram delivers
  `video_note` as an **.mp4**, and Whisper accepts mp4 directly — no need to extract the
  audio track. It's enough to tag it `type='voice'` and put `video_note.file_id` into the
  same field used for voice messages: the existing "download file → Whisper → caption"
  branch handles it with zero new nodes. A useful side effect: if the plan/rate gate reads
  `type`, it automatically covers video notes too — no loophole around the limits.
- **How to test:** send each type live, with a pause longer than the buffer's merge window
  (otherwise messages get merged into one), then check the trigger's input and the
  normalization node's output in the execution. In the coaching product: sticker →
  `text='👍'` → "Great 👍"; video note before transcription → `[video message]` → "Video
  message received without transcription, write a couple of words."
- **Source:** a coaching product, `<workflow_name>` / Normalize, 08.08.2026.
- **Status:** ✅ Fully confirmed on live messages. Sticker → `👍`, a 5-second video note —
  Whisper returned an accurate transcription, captioned "Transcription of video message
  (note)", and the agent replied contextually. The video note passed through the plan gate
  and was counted against the limit, exactly as intended.

### TG-R19. An alert must not depend on the database: send first, log second; if the DB is down, queue in the scenario's memory and send the alert "delayed"

- **Task:** a hardware-event notification (vendor webhook → Telegram) must get through
  even if the database holding recipients and the log is unreachable (the channel is in a
  different datacenter, tunnel down). At the same time, people's data must not be cached
  in the external perimeter.
- **Design (an event receiver for a corporate client, 11.09.2026):**
  1. The webhook replies to the vendor immediately (`responseMode: onReceived`) — the
     vendor only gets one delivery attempt.
  2. Parsing and dedup happen in JS held in the scenario's own memory:
     `$getWorkflowStaticData('global').seen` — the last 500 event ids. Never touches the
     database.
  3. Recipients and the reference table are read from the database **before** sending, in
     a single SELECT with `onError: continueRegularOutput`,
     `options.connectionTimeout: 10`; the source item is returned from the SQL as
     `$2::jsonb AS src`, so a node failure doesn't lose it (the failed item comes back as
     input + `error`).
  4. The alert goes out via the Bot API (HTTP Request, needs `message_id`), then logging
     happens in a single query with `notified_*` fields, `retryOnFail 3 × 10s`,
     `onError: continue`.
  5. If logging fails → the event, together with the fact that it was sent, goes into
     `sd.outbox` (capped at 2000), plus a service message to the owner. A Schedule branch
     ("every 5 minutes") in the **same scenario** (static data is per-workflow, another
     scenario can't see it) drains the queue:
     `INSERT … ON CONFLICT (event_id) DO UPDATE SET notified_* WHERE notified_at IS NULL` —
     a retry doesn't create a duplicate, and a late alert just backfills the sent flag.
  6. Events whose recipients never got them (the DB didn't respond at step 3) are, after
     the queue drains, routed back through the recipients node and sent with the note
     "delayed: the database was unreachable from HH:MM".
  7. `settings.executionTimeout: 120` — a "half-dead" tunnel accepts the connection and
     then goes silent; without an overall timeout, the node would hang indefinitely.
- **Gotchas and fixes along the way:**
  - A node with no input items doesn't execute and breaks the chain (DBG-R5) — so the
    "rejected", "no recipients" and "no database" branches are merged into the logging node
    via an IF, not via a filter that could return `[]`.
  - Static data is only persisted for production runs (not manual ones) — verify via a
    real webhook/simulated call; locally, stub out `$getWorkflowStaticData`, `$input`,
    `$('Node')` in the node.
  - Two concurrent executions can overwrite each other's static data — at a handful of
    events per day the risk is accepted; a higher-volume flow would need a queue table.
  - There's no fallback chat (a deliberate choice by the owner): recipient data is never
    stored in the external perimeter, so during an outage the only signal is a service
    message to the owner with the event's substance taken straight from the payload.
- **Testing:** synthetic events fired directly at n8n (`curl http://127.0.0.1:5678/webhook/…`):
  a new event → a message with buttons plus a row with `notified_msgs`; a repeat → no
  second row or message; an unrelated payload → logged as rejected; the "Acknowledged"
  button tested by simulating a Telegram Trigger update (DBG-R4).
- **Where applied:** a hardware-monitoring event receiver for a corporate client (generator
  in the client's repo, `receiver/build_receiver_wf.py`), 11.09.2026.
- **Status:** 🟡 Draft — synthetic testing passed, the database-failure path was only
  verified locally; will move to ✅ after the first real channel outage or owner
  confirmation.

---

## ⚠️ Errors

### TG-E1. `parse_mode: ""` does not disable Markdown

- **Symptom:** you want to send "raw" text, so you set `parse_mode: ""` — but Telegram
  still parses Markdown, and the text arrives formatted/broken on special characters.
- **Cause:** an empty string is not treated as "no formatting".
- **Fix:** set `parse_mode: "HTML"` (not an empty string, not Markdown). Found after 4
  round trips on the sandbox workbench (see debugging-sandbox.md DBG-R1).

### TG-E2. sendPhoto — the parameter is `file`, not `photo`; `parse_mode` is snake_case

- **Symptom:** you configure a photo send through the API, but the photo doesn't go out /
  the parameter is ignored.
- **Cause:** in the n8n Telegram node, the photo parameter is called `file`, not `photo`.
  Names are case-sensitive: `parse_mode` must be snake_case. n8n silently drops an unknown
  field name (see rest-api-deploy.md DEP-E4).
- **Fix:** use `file`; `parse_mode` in snake_case; verify with a GET after every PUT.

### TG-E3. "sent automatically with n8n" appended to bot messages

- **Symptom:** bot messages have an n8n attribution appended at the bottom.
- **Cause:** `appendAttribution: false` was never set.
- **Fix:** see TG-R1.

### TG-E4. After sendChatAction ("Send Typing"), downstream nodes lose chat_id/message_id/text

- **Symptom:** after the Telegram `sendChatAction` node, the following nodes get
  `undefined` instead of `chat_id` / `message_id` / `text`. Database writes get null, and
  replies go to the wrong place.
- **Cause:** the Telegram node (like most action nodes in n8n) **replaces the current item
  with its own API response** — after it, `$json` looks roughly like `{ "result": true }`,
  not the data from the normalizing node.
- **Fix:** in nodes after Send Typing, reference the upstream node by name —
  `$('Normalize').first().json.<field>` — instead of `$json.<field>`. Or move Send Typing
  to a side branch. The full pattern is TG-R3.

### TG-E5. `{{ }}` substitution in a node's text is delivered as raw text

- **Symptom:** the bot's message literally shows `{{ $json.foo }}` instead of the
  substituted value. Worse, it can break downstream logic that expects the substituted
  marker.
- **Cause:** n8n treats a string field as an **expression** only if it starts with `=`.
  Text containing `{{ }}` without a leading `=` is sent as a literal. Easy to miss on
  static text where the substitution is only in the middle.
- **Fix:** any field (`text`, `chatId`, `callback_data`, `url`) that contains `{{` must
  start with `=`. In the generator, use a helper:
  `'=' + v if '{{' in v and not v.startswith('=') else v`. Verify with a GET after every PUT.

### TG-E6. "Bad Request: chat not found" on sendMessage + the rule for choosing a bot

- **Symptom.** A Telegram sendMessage node fails with `Bad request - please check your parameters`,
  and the `description` shows `Bad Request: chat not found`. The credential is valid (no
  "Credential … does not exist" error).
- **Cause.** The bot behind that credential physically cannot message this `chatId` because
  the user **never pressed Start / never wrote to this bot**. A private chat with a bot
  only exists after the person has messaged that bot first. This is NOT a parameter error —
  Telegram simply doesn't know of such a chat for this bot. Every bot has its own separate
  private chat.
- **Fix.** Open the correct bot in Telegram and send it any message (Start). After that,
  sendMessage to that chatId goes through. Make sure to message the exact bot whose
  credential is set on the node.
- **Related rule.** During a build, Claude connected notifications to the wrong bot (a
  consultant bot) on its own initiative — this must never happen. **Before connecting ANY
  bot/credential to a workflow — always ask which bot to use. Never guess, and never reuse
  someone else's bot.** One bot = one Telegram Trigger (webhook): reusing the trigger bot in
  another scenario breaks the original one.
- **Status:** ⚠️ Recorded.

### TG-E7. Sporadic "chat not found" with an entirely valid config (a Telegram transient)

- **Symptom.** sendMessage fails with `Bad Request: chat not found`, but the config is
  correct and **the same bot successfully messages the same chat in neighboring runs**.
  Different from TG-E6: there, the bot was never started by the user and it fails
  deterministically every time; here, the failure is a one-off.
- **Sign that it's transient.** 2 of 3 identical runs (same `chatId`, same bot, same node,
  the only difference being the inline button's URL, which Telegram doesn't check for
  existence) succeeded, one failed.
- **Fix.** Don't hunt for a "parameter bug" — there isn't one. Enable node-level Retry On
  Fail (see TG-R6). A single transient failure shouldn't kill the whole workflow and lose a
  lead/notification.
- **Status:** ⚠️ Recorded.

### TG-E8. sendPhoto by URL with imgBB: "failed to get HTTP URL content" / "wrong type of the web page content"

- **Symptom.** A Telegram sendPhoto node with `file = {{ image_url }}` (a link to
  `i.ibb.co/.../*.png`) fails: either `Bad Request: failed to get HTTP URL content`
  (Telegram couldn't download it) or `Bad Request: wrong type of the web page content`
  (it downloaded the wrong thing). Sporadic: one generation goes through, the next one
  doesn't.
- **Cause.** Delivering the image **by link** shifts the download onto Telegram. The imgBB
  CDN is either "cold" right after upload (first error class) or serves content in a way
  Telegram doesn't recognize as an image (second error class). On top of that, photo-by-URL
  is capped at 5 MB, which a heavy PNG can exceed.
- **Fix.** Don't let Telegram pull the URL — download the image in the workflow and send it
  as **binary** (multipart). See TG-R7. Retrying the by-URL send doesn't fix "wrong type"
  (it's not a cold-CDN issue but a fundamental limitation of that delivery method).
- **Source.** A publishing product, "Final preview: photo" / "Publish to Telegram".
- **Status:** ⚠️ Recorded.

### TG-E9. An update from a channel/supergroup → "channel direct messages topic must be specified"

- **Symptom.** A bot built for DMs receives an update with a negative `chat.id`
  (channel/supergroup). The reply branch (sendMessage to that chat) fails: `Bad Request:
  channel direct messages topic must be specified` (a newer Telegram "channel direct
  messages" feature).
- **Cause.** The trigger catches messages from more than just DMs. Subscribing to
  `edited_message` or having the bot join a group widens the incoming stream. Telegram
  rejects a reply into a channel without a specified topic.
- **Fix.** Filter incoming updates by chat type. See TG-R8.
- **Source.** A publishing product, "Access denied".
- **Status:** ⚠️ Recorded.

### TG-E10. `require('crypto')` is blocked in Code nodes on this instance — HMAC/SHA256 must be pure JS

- **Symptom:** a Code node doing signing/verification (HMAC-SHA256, SHA256 — Login Widget/
  initData validation, proxy-URL signing, session tokens) fails on `require('crypto')`
  (the module isn't available in this n8n's Code-node sandbox). No crypto logic using the
  standard `crypto` module runs.
- **Cause:** this instance's settings don't allow built-in Node modules in Code nodes
  (`NODE_FUNCTION_ALLOW_BUILTIN` doesn't include `crypto`).
- **Fix:** implement SHA-256 and HMAC-SHA256 in **pure JS** (no `require`) and reuse the
  same block across every crypto node (signature validation, token issuance/verification,
  avatar-URL signing). In the coaching product, the pure-JS implementation was copied
  between the initData workflow and the web workflow (Login Widget + session + URL
  signing).
- **Prevention:** for any new crypto task in a Code node, immediately grab the ready-made
  pure-JS implementation from a neighboring workflow instead of writing
  `require('crypto')` "the backend way". Verify at execution time (the error only shows
  up at runtime, not in the config).
- **Source:** a coaching product, the `Auth`/signing nodes (copied across workflows).
- **Status:** ⚠️ Confirmed in production (pure JS works, `require('crypto')` does not).

### TG-E11. Signing Mini App initData for testing: compact JSON + `%20`, not `+`

- **Context:** to test the Mini App's webhook backend without Telegram (a curl request with
  a fake but validly signed `initData`), the signature is computed via double
  HMAC-SHA256: `secret = HMAC("WebAppData", bot_token)`, `hash = HMAC(secret, data_check_string)`,
  where `data_check_string` is `k=v` pairs (excluding `hash`), sorted, joined with `\n`,
  with values **URL-decoded**.
- **Gotchas:** (1) serialize the `user` field as **compact** JSON with no spaces
  (`separators=(",",":")`) — otherwise the string won't match what n8n gets after
  decodeURIComponent; (2) when building the query string, encode spaces as `%20`, **not
  `+`** (`urlencode(..., quote_via=quote)`): n8n uses `decodeURIComponent`, which doesn't
  turn `+` into a space → `data_check_string` diverges → hash mismatch → 401.
- **Also:** check the "admin only" gate against the `id` from the **verified** `initData.user`,
  never from the payload; `auth_date` no older than 24h.
- **Source:** an access-control product, Mini App API — smoke test for 401/200.
- **Status:** ⚠️ Confirmed (signed requests → 200, forged/malformed/empty → 401).

### TG-E12. A Telegram send node fires once per incoming item → a burst of duplicates

- **Symptom:** "it sent a pile of identical reports." A weekly digest sent the owner
  **16 identical messages** in a single run.
- **Cause:** the `sendMessage` node runs **once per incoming item**. It was fed by
  `UPDATE … RETURNING id` (updating metrics for 16 posts) → 16 items → 16 messages. The
  text was the same for all of them (`$('Metrics').first().json.digest` via `.first()`),
  hence "a pile of duplicates".
- **Fix:** send from a **single-item node**. Split the branches:
  `Summary (1 item) → [Send (1 message), Update metrics (16 rows, DEAD END)]`. Run the
  UPDATE **in parallel**, not before the send. A quick fallback is `executeOnce: true` on
  the send node (but the cleaner fix is the right topology).
- **Rule:** for any side-effect node (send/publish/HTTP-POST) at the end of a pipeline,
  check how many items feed into it. A multi-row SQL query/list upstream means that many
  actions.
- **Source:** a publishing product — Analytics (digest), 19.07.2026.
- **Status:** ✅ Confirmed in production: the Sunday run on 09.08.2026 produced **one**
  message instead of 16 (verified in the sandbox and then in live execution 25704).

### TG-E13. Broadcasting to a recipient list: one unreachable chat aborts everyone after it

- **Symptom:** a broadcast to several recipients in sequence — some got the message, some
  didn't, with no single error in the execution log (or an error present, but with no
  explanation of who exactly didn't receive it).
- **Cause:** the Telegram node gets one item per recipient and sends them one at a time.
  Without `onError`, the very first `chat not found` error (see TG-E6 — the recipient
  never pressed Start) aborts **the entire execution**: recipients before the failing one
  got the message, recipients after it didn't.
- **Fix:** on any broadcast-to-a-list node, set `onError: continueRegularOutput`, and
  surface undelivered recipients separately (a service message to whoever's responsible),
  so the loss isn't silent. Before adding a new recipient to the list, check the chat
  without actually sending: `getChat?chat_id=…` returns `chat not found` until the person
  has pressed Start with the bot.
- **Status:** ⚠️ Recorded.
