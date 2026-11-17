---
name: meridian
description: Build, bill, run and deploy a Shopify app on Meridian. Use when the project uses or should use Meridian (the-meridian.ai), `@the-meridian/sdk`, the `meridian` CLI or the Meridian MCP server, or when the user asks to add plans, pricing, billing, entitlements, feature gating or usage tracking to a Shopify app, to run it against a Meridian developer database, or to deploy it on Meridian hosting and debug that deploy.
---

# Meridian

Meridian is a platform for Shopify app developers: plans and billing through Shopify, feature gating and usage metering (the SDK), managed developer databases (the CLI), hosting on Cloud Run with GitHub deploys, and Grow tools (CRM, emails, automations). This skill tells you how to use each piece and what never to do.

## Before anything else

1. **Read the live facts, never guess them.** Feature keys, event keys, plan ids, environment ids and injected variables come from the Meridian MCP server. Product behaviour comes from the docs: call `docs_search`, then `docs_get` for the full page. Do not answer a Meridian question from memory, and treat this skill as a map, not the reference.
   - MCP not connected yet: ask the user to connect it (`references/setup.md`) before writing code that needs keys or ids. For the docs alone, fetch a page as Markdown at `https://help.the-meridian.ai/<page>.md`, which is the `docs_get` path with `.md` instead of `.mdx` (`sdk-quickstart.mdx` is `https://help.the-meridian.ai/sdk-quickstart.md`; the `.mdx` URL does not exist). The page list is `https://help.the-meridian.ai/llms.txt`.
2. **Check what is already set up**, in this order, and fix only what is missing:
   - `meridian --version` (the CLI is installed)
   - the MCP server is connected (a `whoami` tool call answers)
   - `meridian whoami --json` (signed in), and `.meridian/project.json` exists (the project is linked)
   - `@the-meridian/sdk` is in `package.json` and `app/meridian.server.ts` (or `app/meridian/meridian.server.ts` in the Meridian template) exists (the SDK is wired)

## Pick the workflow

| The user wants to | Read |
| --- | --- |
| Start a project made from the Meridian Shopify app template (`app/meridian/` exists) | [references/template.md](references/template.md) |
| Install the CLI, connect MCP, sign in, link the project | [references/setup.md](references/setup.md) |
| Add plans, pricing, gating or usage tracking (the SDK) | [references/sdk.md](references/sdk.md) |
| Run the app locally on a Meridian database, migrate, seed | [references/cli.md](references/cli.md) |
| Deploy, follow a deploy, fix a failed deploy, set variables | [references/hosting.md](references/hosting.md) |
| Work on webhooks, Shopify API alerts, CRM, emails, automations | "Operate and grow" below |

A new app usually goes through all four references in that order: setup, SDK, local development, deploy. A project made from the Meridian template starts with `references/template.md` instead, which asks the developer a few questions first and skips what the template already did.

## Rules that always apply

**The human does these, never you.** Ask the user, explain why, and wait:

- Signing in: `meridian login` runs a device-code flow the user approves in their browser. Ask them to run it in their own terminal, then check with `meridian whoami --json`.
- Approving production access: the user approves it in their browser, after confirming it's them. You may start the request and hand it over (see "Production changes from the CLI" below), but never approve anything yourself.
- Approving the MCP connection (the OAuth consent screen in their browser).
- Creating or rotating the production `MERIDIAN_API_KEY` (Settings, SDK in the dashboard) and putting it in the hosting secrets. For local development `meridian dev` provides `MERIDIAN_APP_ID`, a dev SDK key and `MERIDIAN_API_URL` itself, so do not ask the user to paste a key into `.env`. Never ask them to paste a key, a password or a secret into the chat.
- Typing a secret's value (see "Variables and secrets" below), installing the Meridian GitHub app, adding a custom domain or a database, choosing the auto-deploy branch and target, choosing a plan.

**Confirm in chat before you do these**, naming exactly what will happen:

- Deploying to production (`deployment_create`, or `meridian deploy` as in "Production changes from the CLI" below), promoting to production, or `deployment_rollback`.
- `meridian db reset`, `dev_database_reset`, or anything else that drops data.
- `automation_update` on an automation that is already active, or `custom_field_values_set` on a real store.
- Setting a secret, changing any variable on production or for all environments, or deleting any variable. Name the key, the scope and the environment.
- Changing a setting of the production environment with `environment_update` (release command, health check path, port, compute). Name the setting and its new value. The tool needs `confirm: true`: pass it only after the user's yes, never on your own.
- `sdk_credentials_set`: the Shopify client ID and secret reach every environment, production included. It needs `confirm: true` too.
- `environment_create`: it uses the plan's environment allowance, and a new development environment changes where pushes to the auto-deploy branch land.

