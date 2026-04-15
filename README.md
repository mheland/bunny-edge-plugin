# Bunny CDN Edge Scripting — Claude Code Plugin

A [Claude Code](https://claude.ai/claude-code) plugin for scaffolding, developing, and deploying [Bunny CDN Edge Scripts](https://docs.bunny.net/scripting) with edge-replicated database support.

## Install

```
/plugin marketplace add mheland/bunny-edge-plugin
/plugin install bunny-edge@mheland
```

## What it does

Use `/bunny-edge` or just describe what you need — the skill auto-triggers when working with Bunny edge scripting.

**Scaffold** — generates project structure with script template and GitHub Actions deploy workflow, for both standalone scripts and middleware.

**Develop** — knows the full runtime API surface so it can help you write edge scripts with:
- Standalone request handlers (`BunnySDK.net.http.serve`)
- Middleware for Pull Zones (`BunnySDK.net.http.servePullZone`)
- HTMLRewriter for streaming HTML transforms
- WebSockets
- Node:FS virtual file system
- Environment variables and secrets

**Database** — covers Bunny Database (edge-replicated libSQL/SQLite), including:
- `@libsql/client/web` SDK usage from edge scripts
- Parameterized queries, batch transactions, interactive transactions
- Replication model, consistency guarantees, and gotchas
- SQL API (HTTP) for direct access

**Deploy** — guides through the GitHub integration workflow (including the constraint that repos must be created from Bunny's side), Actions setup, and the Bunny REST API for programmatic deployment.

## Skill contents

| File | Coverage |
|------|----------|
| `SKILL.md` | Architecture, scaffolding templates, local dev, deployment workflow, key APIs, limits, best practices |
| `runtime-reference.md` | HTMLRewriter (full API), WebSockets, Node:FS, environment variables, secrets |
| `database-reference.md` | Bunny Database — SDK, SQL API, replication, transactions, durability model |
| `deployment-guide.md` | GitHub integration, Actions workflow, REST API endpoints, pricing |
| `examples.md` | Standalone, middleware, HTMLRewriter, and database-backed script examples |

## Usage examples

```
/bunny-edge scaffold standalone my-api
/bunny-edge scaffold middleware auth-layer
```

Or just ask naturally:

- "Create a Bunny edge script that returns JSON"
- "Add HTMLRewriter to strip tracking scripts from the response"
- "Set up a REST API with Bunny Database"
- "How do I deploy this to Bunny via GitHub Actions?"

## Resources

- [Bunny Edge Scripting docs](https://docs.bunny.net/scripting)
- [Bunny Database docs](https://docs.bunny.net/database)
- [Claude Code custom skills](https://docs.anthropic.com/en/docs/claude-code/skills)
