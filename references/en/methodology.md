# n8n Project Methodology

Working rules for organization, naming, architecture, and protocols for self-hosted n8n.
These are best practices developed in real-world use; adapt them to your own team.

---

## 1. Project Structure

**One workflow — one folder.** All files for a workflow (workflow.json, prompts, description)
are kept in a dedicated subfolder. This rule always applies — even if there is only a single file,
even for a minor tweak, even for a new version (v2, v3 go into the same folder). Never
dump workflow files into the root.

**Flat structure inside the workflow folder.** All artifacts live at one level. Nesting is
justified only for a bundle of several workflows (an AI agent with tools, a master + sub).

**Sandbox.** Sandbox workflows live in a shared `sandbox/<name>/` folder, each in its own subfolder,
with its own description. After a test session, deactivate them (see debugging-sandbox.md).

```
/project
├── CLAUDE.md / README             ← project rules
├── _Index.md                      ← map of all workflows (if you keep a vault)
├── [Workflow Name]/
│   ├── Description [Name].md       ← goal, architecture, files, development history
│   ├── workflow.json              ← latest export from n8n
│   ├── system-prompt.md           ← system prompt (if there is an AI node)
│   └── changelog.md               ← version history
└── sandbox/
    └── [Sandbox Name]/
```

---

## 2. Naming

**Workflows:** format `[Client/Domain] — [Action]`. Examples: `Construction — Deadline
Reminders`, `Inbound Leads — Qualification + CRM`. Versions: `v2`, `v3` at the end.

**Workflow folders:** the name matches the workflow name verbatim. Name by purpose, not by
delivery channel (no `TG-`, `WA-` prefixes). Renaming a folder is an atomic operation:
rename the folder, the description file, the index entry, and all links at the same time.

**Nodes inside a workflow:** meaningful names (not `HTTP Request`, but `Fetch data from ERP`).
Left-to-right order = execution order.

**Credentials:** format `[Service] — [Purpose/Client]`. Examples: `Telegram Bot —
Notifications`, `OpenAI — Main`, `PostgreSQL — Prod`.

---

## 3. Sticky Note — the Workflow's Business Card

**Required in every workflow** (including sub- and tool-workflows). Created first. It is the first
thing you see when you open a workflow.

- `type`: `n8n-nodes-base.stickyNote`, `color`: `5` (orange), `width`: `360`, `height`: `216`.
- Position: to the left and slightly above the trigger node (x = trigger_x − 400, y = trigger_y − 20).

