---
name: task-flow
description: >
  Manage the full task lifecycle across Notion (planning) and GitHub Issues (execution).
  Use this skill when the user wants to create a task, graduate a task to GitHub, close
  out completed work, or set up the Notion + GitHub workflow for the first time.
  Every task goes through a Plan → Approve → Execute flow: Claude analyses the Notion
  task, proposes an issue breakdown (one or many), writes a checklist to the Notion page,
  waits for user approval, then creates all issues automatically.
  Triggers: "create task", "add to Notion", "graduate task", "this needs GitHub issues",
  "plan this task", "break this into issues", "close task", "mark done", "task complete",
  "set up task management", "task-flow setup".
---

# task-flow

Two systems, one rule: **Notion for planning, GitHub Issues for execution**.

- **Notion** holds tasks before they touch code — ideas, epics, decisions, research.
- **GitHub Issues** track individual pieces of code work, always in the repo where the change lives.

Every Notion task goes through a structured flow before any code work begins: Claude reads the task, plans a breakdown into one or more GitHub Issues, writes that plan as a checklist on the Notion page, and waits for user approval before creating anything. The Notion task closes only when every linked issue is closed and the user verifies completion.

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
> "Do you have a text property in Notion for storing linked GitHub Issue URLs?
> This field stores one issue URL per line and is used to track all issues linked to a task.
> What is it called? (default: `GitHub Issues` — press enter to skip if you don't use one)
>
> Note: this must be a **Text** or **Rich Text** property, not a URL property — URL properties only hold a single value."

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

### Graduate — plan and execute a Notion task as GitHub Issues

This is a three-phase operation. Do not skip ahead — each phase gates the next.

---

#### Phase 1: Plan

**Enter plan mode before doing anything else.** Use `EnterPlanMode` if available. The goal is to think through the task scope before proposing anything to the user.

Steps:

1. Ask for the Notion task URL (or name to search for it).
2. Fetch the full task via `notion-fetch` — read the title, description, and any existing page content.
3. If the task description is too sparse to reason about scope (a title with no context), ask the user:
   > "This task doesn't have much detail yet. Can you describe what needs to happen so I can plan the right issues?"
4. Analyse the task scope in plan mode:
   - What are the distinct pieces of work?
   - Can any be done independently or in parallel?
   - Is this one focused piece of work, or does it span multiple concerns (e.g. schema + API + UI)?
   - What's the right level of granularity — small enough to ship in a PR, large enough to be meaningful?
5. Produce a proposed breakdown: a numbered list of issues, each with a title and one-sentence description.
6. Append the plan as a checklist to the bottom of the Notion task page using `notion-update-page` with `command: "insert_content"` and `position: {"type": "end"}`:

   ```markdown
   ## GitHub Issues Plan

   - [ ] <Issue title> — <one-sentence description>
   - [ ] <Issue title> — <one-sentence description>
   - [ ] <Issue title> — <one-sentence description>
   ```

   Before calling `insert_content`, read the Notion Markdown spec via `notion://docs/enhanced-markdown-spec` to ensure correct checkbox syntax.

7. Present the plan to the user:
   > *"I've planned X issues for this task and added the checklist to the Notion page. Here's the breakdown:*
   > *1. Issue title — description*
   > *2. Issue title — description*
   >
   > *Review the checklist on the Notion page and let me know if you'd like to adjust anything, or approve to proceed."*

8. **WAIT for user approval.** Do not create any GitHub Issues until the user explicitly approves.

If the user requests changes, update the checklist on the Notion page using `notion-update-page` with `command: "update_content"` (search-and-replace the relevant lines), then present the revised plan again.

---

#### Phase 2: Execute

Fires immediately after the user approves the plan. No further prompting between issues — execute the full plan automatically.

**Task Start checkpoint** — announce before creating the first issue:
> *"Task Start checkpoint — executing plan, creating X GitHub Issues."*

For each planned issue in order:

1. Create the GitHub Issue:
   ```bash
   gh issue create \
     --repo <repo> \
     --title "<issue title>" \
     --body "<issue description>

   ---
   Notion task: <notion url>"
   ```
2. Update the Notion task:
   - Append the new Issue URL to `github_issues_field` (one URL per line, preserve existing).
   - Update the corresponding checklist item on the page using `notion-update-page` with `command: "update_content"`, replacing the plain item with the issue number and URL:
     - old: `- [ ] <Issue title> — <description>`
     - new: `- [ ] <Issue title> — #<N> <url>`
3. Announce each issue as it is created: `"Created #<N>: <title>"`

After all issues are created:
- Set Notion task status to `in_progress`.
- Announce completion:

```
Announcement: "Task Start checkpoint complete — X GitHub Issues created, Notion task In Progress.
  #1: <title> — <url>
  #2: <title> — <url>
  ..."
```

---

### Close — verify and complete a task

Use when the user indicates work is done. This operation has a verification gate — the Notion task is not marked Done until the user confirms.

Steps:

1. Ask for the Notion task URL or GitHub Issue number.
2. Fetch the Notion task and read all linked issue URLs from `github_issues_field`.
3. Check the state of each linked issue:
   ```bash
   gh issue view <N> -R <repo> --json state,title --jq '{state: .state, title: .title}'
   ```
4. Present a completion summary to the user:

   **All issues closed:**
   > *"All X issues for this task are closed:*
   > *✓ #1: <title>*
   > *✓ #2: <title>*
   >
   > *Ready to mark the Notion task Done. Approve?"*

   **Some issues still open:**
   > *"X of Y issues are closed. These are still open:*
   > *○ #3: <title>*
   >
   > *Do you want to close them now, or keep the task In Progress until they're done?"*

5. **WAIT for user verification and approval.**

6. On approval:

   **Task Close checkpoint** — announce:
   > *"Task Close checkpoint — marking Notion task Done."*

   - Set Notion task status to `done` via `notion-update-page` with `command: "update_properties"`.
   - Check off each completed item in the checklist using `notion-update-page` with `command: "update_content"`:
     - old: `- [ ] <Issue title> — #<N> <url>`
     - new: `- [x] <Issue title> — #<N> <url>`

```
Announcement: "Task Close checkpoint complete — Notion task marked Done."
```

---

## Edge cases

| Situation | Behaviour |
|---|---|
| Notion MCP unavailable | Skip all Notion steps; announce which actions need to be done manually |
| `gh` CLI not authenticated | Stop; instruct user to run `gh auth login` |
| Task description too sparse to plan | Ask user to elaborate before entering plan mode |
| User adjusts plan after approval | Re-enter Phase 1, update checklist, wait for re-approval |
| Some issues still open at Close | Present summary, ask whether to close them now or leave task In Progress |
| No `Notion task:` line in issue body | Cannot retrieve Notion URL; ask user to provide it manually |
| `github_issues_field` not configured | Skip field updates; rely on checklist on the page for tracking |
| MCP does not support page block updates | Display checklist in conversation; ask user to paste it into Notion |
| Issue URLs span multiple repos | Extract `owner/repo` from each stored URL when checking issue state |

---

## Reference

For detailed setup instructions (finding your Notion database ID, installing Notion MCP, authenticating `gh`, creating the GitHub Issues text field), read `references/setup-guide.md`.
