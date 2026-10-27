# Local development with the CLI

`meridian dev` gives each developer a private MySQL 8.4 database on Meridian, reached through the Meridian database gateway on `127.0.0.1`. No Docker, no local MySQL, no cloud credentials. Billing works locally because Meridian can read the sessions this database holds (installs made locally are Shopify test charges).

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
| `meridian dev` | Provisions the database (up to 5 minutes the first time), opens the gateway, writes `DATABASE_URL` and `SHADOW_DATABASE_URL` into `.env`, then runs `npm run dev` | Long-running: start it in the background. Run another command with `meridian dev -- shopify app dev`. `--no-env` prints the URL instead of editing `.env` |
| `meridian migrate -- --name <name>` | Snapshots the database to `.meridian/snapshots/`, runs `prisma migrate dev`, restores the snapshot if it fails | Always pass `--name`, or Prisma prompts. Arguments after `--` go to Prisma |
| `meridian db seed` | Runs `npx prisma db seed` with the gateway open | |
| `meridian db shell --repl -e "<sql>" --json` | Runs one statement and prints the rows as JSON | `--repl` matters: without it an installed `mysql` client prints plain text |
| `meridian db reset --yes --migrate` | Drops every table, then reapplies the migrations (`--seed` to seed too) | Destructive. Confirm with the user first |
| `meridian db credential list --env production` | Lists external access credentials of a hosted environment's database | Creating and rotating them is dashboard-only |
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
| `meridian dev` is already running in this process tree | You are inside a `meridian dev` session. Run the command directly, without nesting another one |

The MCP tools `dev_database_get` (status, storage) and `dev_database_reset` cover the same database without the CLI. `dev_database_migrate` only records a run, it does not execute one.
