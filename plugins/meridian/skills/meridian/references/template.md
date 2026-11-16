# Starting from the Meridian Shopify app template

The project was made from the Meridian template (`the-meridian-ai/shopify-app-template`) when it has an `app/meridian/` folder with `meridian.server.ts`, and its README is titled "Shopify app template for Meridian". It is Shopify's React Router template with Prisma on MySQL, the SDK wired in (`afterAuth`, the shop token, the provider), a `/health` route, two optional examples and these agent files. Setup, the SDK wiring and the Prisma switch to MySQL are already done: skip those parts of the other references.

## Ask first, one question at a time

The template still holds placeholders. Before changing code, ask the developer these questions in order, each with its default, and wait for each answer. Skip a question the project already answers (a filled client ID in `shopify.app.toml`, a `.meridian/project.json`).

1. **What does the app do, in one sentence?** Use it for the app's name in `shopify.app.toml`, the landing page (`app/routes/_index/route.tsx`, still placeholder text) and the first line of `AGENTS.md`.
2. **Is there already a Shopify app for it, or should one be created?** Either way, the developer runs `shopify app config link` in their terminal (it signs in to Shopify and asks which app). It fills the client ID in `shopify.app.toml`.
3. **Is the app already on Meridian?** Check with `apps_list`. If not, the developer creates it in the dashboard (Create app) with the Shopify app's client ID and secret, and connects the GitHub repository. Then link the project: `meridian link --org <org-slug> --app <app-slug>`.
4. **Keep the examples, adapt them, or remove them?** Default: keep. The template gates a feature keyed example_feature and tracks an event keyed product_generated (`app/meridian/examples.server.ts`). Keeping or adapting them means the keys must exist in the app's catalog: read `features_list` and `events_list`, and create what is missing with `feature_create` and `events_create` after the developer agrees on the key and name. Removing them means deleting `examples.server.ts`, `ExampleGate.tsx` and the lines marked "Meridian example" in `app/routes/app._index.tsx`.
5. **Paid plans now or later?** Default: later. Now means drafting plans with `plan_create`. Activating one is the developer's decision, and the template has no pricing page yet (`references/sdk.md`, step 6, adds one).

## Then run it and deploy it

1. Locally: `npm install`, then `meridian dev` in the background. The first run provisions the developer database (up to five minutes) and applies the template's migration.
2. Readiness: `environments_list`, then `app_readiness_get` for production. On a new app, the developer does the dashboard steps in `references/hosting.md`; for this template they are, at least:
   - add a database to the environment;
   - set the release command to `npx prisma migrate deploy` (the health check path stays `/health` and the container port stays empty, which means 3000).

   Meridian injects SCOPES from scopes in `shopify.app.toml` at the deployed commit, so do not add it. A plans warning (no active plan) is expected until the developer activates one, and so is the boot variables warning that SHOPIFY_APP_URL differs from the app URL in `shopify.app.toml`, until step 4 points that URL at the Meridian address.
3. Deploy: confirm with the developer, then `deployment_create` with a pushed commit, or `meridian deploy --watch`, which to production also needs production access the developer approves in the browser, and `--yes` once they agreed ("Production changes from the CLI" in `SKILL.md`). Follow it with `deployment_get` (`references/hosting.md`).
4. Shopify side, after the deploy succeeds and the project is linked (question 2): set the app URL and redirect URLs in `shopify.app.toml` to the environment's address. Linking replaces the whole file with the Dev Dashboard's copy, so edit it after linking, never before (from `environments_list`), then the developer runs `shopify app deploy` from the same commit, with `APP_URL=<that address>` when an extension calls the backend.

## Keep the template's shape

- Prisma stays on MySQL unless the app's database is PostgreSQL. Do not switch it because an example online uses another engine: MySQL is the default engine and the one developer databases use, and SQLite loses sessions on every deploy.
- New Meridian code goes in `app/meridian/`, and lines added to Shopify's files carry a "Meridian" comment, so the developer can still remove Meridian or pull Shopify's template updates.
- Configuration stays in environment variables. Do not hardcode a URL, an id or a key.
