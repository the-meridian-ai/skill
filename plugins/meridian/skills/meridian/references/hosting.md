# Deploying on Meridian hosting

Meridian builds the app from a commit of its connected GitHub repository, runs it on Cloud Run, and puts it behind its own edge (`*.the-meridian.app` or a custom domain). Deploys come only from GitHub: a push to the auto-deploy branch, a manual deploy of a branch and commit (dashboard or MCP `deployment_create`), a promote, or a rollback. There is no CLI deploy command.

Each app has a Production environment. Development is optional on every plan, Staging is Enterprise only. Each environment has its own service, database, variables and add-ons.

## Readiness checklist

Check each item in the repository before the first deploy, and fix what fails:

1. **`Dockerfile` at the repository root.** A deploy is refused without one ("Dockerfile not found at root of repository"). The Shopify template's Dockerfile works. The build runs with `NODE_ENV=production`.
2. **Prisma on MySQL** with `url = env("DATABASE_URL")` (`references/cli.md`). Meridian injects `DATABASE_URL` for the environment's managed database, plus the `DB_*` variables. A `DATABASE_URL` the user sets wins.
3. **Migrations in the release command**, not at startup. The environment's release command should be `npx prisma migrate deploy`. It runs once per deploy in the new image, before traffic moves, with a 15 minute limit, and a non-zero exit fails the deploy while the old revision keeps serving. Then remove `prisma migrate deploy` from the `docker-start` script and run `prisma generate` at build time, which cuts the cold start from over ten seconds to a few.
4. **A health check route answering 200** on the environment's health check path (usually `/health`), with no redirect and no authentication. A redirect counts as a failure. A minimal React Router route: `app/routes/health.tsx` exporting `export const loader = () => new Response("ok")`.
5. **The container port** set on the environment matches the port the server listens on (the template serves on `3000`, or reads `PORT`).
6. **The host header.** The edge passes the public hostname in `X-Forwarded-Host`. Code that reads the request host reads that header. React Router form actions need no `allowedActionOrigins` for Meridian hostnames.

## What the user sets up in the dashboard

You cannot do these. List the missing ones for the user:

- Connect the repository: Settings, Integrations, GitHub card. It installs the Meridian GitHub app.
- Add the database: Hosting, Database (MySQL, up to five minutes to provision).
- Set the environment's release command, health check path and container port.
- Add their own variables and secrets on the Environment page (write-only secrets, scoped to all environments or one). Meridian injects `SHOPIFY_API_KEY`, `SHOPIFY_API_SECRET`, `MERIDIAN_APP_ID`, `MERIDIAN_API_URL`, `MERIDIAN_API_KEY`, `MERIDIAN_WEBHOOK_SECRET`, `DB_*` and the add-on variables itself.
- Choose the auto-deploy branch and target.

Variables, secrets, the release command and the health check path reach the app with the next deploy. `environments_list` shows `config_pending_deploy: true` until then.

## Deploy and follow it

1. `apps_list`, then `environments_list` for the `environment_id`.
2. `managed_variables_list` and `secrets_list` to check every variable the code reads exists (names only, never values).
3. `github_refs_list` for a pushed `git_sha`. A commit that is not pushed fails late in the build.
4. Confirm with the user, then `deployment_create` with the environment and the sha (or the user pushes to the auto-deploy branch). Production needs the **hosting.promote** permission.
5. Poll `deployment_get`. Status runs Queued, Building, Deploying (the release command, then the new revision), Verifying (the health check), then Succeeded, Failed or Cancelled. One deployment is active per environment.
6. After it succeeds, `logs_tail` for runtime errors (`min_severity: "WARNING"`, `contains`, `since`).

## When a deploy fails

Run the MCP prompt `debug_failed_deploy` with the deployment: it returns the real error, the recent build events and the calls to make next. Common causes:

| Failed at | Usual cause |
| --- | --- |
| Building | The Dockerfile build fails (a missing dependency, a build script that needs a variable a build cannot see) |
| Running release command | The migration fails. Read its output in the deploy log |
| Verifying | The health check path redirects, needs auth, answers non-200, or the container port is wrong |
| Runtime errors after success | A missing variable (`managed_variables_list`, `secrets_list`) or a schema mismatch (`database_schema_get`) |

`deployment_rollback` sends traffic back to an earlier succeeded deployment with no rebuild. Ask the user before you call it.

## Pull request previews

A pull request from a branch of the connected repository gets a preview deployment (two live at most per app, removed when the PR closes). A preview gets only the plain all-environment variables, never secrets or managed variables. `previews_list` gives its URL and status.
