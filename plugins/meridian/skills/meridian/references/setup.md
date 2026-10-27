# Setup: CLI, MCP, sign-in, link

Do these once per machine (CLI, sign-in) and once per project (MCP config, link). Skip any step that is already done.

## 1. Install the CLI

Needs Node 20 or later.

```bash
npm install -g @the-meridian/cli
meridian --version
```

## 2. Connect the Meridian MCP server

Server URL: `https://mcp.the-meridian.ai/mcp` (Streamable HTTP, OAuth 2.1 with PKCE). There is no API key: the client registers itself and the user approves it once in the browser.

`meridian skills install` writes the project config for every agent it finds (pass `--no-mcp` to skip). To do it by hand:

| Client | Config | Then |
| --- | --- | --- |
| Claude Code | `claude mcp add --transport http meridian https://mcp.the-meridian.ai/mcp`, or `.mcp.json`: `{"mcpServers": {"meridian": {"type": "http", "url": "https://mcp.the-meridian.ai/mcp"}}}` | The user runs `/mcp` and approves |
| Cursor | `.cursor/mcp.json`: `{"mcpServers": {"meridian": {"url": "https://mcp.the-meridian.ai/mcp"}}}` | The user opens **Customize** in Cursor's sidebar (Settings, Tools & MCP before Cursor 3) and connects Meridian |
| Codex | `.codex/config.toml` (trusted projects) or `~/.codex/config.toml`: `[mcp_servers.meridian]` with `url = "https://mcp.the-meridian.ai/mcp"` | The user runs `codex mcp login meridian` |

The Claude Code plugin (`/plugin marketplace add the-meridian-ai/skill`) already declares the server.

Check the connection with the `whoami` tool. It returns the user and every organization they belong to.

The MCP server only reaches the production Meridian platform (and the staging and development environments of apps there).

## 3. Sign in the CLI (the user does it)

Ask the user to run this in their own terminal, because it waits for them to approve a code in the browser:

```bash
meridian login
```

Then check it yourself:

```bash
meridian whoami --json
```

`whoami --json` prints `user`, `orgs` (each with `id`, `name`, `slug`) and the linked project. A token lasts 90 days. "You are not logged in" or "token has expired" means the user signs in again.

## 4. Link the project

Run from the app's root (the directory with `package.json`). Find the organization slug in `whoami` and the app slug in `apps_list` (both MCP tools), then:

```bash
meridian link --org <org-slug> --app <app-slug>
```

This writes `.meridian/project.json` (commit it, it holds no secret) and `.meridian/.gitignore`. With one organization or one app the CLI picks it. Add `--force` to relink a linked project.

No app yet: the user creates it in the dashboard (Create app). It needs the Shopify app's client ID and secret.

## 5. Install this skill in the project

So every teammate's agent gets it:

```bash
meridian skills install
```

It writes the skill to `.claude/skills/meridian`, `.agents/skills/meridian` (Codex) and `.cursor/skills/meridian`, for the agents it detects (`--agent claude,codex,cursor` or `--agent all` to choose), plus the MCP config above. Commit those folders. Run it again after a CLI upgrade to update the skill.