Content format (4–5 lines, don't duplicate the description file):

```
## [Workflow Name]
[One sentence: what it does and for whom.]
[Key actors / entry points: who or what triggers it.]
[Integrations and tables it works with.]
[Any special constraints or warnings, if applicable.]
```

If the workflow is expensive (LLM/image generation), add `⚠️ Expensive run — debug via
sandbox.` Update the business card silently on: a new input channel, a new integration, a new credential,
a change in key logic, a rename, or promotion to Production.

---

## 4. Workflow Description Files

Each folder contains `Description [name].md`. It includes: the gist (1–2 sentences); a "what's in the folder"
table; the trigger and launch conditions; a description of the key nodes and data flow (with an ASCII
diagram if there are > 5 nodes); dependencies (credentials, external services); the **development
history** (origin, what was done, difficulties and how they were resolved, what was deferred); and the
status (Draft / Testing / Production).

The development history is mandatory if the workflow was ported from another tool, if there were
non-trivial decisions, or if part of the functionality was deferred.

---

## 5. Architectural Principles

- **One workflow — one responsibility.** Better 3 simple ones than 1 tangled one.
- **Modularity through sub-workflows.** Move reusable logic (formatting notifications,
  validation, calling a specific API) into a separate workflow (Execute Sub-workflow).
- **An Error Workflow is mandatory** for every production workflow. Standard chain:
  `Error Trigger → Edit Fields (formatting) → Telegram (notification)`. It contains: the name of
  the failed workflow, the node ID, the error message, and a timestamp.
- **Retry for unstable APIs.** On external HTTP requests, set `Retry On Fail`: 3 attempts, 5 sec.
- **Validation on input.** After a Webhook/Form trigger, the first node should be an IF on required fields.
  If any are missing — a clear error + Stop And Error.

---

## 6. Working with Data (expressions cheat sheet)

```
{{ $json.name }}                            — field from the current item
{{ $node["Node Name"].json.field }}         — field from a specific node
{{ $('Node Name').first().json.field }}     — safe cross-node ref (see AI-E1)
{{ DateTime.now().toFormat('dd.MM.yyyy') }} — date
{{ $input.all().length }}                   — number of items
{{ $runIndex + 1 }}                         — iteration number starting from 1
{{ JSON.stringify($json) }}                 — entire item as a string
```

Typical transformations: rename/add fields → Edit Fields (Set); filter →
Filter; remove duplicates → Remove Duplicates; split into batches → Split In Batches; merge
branches → Merge; aggregate → Aggregate / Summarize.

---

## 7. AI Agent Structure (typical)

```
Webhook/Chat → AI Agent
                 ├── Chat Model (LLM)
                 ├── Memory (Postgres Chat Memory)
                 └── Tools:
                       ├── HTTP Request Tool (external APIs)
                       ├── Workflow Tool (sub-workflows)
                       └── Vector Store Tool (RAG)
```

RAG pattern:
```
[Ingest] → Document Loader → Text Splitter → Embeddings → Vector Store
[Query]  → Webhook → AI Agent + Vector Store Tool → Response
```

Details on memory, model selection, and RAG without vectors are in ai-agents.md.

---

## 8. The "Plan First — Then Build in One Pass" Protocol

For porting workflows (Make/Zapier → n8n) and any large or non-trivial task. The stages are
strictly sequential:

1. **Study the source materials** — all provided files and the current state of n8n. Propose
   nothing until everything has been read.
2. **Draft a detailed plan** — for porting, a "source module → n8n node" table;
   for a new workflow, the left-to-right node architecture.
3. **Propose solutions for each node** and explicitly surface the decision points: model selection,
   memory type, error and empty-response handling, session-key source, cutover, access control,
   and data-protection compliance.
4. **Agree with the client** — ask clarifying questions, wait for explicit consent.
5. **Build in one pass** — only after consent: create the folder and files, deploy, and
   run a smoke test.

Don't start building before an explicit "let's do it." The discussion stage is itself the work.

---

## 9. Finalization After "It Works" Is Confirmed

Once the client confirms the workflow is finally working — do the following silently, without waiting
for a separate request:

1. **Record the reusable solution** in the knowledge base (status ✅, now permitted).
2. **Take local snapshots of ALL workflows in the bundle** (not only the changed ones): re-export from
   live, restore `credentials` by node id (GET does not return them), save into the folder. The folder
   becomes a complete, up-to-date snapshot of the bundle.
3. **Sync the description and the index.**
4. **Briefly report what was done** — which solutions were recorded, which snapshots were updated.

Interim "ok, next" or "got it" during discussion are NOT a finalization signal.

---

## 10. Security

- **⛔ Do not delete or irreversibly change DATA without explicit permission.** `DELETE`, `DROP`,
  `TRUNCATE`, destructive `UPDATE`, clearing conversation history, deleting files in the working folder,
  deleting production workflows/credentials. First state exactly what will be deleted and where, and
  wait for an explicit "yes." Exception — your own technical scaffolding (temporary SQL runners,
  debugging harnesses): you may clean those up freely. When in doubt — treat it as data and ask.
- Store credentials only in n8n (not in code, not in Sticky Notes).
- Do not publish webhook URLs.
- For sensitive personal data — use a local LLM (Ollama), not the cloud (data-protection compliance).
- Tokens and secrets — via n8n environment variables, never hardcoded.
- Store REST/MCP API keys in a separate environment file outside the repository (for example,
  `.n8n-api-key`, added to `.gitignore`), not in workflow code.

---

## 11. New Workflow Checklist

- [ ] Meaningful name `[Domain] — [Action]`
- [ ] Folder created before saving files
- [ ] Description file created
- [ ] Index entry added
- [ ] Sticky Note — business card (color 5, left of trigger, 4–5 lines)
- [ ] Error Workflow assigned
- [ ] Input data validation
- [ ] Retry On Fail on HTTP requests
- [ ] Credentials not hardcoded
- [ ] Tested with Pin Data
- [ ] If expensive (LLM, images, paid APIs) — a sandbox is set up/used for the last mile
