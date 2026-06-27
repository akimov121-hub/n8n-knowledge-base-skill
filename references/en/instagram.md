# Instagram (Graph API, content publishing)

Publishing to Instagram from self-hosted n8n via `graph.instagram.com` (Instagram API with
Instagram Login), Meta tokens, image hosting.

> The Meta dashboard UI steps and the API version (`v23.0`) are current as of writing —
> Meta periodically renames menu items and bumps the version. Cross-check against the current documentation.

---

## IG-R1 ✅ Access to the Instagram API for your own account without App Review

**Context.** You need to publish posts to your own IG account from n8n. Self-hosted n8n has no
intermediary services (like the master application that cloud platforms provide).

**Solution** (✅ confirmed in production):

1. The IG account must be **professional** (Business/Creator) and **public**.
2. developers.facebook.com → Create app → use case **"Manage messaging and content on
   Instagram"** (INSTAGRAM_BUSINESS).
3. **App roles → Add people → "Instagram tester"** (this exact role, not
   "Developer") → the IG account's username. The invitation for the app owner's account is
   confirmed automatically (instagram.com/accounts/manage_access/ → "Tester invites").
4. Use cases → Set up Instagram login API → block 1 "Add all required
   permissions" → block 2 **"Add account"** (Instagram login + 2FA) → **"Generate token"**.
5. The token is long-lived (~60 days) and starts with `IGAA...`. It works both as `?access_token=` and as
   the `Authorization: Bearer` header → in n8n, store it as an **httpHeaderAuth** credential.

**Photo publishing cycle** (no polling, which is only needed for video/Reels):

```
POST https://graph.instagram.com/v23.0/me/media        (image_url, caption)  → {id: container_id}
POST https://graph.instagram.com/v23.0/me/media_publish (creation_id)        → {id: media_id}
GET  https://graph.instagram.com/v23.0/{media_id}?fields=id,permalink        → link to the post
```

App Review is not required as long as you only publish to accounts that have a role in the app. Token renewal:
`GET https://graph.instagram.com/refresh_access_token?grant_type=ig_refresh_token&access_token=<token>`
(the token must be older than 24h). Deleting posts via the API is not supported — only manually.

The host is **graph.instagram.com** (not graph.facebook.com): the Instagram Login token works
there specifically, and no Facebook Page is needed.

---

## IG-E1 ⚠️ "Insufficient developer role permissions" when linking the IG account

**Symptom.** In the Meta dashboard, "Generate access tokens" block → "Add account" → after
logging into Instagram, a popup with the error "Insufficient developer role permissions".

**Cause.** The IG account has not been granted the **"Instagram tester"** role in the app. The
"Developer" role (Facebook profile) is not the same thing and does not help.

**Fix.** App roles → Add people → "Instagram tester" (the lower item,
"Additional roles") → the IG account's username → for your own account the invitation
is auto-confirmed → retry "Add account". Status: ✅ reproduced and resolved.

---

## IG-E2 ⚠️ Error 9004/2207052 "Failed to download media file" when creating a container

**Symptom.** `POST /me/media` →
`{"error":{"code":9004,"error_subcode":2207052,"message":"Only photo or video can be accepted as media type."}}`
— the wording is misleading; the real text is in `error_user_msg`: "Failed to fetch the media file by
URI".

**Cause.** Meta's bot could not download the image from `image_url`. Some CDNs with anti-bot
protection (e.g. `upload.wikimedia.org`) refuse it.

**Fix.** Use hosting that Meta can download from without issues (verified: imgBB,
`i.ibb.co`). Format — JPEG. Status: ✅ reproduced and resolved.
