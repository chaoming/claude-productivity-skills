# Setup Guide

## Finding your Notion database ID

1. Open your Notion tasks database in a browser.
2. Look at the URL: `https://www.notion.so/<workspace>/<DATABASE_ID>?v=<view_id>`
3. The database ID is the 32-character string between the last `/` and the `?v=`.
4. Example: `https://notion.so/myworkspace/b84d0b3fa32847dc9af84316e4ef37be?v=...`
   → database ID is `b84d0b3fa32847dc9af84316e4ef37be`

If the URL doesn't show a database ID (e.g. you're viewing a page, not an inline database), open the database in full page view: click `⋯` → `Open as full page`.

## Installing and connecting Notion MCP

1. In Claude Code, run `/mcp` to open the MCP manager.
2. Search for "Notion" and install the official Notion MCP.
3. Follow the prompts to connect your Notion account and grant access to your workspace.
4. Verify it's working: ask Claude to "list my Notion databases" — it should return your databases.

## Authenticating the GitHub CLI

```bash
gh auth login
```

Choose GitHub.com, HTTPS, and authenticate via browser. Verify with:

```bash
gh auth status
```

## Adding `.task-management.json` to `.gitignore`

If you don't want to commit your Notion database ID:

```bash
echo ".task-management.json" >> .gitignore
```

If you want the whole team to share the same config, commit the file instead — the database ID is not a secret, just an identifier.