**Hosting settings** (`references/hosting.md` has the detail):

- Development and staging: change a setting yourself with `environment_update`, then tell the user what you set. Nothing in production changes.
- Production: nothing without the user's yes in chat, as listed above. There is no production setting you may change without asking.
- The Shopify client ID is public: after the user's yes, set it with `sdk_credentials_set`. The client secret is a secret: ask the user to type it in the dashboard (App settings, Meridian SDK). Use `sdk_credentials_set` for it only when the value is already in the project's untracked local environment and you can pass it without printing it.

**Production changes from the CLI** (any `meridian` command that writes to production). The CLI token is enough for development and staging. A production change also asks a confirmation question before it is sent, and may need production access, which the user approves in their browser for a limited time (`meridian deploy` to production needs it). Go in this order:

1. Ask in the chat first, naming the action, the app and the environment, and wait for the user's agreement.
2. When the command needs production access, check it with `meridian whoami --json`: `production_grant_expires_at` is when it ends, null without it. Without it, or when the CLI refuses with `Run: meridian login --production`, start `meridian login --production --json` in the background. It puts the link and the code on stderr and waits up to 15 minutes. Give the user the link and the code, ask them to approve it in their browser, and never approve anything yourself. Do not pass `--for`: keep the default of 1 hour unless the user asks for another duration. Once they approved, check `meridian whoami --json` again.
3. Only then run the command with `--yes`. `--yes` answers the CLI's own question and never replaces production access. Never pass it for a production change the user has not agreed to in the chat.

**Variables and secrets** (the app's hosting environment variables):

- A plain variable on development or staging: set it yourself without asking, then tell the user what you set. Pipe the value in, for example `printf '%s' "$VALUE" | meridian env set API_URL --env development --plain`. That is fine only because a plain value is not secret.
- A secret on development or staging: once the user agrees, either they type it in their own terminal with `meridian env set KEY --env development` (a masked prompt) or in the dashboard, or you generate a random one and pipe it without ever printing it: `openssl rand -hex 32 | meridian env set KEY --env development --secret`.
- Production, or all environments at once: the CLI refuses both. Once the user agrees, use MCP `secrets_set` / `secrets_unset` for a plain variable, and ask the user to type a secret in the dashboard.
- Deleting: `meridian env unset KEY --env development` (or `secrets_unset`), only after the user agrees.
- Read names with `meridian env list --env development` or `secrets_list`. Neither shows a value.
- Never ask for a value in the chat, never put a secret on the command line or in a file you commit, and never print one.

**Never:**

- Put `MERIDIAN_API_KEY` in browser code, a client bundle, a theme or a commit. It is server only and imports from `@the-meridian/sdk/server` only.
- Invent a feature key, event key, plan id or variable name. Read it (`features_list`, `events_list`, `plans_list`, `managed_variables_list`).
- Copy code from an old Meridian example that imports from `meridian-sdk` (the package's former name) or sends a Shopify access token to Meridian. The current SDK never does.

## Operate and grow

Every tool below takes `app_id` (from `apps_list`). Read the tool's own description before calling it.

| Question | Tools |
| --- | --- |
| A Shopify webhook is not arriving | `webhook_routes_list` (is the topic routed and registered?), then `webhook_deliveries_list` (did it arrive, what did the handler answer?) |
| Shopify API deprecations affecting the app | `shopify_alerts_list`, `shopify_alert_get` |
| What a store is entitled to right now | `entitlements_explain` |
| CRM fields and segments | `custom_fields_list`, `crm_segments_list`, `crm_segment_get`, `custom_field_values_set` |
| Emails | `emails_list`, `email_get`, `emails_variables_list`, `email_create` and `email_update` (inactive drafts and metadata only) |
| Automations | `automations_catalog_get` first, then `automation_create` or `automation_create_from_template` (a catalog template with its starter emails, all inactive), `automation_update` (inactive only), `automation_test_run` (dry run, sends nothing) |

An agent never writes an email body: the only body is Meridian's own starter copy from `automation_create_from_template`. Activating an email or automation, live sends, prices and discounts stay in the dashboard. Say so instead of looking for a workaround.

## When something is refused

- "App not found, or you do not have access to it" means a wrong `app_id` or a missing permission. The two read the same on purpose. Re-run `apps_list`, then ask the user to check their role.
- `402 PLAN_LIMIT_REACHED` from the SDK is the developer's own Meridian plan quota, not the merchant's. Never show it to a merchant.
- A CLI error names its fix in the `›` hint line. Read it before retrying.
