# Meridian plugin

The Meridian skill for Claude Code, Codex and Cursor. It teaches your coding agent to build, bill and deploy a Shopify app on [Meridian](https://the-meridian.ai):

- connect the Meridian MCP server, so the agent reads your real plans, feature keys, deploys and logs
- install and drive the Meridian CLI (`meridian login`, `link`, `dev`, `migrate`, `db`)
- add `@the-meridian/sdk` to your app: pricing page, feature gating, usage tracking
- get the app ready for Meridian hosting, deploy it, and debug a failed deploy

The agent leaves sign-in, API keys and secrets to you, and asks before a production deploy, a rollback or a database reset.

## Install

**With the Meridian CLI** (any agent), from your app's root:

```bash
npm install -g @the-meridian/cli
meridian skills install
```

It writes the skill to `.claude/skills/meridian`, `.agents/skills/meridian` (Codex) and `.cursor/skills/meridian` for the agents it finds in the project, and adds the Meridian MCP server to their config. Commit those folders so your teammates get them.

**As a Claude Code plugin** (skill and MCP server together):

```
/plugin marketplace add the-meridian-ai/skill
/plugin install meridian@meridian
```

**As a Codex plugin** (skill and MCP server together):

```bash
codex plugin marketplace add the-meridian-ai/skill
codex plugin add meridian@meridian
```

**In Cursor:** install Meridian from the Cursor Marketplace (Customize, Marketplace).

**With the skills CLI:**

```bash
npx skills add the-meridian-ai/skill
```

Then approve the Meridian MCP connection in your browser the first time your agent uses it.

## Documentation

[AI agents](https://help.the-meridian.ai/ai-agents) in the Meridian docs.

## Privacy and terms

- [Privacy policy](https://app.the-meridian.ai/privacy)
- [Terms of service](https://app.the-meridian.ai/terms)

## Contributing

This plugin is generated from Meridian's main repository on every release. Pull requests to the-meridian-ai/skill are not merged. Report problems to support@the-meridian.ai.
