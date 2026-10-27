# Meridian agent skill (`packages/skill`)

The public `meridian` agent skill, laid out as a Claude Code plugin marketplace. The folder is mirrored as-is to the public repository `the-meridian-ai/skill` (`.github/workflows/skill-mirror.yml` in meridian-world), so everything here is public: no internal links, hosts, names or plans.

## Layout

One plugin (`plugins/meridian`), described for three clients:

| Client | Marketplace | Plugin manifest | MCP server |
| --- | --- | --- | --- |
| Claude Code (and Anthropic's directory) | `.claude-plugin/marketplace.json` | `plugins/meridian/.claude-plugin/plugin.json` | `plugins/meridian/.mcp.json` |
| Cursor (and the Cursor Marketplace) | `.cursor-plugin/marketplace.json` | `plugins/meridian/plugin.json` (Agent Plugins format) | `plugins/meridian/mcp.json` |
| Codex (and OpenAI's plugin directory) | `.agents/plugins/marketplace.json` | `plugins/meridian/plugin.json`, OpenAI fields under `extensions.com.openai` | `plugins/meridian/mcp.json` |

- `plugins/meridian/skills/meridian/`: the skill itself (`SKILL.md` and `references/`), shared by every client
- `plugins/meridian/assets/`: the logo (`logo.png`, 512 px, and `logo.svg`), copied from `apps/web/app/icon.*`

Bump `version` in both plugin manifests and the Cursor marketplace together when the skill changes, or installed copies never update. `packages/cli/src/skill-references.test.ts` fails when the manifests disagree on the name, version, folder or MCP URL.

The listing copy and the submission steps for the three directories are in `docs/platform/DIRECTORY_LISTING.md`.

## Rules

- The skill routes and guards. Product behaviour lives in `docs/public` and the agent reads it through MCP `docs_search` / `docs_get`. Do not copy doc pages in here.
- Every CLI command and flag named in the skill must exist, and every backticked snake_case word must be an MCP tool or prompt registered in `apps/api/app/Mcp/Servers/MeridianServer.php` (or be listed in `NOT_MCP_NAMES`). `packages/cli/src/skill-references.test.ts` checks both, in CLI CI.
- `@the-meridian/cli` bundles the skill folder (`meridian skills install`). A skill change reaches CLI users only with a CLI release, so add a changeset in `packages/cli`.
- No em dashes.
- Validate with `claude plugin validate --strict .` and `claude plugin validate --strict ./plugins/meridian` from this directory, and try the Codex install with `codex plugin marketplace add <this folder>` under a throwaway `CODEX_HOME`.
