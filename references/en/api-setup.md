# Connecting to n8n via API — the easiest way to work with n8n

**The main recommendation of this skill.** The most convenient, fastest, and most predictable
way to manage n8n with an AI assistant (the way we do it) is **through the n8n server's REST
API**, not by clicking through the UI. The assistant reads workflows itself, makes surgical edits
to nodes, deploys, checks executions, and finds errors — without manually clicking around the
canvas.

> Why this approach:
> - **Surgical edits.** You can change a single node in the JSON and deploy it without touching
>   anything else.
> - **Full cycle from the terminal/scripts.** Creating, activating, checking runs, diagnosing
>   errors — all programmatic and reproducible.
> - **Versionable.** The exported `workflow.json` lives in the repository, and history is visible
>   through diffs.
> - **The AI assistant works directly.** By giving it an API key and a domain, you delegate all
>   the routine work of building and debugging to it.

Deploy discipline (PUT overwrites credentials, minimal settings, activation checks) is covered in
`rest-api-deploy.md`. This file is about how to connect in the first place.

---

## What you need

- **Self-hosted n8n** (your own installation on a server/VPS). n8n has a built-in **Public REST
  API** — that's what we use.
- Access to the n8n UI with permissions that allow you to create an API key.

---

## Step 1. Get a Public REST API key

1. In the n8n interface: **Settings → n8n API → Create an API key**.
2. Copy the key (it's shown only once). It's a JWT with audience `public-api`.
3. This key is passed in every request via the `X-N8N-API-KEY: <key>` header, and the endpoints
   are `/api/v1/...`.

> If there's no "n8n API" item, the Public API is disabled in the instance config
> (`N8N_PUBLIC_API_DISABLED`). It's enabled by the server administrator.

## Step 2. Determine the base URL

This is the address of your n8n, for example `https://n8n.example.com`. All calls go to
`<BASE_URL>/api/v1/...`.

## Step 3. Store credentials in a separate env file (not in code!)

Create a `.n8n-api-key` file (add it to `.gitignore`) and keep the key and domain in it:

```bash
# .n8n-api-key
export N8N_API_KEY="<your public-api JWT>"
export N8N_BASE_URL="https://n8n.example.com"
```

Then, in any script or terminal: `source ./.n8n-api-key` — and the key and domain are available
as variables. For the AI assistant, it's enough to point it at this file — it substitutes the
values itself, without ever exposing the secret in plaintext in the chat history.

## Step 4. Test connectivity

```bash
source ./.n8n-api-key
curl -s -H "X-N8N-API-KEY: $N8N_API_KEY" \
  "$N8N_BASE_URL/api/v1/workflows?limit=5" | head
```

If you get back JSON with a list of workflows, the connection works. A `401` means the key is
wrong; a `404` on the domain means the `BASE_URL` is wrong or the Public API is disabled.

---

## Key Public API endpoints

| Action | Request |
|---|---|
| List workflows (search by name) | `GET /api/v1/workflows?limit=200` |
| Get a workflow (current JSON) | `GET /api/v1/workflows/{id}` |
| Create a workflow | `POST /api/v1/workflows` |
| Update a workflow (full node replacement!) | `PUT /api/v1/workflows/{id}` |
| Activate / deactivate | `POST /api/v1/workflows/{id}/activate` · `/deactivate` |
| Executions (error diagnostics) | `GET /api/v1/executions?workflowId={id}&limit=3` |
| List credentials (find their IDs) | `GET /api/v1/credentials` |
| Tags | `GET/POST /api/v1/tags`, attach `PUT /api/v1/workflows/{id}/tags` |

> ⚠️ Before your first `PUT`, be sure to read `rest-api-deploy.md` (DEP-R2): GET does not return
> `credentials`, and a PUT without them wipes out all node bindings. This is the main pitfall of
> API deployment.

---

## Optional: the n8n MCP server

Newer versions of n8n have an MCP server (audience `mcp-server-api`, header
`Authorization: Bearer <token>`, endpoint `/mcp-server/http`) — a separate token for SDK/MCP
operations. This is an alternative transport; the basic, universal path is the Public REST API
above. Don't mix up the two tokens: the REST key doesn't work on the MCP endpoint and vice versa.

---

## Access security

- An API key = full access to workflows and data. Store it like a password, don't commit it,
  don't paste it into Sticky Notes.
- If it's compromised, revoke the key in **Settings → n8n API** and create a new one.
- Don't expose the domain or webhook URLs publicly.
