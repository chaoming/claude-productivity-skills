# Claude Productivity Skills

General-purpose productivity skills for [Claude Code](https://claude.ai/code). Each skill is a self-contained workflow methodology — install once, use across any project.

## Skills

### `task-flow`

Manages the full task lifecycle across **Notion** (planning) and **GitHub Issues** (execution).

- **Setup wizard** — configure your Notion database and default GitHub repo once per project
- **Create** — add tasks to Notion before they're ready for code
- **Graduate** — move a Notion task to a GitHub Issue when code work begins (Task Start checkpoint)
- **Close** — close the GitHub Issue and mark the Notion task Done when work is complete (Task Close checkpoint)

**Prerequisites:** Notion MCP connected + `gh` CLI authenticated.

## Installation

### Via Claude Code plugin marketplace

Add this repo as a marketplace in your Claude Code settings:

```json
{
  "plugins": {
    "extraKnownMarketplaces": [
      {
        "name": "claude-productivity-skills",
        "url": "https://github.com/chaoming/claude-productivity-skills"
      }
    ]
  }
}
```

Then install the plugin via `/plugins` in Claude Code and search for `task-flow`.

### Manual installation

Clone this repo and copy the skill directory into your local skills folder:

```bash
cp -r plugins/task-flow/skills/task-flow ~/.claude/skills/task-flow
```

## Usage

Once installed, Claude will invoke the skill automatically when you say things like:

- "create a task"
- "graduate this to GitHub"
- "close the task"
- "set up task management"

On first use in a new project, the setup wizard will run and write a `.task-management.json` config file.
