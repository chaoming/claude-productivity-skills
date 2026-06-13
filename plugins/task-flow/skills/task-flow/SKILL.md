---
name: task-flow
description: >
  Manage the full task lifecycle across Notion (planning) and GitHub Issues (execution).
  Use this skill when the user wants to create a task, move a task from planning to code,
  close out completed work, or set up the Notion + GitHub workflow for the first time.
  Supports both simple tasks (one Notion task → one GitHub Issue) and epics (one Notion
  task → multiple GitHub Issues). Triggers: "create task", "add to Notion", "graduate task",
  "this needs a GitHub issue", "break this into issues", "close task", "mark done",
  "task complete", "set up task management", "task-flow setup".
---

# task-flow

Two systems, one rule: **Notion for planning, GitHub Issues for execution**.

- **Notion** holds tasks before they touch code — ideas, epics, decisions, research.
- **GitHub Issues** track individual pieces of code work, always in the repo where the change lives.

A Notion task can map to **one GitHub Issue** (a simple task) or **many** (an epic broken into stories). Either way, the Notion task is the source of truth for planning; each GitHub Issue is the source of truth for its slice of the code work. The Notion task closes only when every linked issue is closed.

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

**4. GitHub Issues link field**
> "Do you have a text property in Notion for tracking linked GitHub Issue URLs?
> This field stores one issue URL per line, so a single Notion task can link to multiple issues (useful for epics).
> What is it called? (default: `GitHub Issues` — press enter to skip if you don't use one)"

Note: this must be a **Text** or **Rich Text** property in Notion, not a URL property. URL properties only hold a single value.

**5. Default GitHub repository**
> "What is the default GitHub repo for issues? Format: `owner/repo` (e.g. `acme/backend`). You can override this per issue."

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
    "github_issues_field": "<captured or null>"
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

### Graduate — create a GitHub Issue from a Notion task

Use when a Notion task (or part of an epic) is ready for code work.

This operation can be called multiple times on the same Notion task — each call adds another linked Issue, allowing a task to be broken into as many issues as needed.

**Task Start checkpoint** — announce this before taking any action:
> *"Task Start checkpoint — creating GitHub Issue for Notion task."*

Steps:

1. Ask for the Notion task URL (or name to search for it).
2. Fetch the task via `notion-fetch` to get its title, description, and current `github_issues_field` value.
3. Determine whether this is the first issue or an addition to an existing epic:
   - **First issue** (field is empty or null): this is either a simple task or the start of an epic.
   - **Subsequent issue** (field already contains URLs): this is adding another issue to an existing epic. Show the user the existing linked issues so they have context.
4. Ask for: issue title (defaults to Notion task title), description (optional), repo (defaults to config default).
5. Create the GitHub Issue:
   ```bash
   gh issue create \
     --repo <repo> \
     --title "<title>" \
     --body "<description>

   ---
   Notion task: <notion url>"
   ```
6. Update the Notion task via `notion-update-page`:
   - Append the new Issue URL to `github_issues_field` (one URL per line, preserve existing URLs).
   - If this is the **first issue**: set status to `in_progress`.
   - If this is a **subsequent issue**: leave status unchanged (already In Progress).
7. Report the new Issue URL and the running issue count for this task.

```
Announcement: "Task Start checkpoint complete — GitHub Issue #<N> open (issue 2 of 2 for this task). Notion task In Progress."
```

---

### Close — complete a GitHub Issue

Use when a PR is merged or work on a specific issue is accepted as complete.

**Task Close checkpoint** — announce this before taking any action:
> *"Task Close checkpoint — closing GitHub Issue #<N>."*

Steps:

1. Ask for the GitHub Issue number (and repo if different from default).
2. Close the issue:
   ```bash
   gh issue close <N> -R <repo>
   ```
3. Retrieve the Notion task URL from the issue body:
   ```bash
   gh issue view <N> -R <repo> --json body --jq '.body' | grep "Notion task:" | grep -o 'https://[^ ]*'
   ```
4. If a Notion URL is found:
   - Fetch the Notion task and read all URLs from `github_issues_field`.
   - For each linked issue URL, check its state:
     ```bash
     gh issue view <issue_number> -R <repo> --json state --jq '.state'
     ```
   - **All issues closed** → update Notion task status to `done`.
   - **Some issues still open** → leave Notion task In Progress; announce which issues remain open.
5. Report completion.

```
# All issues closed:
Announcement: "Task Close checkpoint complete — GitHub Issue #<N> closed. All issues done — Notion task marked Done."

# Some issues still open:
Announcement: "Task Close checkpoint — GitHub Issue #<N> closed. 1 of 3 issues still open (#12) — Notion task stays In Progress."
```

---

## Edge cases

| Situation | Behaviour |
|---|---|
| Notion MCP unavailable | Skip Notion steps; announce "Notion MCP unavailable — update Notion manually" |
| `gh` CLI not authenticated | Stop; instruct user to run `gh auth login` |
| No `Notion task:` line in issue body | Close GitHub Issue only; announce "No Notion task linked — update Notion manually" |
| `github_issues_field` not configured | Skip the Notion field update; still update status |
| Notion task already Done | Update anyway — never skip because status looks correct |
| Different repo than default | Ask the user to confirm `owner/repo` before creating or closing |
| Issue URL in Notion field points to a different repo | Extract owner/repo from the URL when checking issue state |

---

## Reference

For detailed setup instructions (finding your Notion database ID, installing Notion MCP, authenticating `gh`), read `references/setup-guide.md`.
