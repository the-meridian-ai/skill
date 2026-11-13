# Local development with the CLI

`meridian dev` gives each developer a private database on Meridian (MySQL 8.4 or PostgreSQL 18, the engine of the app), reached through the Meridian database gateway on `127.0.0.1`. No Docker, no local database server, no cloud credentials. Billing works locally because Meridian can read the sessions this database holds (installs made locally are Shopify test charges).

On an app whose engine is PostgreSQL, keep `provider = "postgresql"`: `meridian dev` writes `postgresql://` URLs for `DATABASE_URL` and `SHADOW_DATABASE_URL`. Do not switch the provider to `mysql`, because the deploy would then fail its engine check.

The project must be linked first (`references/setup.md`).

## Prepare Prisma (once)

The Shopify template ships with SQLite. Every database command refuses a schema that is not:

```prisma
datasource db {
  provider = "mysql"
  url      = env("DATABASE_URL")
}
```

After switching, delete the old SQLite migrations in `prisma/migrations` (they cannot apply to MySQL) and create a fresh one with `meridian migrate -- --name init`. Ask the user before deleting migrations that already ran against a deployed database.

`meridian migrate` also needs `shadowDatabaseUrl = env("SHADOW_DATABASE_URL")` in the datasource. It offers to add the line (`--yes` accepts). In a non-interactive shell without `--yes` it stops and prints the line to add.

## Commands

Every command accepts the global flags `--json` (one JSON result on stdout, human lines on stderr), `--yes` (approve confirmations), `--no-input` (never prompt, fail instead) and `--cwd <dir>`.

| Command | What it does | Notes for agents |
| --- | --- | --- |
| `meridian dev` | Provisions the database (up to 5 minutes the first time), opens the gateway, writes `DATABASE_URL` and `SHADOW_DATABASE_URL` into `.env`, injects `MERIDIAN_APP_ID`, a dev SDK key and `MERIDIAN_API_URL` into the dev server's environment (never into `.env`), then runs `npm run dev` | Long-running: start it in the background. Run another command with `meridian dev -- shopify app dev`. `--no-env` prints the URL instead of editing `.env`. The SDK works with no manual step. Do not ask the user for a key; a `MERIDIAN_APP_ID` or `MERIDIAN_API_KEY` already in their shell or `.env` wins and nothing is injected |
| `meridian migrate -- --name <name>` | Snapshots the database to `.meridian/snapshots/`, runs `prisma migrate dev`, restores the snapshot if it fails | Always pass `--name`, or Prisma prompts. Arguments after `--` go to Prisma |
| `meridian db seed` | Runs `npx prisma db seed` with the gateway open | |
| `meridian db shell --repl -e "<sql>" --json` | Runs one statement and prints the rows as JSON | `--repl` matters: without it an installed `mysql` client prints plain text |
| `meridian db reset --yes --migrate` | Drops every table, then reapplies the migrations (`--seed` to seed too) | Destructive. Confirm with the user first |
| `meridian db credential list --env production` | Lists external access credentials of a hosted environment's database | Creating and rotating them is dashboard-only |
| `meridian env list --env development` | Lists the environment's variables and secrets, with the ones set for all environments: key, type, scope, whether the environment overrides it, last update | Never shows a value. Works on every environment, production included |
| `meridian env set KEY --env development --plain` | Creates or replaces one variable in development or staging. The value comes from stdin when it is piped, from `--from-file <path>`, or from a masked prompt in a terminal. A new key is a secret unless `--plain`. An existing key keeps its type | Never pass the value as an argument. A plain variable you may set without asking (say what you set). A secret: ask first, then let the user type it in their own terminal or pipe a generated one (`SKILL.md`, "Variables and secrets"). Production and all environments are refused: use MCP `secrets_set` once the user agrees |
| `meridian env unset KEY --env development` | Deletes the environment's own entry of a key | Ask the user first. Needs `--yes` when it cannot prompt. A key set for all environments is refused (MCP `secrets_unset` with the user's agreement) |
| `meridian deploy --watch` | Deploys the checked-out commit to production (`--env staging`, `--sha <commit>`) and follows it to the end, printing each phase and the new log lines | The commit must be pushed. Exits 1 when the deploy fails, after printing the reason and the last log lines. Without `--watch` it only starts the deploy. `--deployment <id>` follows one already started. Confirm a production deploy with the user first |
| `meridian whoami --json` | The signed-in user, their organizations and the linked project | |
| `meridian link --org <slug> --app <slug>` | Links the project | See setup |
| `meridian skills install` | Installs or updates this skill and the MCP config in the project | `--agent all`, `--no-mcp` |

`.env` keeps every other line. The first change saves `.env.backup`.

## Errors

Exit code 1 for an expected error, 130 on Ctrl-C, otherwise the child process's code. With `--json` an error prints `{"error": {"message", "hint"}}` on stdout. Read the hint before retrying.

| Message | Fix |
| --- | --- |
| You are not logged in / token has expired / no longer valid (401) | The user runs `meridian login` |
| This project is not linked to a Meridian app | `meridian link --org <slug> --app <slug>` |
| Cannot prompt for X in a non-interactive shell | Pass the flag the hint names, or `--yes` |
| `schema.prisma` sets `provider = "sqlite"`, or reads another variable than `DATABASE_URL` | Switch the datasource (above) |
| `dev_database_limit_reached` | The organization's plan has no developer database left. Tell the user |
| `cli_upgrade_required` | `npm install -g @the-meridian/cli@latest` |
| Production variables can't be changed from the CLI | Ask the user, then use MCP `secrets_set` / `secrets_unset` for a plain variable, or ask them to set a secret in the dashboard |
| `meridian dev` is already running in this process tree | You are inside a `meridian dev` session. Run the command directly, without nesting another one |

The MCP tools `dev_database_get` (status, storage) and `dev_database_reset` cover the same database without the CLI. `dev_database_migrate` only records a run, it does not execute one.
