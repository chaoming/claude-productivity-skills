# Claude Productivity Skills

A collection of workflow skills for [Claude Code](https://claude.ai/code) that give Claude opinionated, automated procedures for common development workflows — so you stop explaining the same process every session and just get things done.

---

## Who this is for

**Solo developers and small teams** who use Notion for planning and GitHub for code, and want Claude to be a full participant in their task workflow — not just a coding assistant that needs to be told what to do at every step.

If you've ever found yourself manually copying task titles from Notion into GitHub Issues, forgetting to update task statuses after merging a PR, or explaining to Claude "now go update the Notion task" every single time — these skills automate that loop.

---

## Skills

### `task-flow` — Notion + GitHub task lifecycle

Manages tasks across two systems with one clear rule: **Notion for planning, GitHub Issues for execution**.

The idea is simple: not every task needs a GitHub Issue. An idea, an epic, a decision, a piece of research — these live in Notion until they're ready for code. When a task reaches the point where code must change, it **graduates**: Claude opens a GitHub Issue, links it back to the Notion task, and keeps both systems in sync as work progresses and completes.

#### What it does

| Operation | What happens |
|---|---|
| **Create** | Adds a new task to your Notion database |
| **Graduate** | Three-phase flow: Claude plans the issue breakdown, writes a checklist to the Notion page, waits for your approval, then creates all GitHub Issues automatically |
| **Close** | Checks all linked issues, presents a completion summary, and waits for your verification before marking the Notion task Done |
| **Setup** | One-time wizard that configures your Notion database and default GitHub repo |

#### The workflow

```
Notion task (idea / planning)
        │
        │  "graduate this task"
        ▼
Phase 1 — Plan
  Claude reads the task, enters plan mode,
  proposes a breakdown (1 or many issues),
  writes checklist to the Notion page
        │
        │  you review and approve
        ▼
Phase 2 — Execute (automatic)
  Claude creates all GitHub Issues
  Links each one back to the Notion task
  Notion task → In Progress
        │
        │  PRs merged, work done
        │  "close task"
        ▼
Phase 3 — Verify
  Claude checks all linked issues
  Presents completion summary
        │
        │  you approve
        ▼
Notion task → Done
```

Claude decides whether a task needs one issue or ten — you don't have to specify upfront. The checklist on the Notion page is the living plan: items appear when planned, get issue numbers when created, and get checked off when closed.

At each transition, Claude announces a named checkpoint so the sync is visible and auditable — not silent background work.

#### Who benefits most

- **Solo founders** managing a backlog across strategy (Notion) and code (GitHub) without a project manager
- **Small dev teams** where Notion is the shared planning space and GitHub is where code happens
- **Developers who use Claude Code daily** and want task management to be part of the conversation, not a separate manual step

#### Prerequisites

- [Notion MCP](https://github.com/makenotion/notion-mcp-server) connected in Claude Code
- [GitHub CLI](https://cli.github.com/) installed and authenticated (`gh auth login`)

Both are free and take a few minutes to set up. See [the setup guide](plugins/task-flow/skills/task-flow/references/setup-guide.md) for step-by-step instructions.

#### Example triggers

Once installed, say any of these to Claude:

```
"create a task: redesign the onboarding flow"
"add this to Notion as a task"
"this is ready for code, graduate it to GitHub"
"open a GitHub issue for the auth bug"
"close task — PR was merged"
"mark the task done"
"set up task management for this project"
```

On first use in a new project, Claude runs a short setup wizard and writes a `.task-management.json` config file. Each subsequent use reads from that config — no repeated setup.

---

## Installation

### Via Claude Code plugin marketplace (recommended)

Add this repo as a known marketplace in your `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "claude-productivity-skills": {
      "source": {
        "source": "github",
        "repo": "chaoming/claude-productivity-skills"
      }
    }
  },
  "enabledPlugins": {
    "task-flow@claude-productivity-skills": true
  }
}
```

### Manual installation

Clone this repo and copy the skill into your local Claude skills folder:

```bash
git clone https://github.com/chaoming/claude-productivity-skills.git
cp -r claude-productivity-skills/plugins/task-flow/skills/task-flow ~/.claude/skills/task-flow
```

---

## Why a skill instead of a CLAUDE.md section?

You could describe this workflow in your project's `CLAUDE.md` — and many people do. The problem: you'd have to copy that description into every project, maintain it manually, and it loads into Claude's context window on every session whether you need it or not.

A skill loads on demand, lives in one place, ships with a versioned install, and works identically across every project without any copy-paste. It's the difference between a sticky note and a procedure that actually runs.

---

## Coming soon

More productivity skills are planned for this repo:

- **`deep-work`** — distraction-free session planning with time-boxed goals
- **`weekly-review`** — structured weekly review across Notion tasks and GitHub Issues
- **`standup`** — generate a standup summary from yesterday's GitHub activity

Contributions welcome — open an issue to propose a new skill.
