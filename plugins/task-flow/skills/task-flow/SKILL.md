---
name: task-flow
description: >
  Manage the full task lifecycle across Notion (planning) and GitHub Issues (execution).
  Use this skill when the user wants to create a task, move a task from planning to code,
  close out completed work, or set up the Notion + GitHub workflow for the first time.
  Triggers: "create task", "add to Notion", "graduate task", "this needs a GitHub issue",
  "close task", "mark done", "task complete", "set up task management", "task-flow setup".
---

# task-flow

Two systems, one rule: **Notion for planning, GitHub Issues for code**.

- **Notion** holds tasks before they touch code — ideas, epics, decisions, research.
- **GitHub Issues** track work once code must change, always in the repo where the change lives.

When a Notion task reaches the point where code must be written, it **graduates** — a GitHub Issue is opened, its URL is recorded in the Notion task, and Notion status is updated. The Issue is the source of truth from that point.

## Prerequisites

- Notion MCP connected (provides `notion-*` tools)
- `gh` CLI authenticated (`gh auth status`)

If either is missing, announce which is unavailable and fall back to manual instructions for that step.

---

## Step 0 — Check setup

Before any operation, look for `.task-management.json` in the project root.

- **Found** → load config and proceed to the requested operation.
- **Not found** → run the Setup Wizard before doing anything else.

---

## Setup Wizard

Run this once per project. At the end, write `.task-management.json` and confirm.

Walk the user through these questions in order, one at a time:

**1. Notion database ID**
> "What's the ID of your Notion tasks database? You can find it in the database URL:
> `notion.so/<workspace>/<DATABASE_ID>?v=...`
> Just the ID portion — 32 hex characters."

**2. Status field name**
> "What is the name of the status property in your Notion database? (default: `Status`)"

**3. Status values**
> "What are the values for each state in your status field?
> - Not started yet (default: `Not Started`)
> - In progress (default: `In Progress`)
> - Done (default: `Done`)"

**4. GitHub Issue link field**
> "Do you have a property in Notion for linking to the GitHub Issue URL? If so, what is it called? (default: `GitHub Issue` — press enter to skip if you don't use one)"

**5. Default GitHub repository**
> "What is the default GitHub repo for issues? Format: `owner/repo` (e.g. `acme/backend`). You can override this per task."

Write config:

```json
{
  "notion": {
    "database_id": "<captured>",
    "status_field": "<captured>",
    "status_values": {
      "todo": "<captured>",
      "in_progress": "<captured>",
      "done": "<captured>"
    },
    "github_issue_field": "<captured or null>"
  },
  "github": {
    "default_repo": "<captured>"
  }
}
```

Confirm: *"Config saved to `.task-management.json`. Add this file to `.gitignore` if your Notion database ID is sensitive, or commit it if you want the whole team to share the same config."*

---

## Operations

### Create — add a task to Notion

Use when the user wants to capture something that isn't ready for code yet.

1. Ask for: task name (required), brief description (optional), priority (optional).
2. Call `notion-create-pages` with the tasks database ID and the provided fields. Set status to the `todo` value.
3. Report the Notion page URL.

```
Announcement: "Task created in Notion: <title> — <url>"
```

---

### Graduate — move from Notion to GitHub

Use when a Notion task reaches the point where code must change.

**Task Start checkpoint** — announce this before taking any action:
> *"Task Start checkpoint — graduating Notion task to GitHub Issue."*

Steps:

1. Ask for the Notion task URL (or name to search for it).
2. Fetch the task via `notion-fetch` to get its title and description.
3. Create a GitHub Issue:
   ```bash
   gh issue create \
     --repo <repo> \
     --title "<notion task title>" \
     --body "<description>

   ---
   Notion task: <notion url>"
   ```
4. Update the Notion task:
   - If `github_issue_field` is configured: set it to the GitHub Issue URL via `notion-update-page`.
   - Set status to `in_progress` via `notion-update-page`.
5. Report both URLs.

```
Announcement: "Task Start checkpoint complete — GitHub Issue #<N> open, Notion task In Progress."
```

---

### Close — complete a task

Use when a PR is merged or work on the branch is accepted as complete.

**Task Close checkpoint** — announce this before taking any action:
> *"Task Close checkpoint — closing GitHub Issue and marking Notion task Done."*

Steps:

1. Ask for the GitHub Issue number (and repo if different from default).
2. Close the issue:
   ```bash
   gh issue close <N> -R <repo>
   ```
3. Retrieve the Notion URL from the issue body:
   ```bash
   gh issue view <N> -R <repo> --json body --jq '.body' | grep "Notion task:" | grep -o 'https://[^ ]*'
   ```
4. If a Notion URL is found: update the Notion task status to `done` via `notion-update-page`.
5. Report completion.

```
Announcement: "Task Close checkpoint complete — GitHub Issue #<N> closed, Notion task Done."
```

---

## Edge cases

| Situation | Behaviour |
|---|---|
| Notion MCP unavailable | Skip Notion steps; announce "Notion MCP unavailable — update Notion manually" |
| `gh` CLI not authenticated | Stop; instruct user to run `gh auth login` |
| No `Notion task:` line in issue body | Close GitHub Issue only; announce "No Notion task linked — update Notion manually" |
| Notion task already In Progress or Done | Update anyway — never skip because status looks correct |
| Different repo than default | Ask the user to confirm `owner/repo` before creating or closing |

---

## Reference

For detailed setup instructions (finding your Notion database ID, installing Notion MCP, authenticating `gh`), read `references/setup-guide.md`.
