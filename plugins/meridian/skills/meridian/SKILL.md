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
   - `@the-meridian/sdk` is in `package.json` and `app/meridian.server.ts` exists (the SDK is wired)

## Pick the workflow

| The user wants to | Read |
| --- | --- |
| Install the CLI, connect MCP, sign in, link the project | [references/setup.md](references/setup.md) |
| Add plans, pricing, gating or usage tracking (the SDK) | [references/sdk.md](references/sdk.md) |
| Run the app locally on a Meridian database, migrate, seed | [references/cli.md](references/cli.md) |
| Deploy, follow a deploy, fix a failed deploy, set variables | [references/hosting.md](references/hosting.md) |
| Work on webhooks, Shopify API alerts, CRM, emails, automations | "Operate and grow" below |

A new app usually goes through all four references in that order: setup, SDK, local development, deploy.

## Rules that always apply

**The human does these, never you.** Ask the user, explain why, and wait:

- Signing in: `meridian login` runs a device-code flow the user approves in their browser. Ask them to run it in their own terminal, then check with `meridian whoami --json`.
- Approving the MCP connection (the OAuth consent screen in their browser).
- Creating or rotating `MERIDIAN_API_KEY` (Settings, SDK in the dashboard) and putting it in `.env` or the hosting secrets. Never ask them to paste a key, a password or a secret into the chat.
- Entering hosting secrets and variables, installing the Meridian GitHub app, adding a custom domain, choosing a plan.

**Confirm in chat before you do these**, naming exactly what will happen:

- Deploying to production, promoting to production, or `deployment_rollback`.
- `meridian db reset`, `dev_database_reset`, or anything else that drops data.
- `automation_update` on an automation that is already active, or `custom_field_values_set` on a real store.

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
| Automations | `automations_catalog_get` first, then `automation_create`, `automation_update` (inactive only), `automation_test_run` (dry run, sends nothing) |

Email bodies, activating an email or automation, live sends, prices and discounts stay in the dashboard. Say so instead of looking for a workaround.

## When something is refused

- "App not found, or you do not have access to it" means a wrong `app_id` or a missing permission. The two read the same on purpose. Re-run `apps_list`, then ask the user to check their role.
- `402 PLAN_LIMIT_REACHED` from the SDK is the developer's own Meridian plan quota, not the merchant's. Never show it to a merchant.
- A CLI error names its fix in the `›` hint line. Read it before retrying.
