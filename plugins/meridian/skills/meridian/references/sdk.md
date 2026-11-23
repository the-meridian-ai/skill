# The SDK: plans, pricing, gating, usage

`@the-meridian/sdk` goes inside the Shopify app. It targets the Shopify React Router app template (Remix-shaped loaders work the same way). Two entry points:

- `@the-meridian/sdk`: browser code (`MeridianProvider`, `useMeridian`, `MeridianPricingPage`, `MeridianAccountPage`).
- `@the-meridian/sdk/server`: Node only (`createMeridianApp`, webhook verification). The only place the API key is used.

Before writing code, get the real keys: the MCP prompt `integrate_sdk` returns a walkthrough pre-filled with the app's features, events and plans (optional `surface`: `provider`, `pricing`, `usage` or `server`). Or call `features_list`, `events_list` and `plans_list`. For any detail, `docs_get` on `sdk-quickstart.mdx`, `sdk-app.mdx`, `sdk-use-meridian.mdx` or `sdk-errors.mdx` (without MCP, the same pages as `.md` on help.the-meridian.ai).

## Credentials

| Variable | Where it comes from |
| --- | --- |
| `MERIDIAN_APP_ID` | Dashboard, the app's Settings, SDK page. Not a secret. Under `meridian dev`, injected for you. |
| `MERIDIAN_API_KEY` | Same page, **Create API key** (shown once, starts `mrd_sk_`). Server only. For local development `meridian dev` injects a dev key limited to dev stores, so nothing goes in `.env`. For production the user creates one and sets it as a hosting secret themselves. A value already in the shell or `.env` wins, and then `meridian dev` injects none of the three. |
| `MERIDIAN_WEBHOOK_SECRET` | Only to verify forwarded webhooks. Pub/Sub, Webhooks page. |
| `MERIDIAN_API_URL` | Leave unset. `meridian dev` sets it only when it injects the key (staging for a CLI logged in on staging). |

On Meridian hosting these are injected at every deploy (check with `managed_variables_list`). Locally, run the dev server through `meridian dev`, which injects them (see the CLI reference), instead of copying them into `.env`.

## Wire it in

1. `npm install @the-meridian/sdk`
2. `app/meridian.server.ts`:

   ```ts
   import { createMeridianApp } from "@the-meridian/sdk/server"

   export const meridian = createMeridianApp()
   ```

   It never throws. Unconfigured, `meridian.configured` is `false`. In SDK 3.4.0 and later, construction warns once per process, including in production, naming the missing credentials. Methods return their empty result, or `status: "unconfigured"` for `trackDetailed` and `trackManyDetailed`.
3. `app/shopify.server.ts`, in `shopifyApp({ hooks })`:

   ```ts
   afterAuth: async ({ session, request }) => {
     await meridian.afterAuth(session, { request })
   },
   ```

   This registers the webhooks, reports the install (automations fire), syncs Shopify subscriptions and trials, and attributes a referral.
4. `app/routes/app.tsx` loader: `const shopToken = await meridian.mintShopToken({ shop: session.shop })`, returned to the component with `meridian.baseUrl`. `null` means "render without Meridian", not an error.
5. Same file, inside `<AppProvider embedded>`: `shopToken ? <MeridianProvider shopToken={shopToken} baseUrl={baseUrl} getShopToken={getShopToken}><Outlet /></MeridianProvider> : <Outlet />`. Without `baseUrl`, the browser calls the production API, so an app on any other Meridian platform gets "Invalid shop token". `getShopToken` (SDK 3.7.0 and later) runs when Meridian refuses the current token: it POSTs `{ refusedShopToken }` with a plain `fetch` to `/app/meridian/shop-token` (App Bridge adds its session token to `fetch` calls to the app's own origin), and resolves `null` on any non-200 answer or a body without a string `shopToken`. The route, `app/routes/app.meridian.shop-token.tsx`, has an action only. It runs `authenticate.admin(request)` first, before it reads the body or mints anything, because that check is what keeps the store's token from anyone without access to the store. Then it answers `{ shopToken }` from `meridian.mintShopToken({ shop: session.shop, refusedShopToken })` with `Cache-Control: no-store`, or `503` when the mint returns `null`. A missing or non-string `refusedShopToken` is a plain mint. See `sdk-app.mdx` for the code. With the route in place, `createMeridianApp({ browserTokenRefresh: false })` limits a token copied from a page to its hour. Never turn it off without `getShopToken`: the provider then shows `SHOP_TOKEN_REFRESH_DISABLED` once the token's hour ends. The Meridian template does both. See `sdk-provider.mdx`.
6. Pricing route `app/routes/app.pricing.tsx`: build `returnUrl` in the loader as `https://<shop>/admin/apps/<client id>/app/pricing`, where `<shop>` is `session.shop` and `<client id>` is the app's Shopify client ID (public, already in `shopify.app.toml`), and render `<s-page><s-section><MeridianPricingPage returnUrl={returnUrl} /></s-section></s-page>`.
7. Account route: `<MeridianAccountPage onChangePlan={() => navigate("/app/pricing")} />`.

## Gate and track

```tsx
const { isEnabled, getLimit, loading, track } = useMeridian()
if (loading) return null
if (!isEnabled("advanced_reports")) return <UpgradePrompt />
await track({ eventKey: "api_calls", quantity: 1, idempotencyKey: requestId })
```

Server side: `meridian.track({ shop, eventKey, quantity, idempotencyKey })`, or `meridian.trackMany({ shop, events })` for up to 50 events in one request. `meridian.gatesForShop(shop)` returns the gates for a loader or a job. In SDK 3.5.0 and later it keeps each shop's entitlements in memory for 30 seconds, because every read is a metered SDK request: `entitlementsCacheTtlMs: 0` on `createMeridianApp` turns that off, and `gatesForShop(shop, { fresh: true })` reads past it (the billing return route). A plan change made in the browser shows on the server within those 30 seconds. See `sdk-app.mdx`.

## Pitfalls

- **Confirm tracking.** Without credentials, `track` returns `null` and logs nothing per call. Check `meridian.configured` or use `trackDetailed` (SDK 3.4.0 and later), and never report an event as counted without `status: "recorded"`.
- **Billing needs Meridian hosting.** `subscribe`, `cancelSubscription` and `updateUsageCap` work only when the app runs on Meridian with a managed database, or locally under `meridian dev`. Otherwise they fail with `403 PLAN_BUILDER_UNAVAILABLE` and `err.details.plan_builder_billing_status` names the cause. Two things fail quietly instead: usage overage is never charged and prices fall back to USD. See `sdk-hosting.mdx`.
- **SQLite breaks billing** with `session_table_missing`. Prisma must use the app's database engine, MySQL or PostgreSQL (`references/hosting.md`).
- **`returnUrl`** must be the `https` shop-admin URL of the pricing route itself. One built from `request.url`, `window.location` or the app's own origin is rejected (`INVALID_RETURN_URL`).
- **Gates fail closed.** While the provider is loading, after a failed first load, when Meridian is unreachable or rate-limits the app, and without credentials, every feature reads as off. In SDK 3.5.0 and later, read `status` before showing an upgrade prompt: `"ok"` means a `false` is the plan's answer (a shop with no plan is `"ok"` too), `"loading"`, `"failed"`, `"rate_limited"` or `"unconfigured"` mean Meridian has not answered. `failOpen: true` on `createMeridianApp` or `MeridianProvider`, or per call (`can(key, { failOpen })`, `isEnabled(key, { failOpen })`), turns features on when Meridian failed or rate-limited the app, never without credentials and never while loading. Recommend it for paid capabilities, and keep features that cost money per use closed. An unknown key reads `false` with no error. See `features.mdx`.
- **Keys, not ids.** `track` and `updateUsageCap` take the event key. A UUID is refused with 422.
- **Idempotency.** Pass `idempotencyKey` on every `track`: retries on 429 and 5xx only happen with one. A backdated `timestamp` requires one.
- **Rate limits.** Reads and tracking share 120 requests a minute per shop token, billing writes 30. Batch with `trackMany`, never `Promise.all` over `track`.
- **The pages need App Bridge.** The pricing and account pages render Polaris web components, so they live in the embedded admin.
- **Serverless hosting** starts every instance with a cold token cache. Persist the token with `mintShopTokenDetailed`.
- **Webhook topics** added in the dashboard reach installed stores at their next `afterAuth`. `meridian.registerWebhooks({ shop, accessToken })` applies them at once.
- **Forwarded webhooks**: verify with `verifyMeridianWebhookDetailed` against the raw body, and take the shop and topic from what it returns, not from a header. See `sdk-webhooks.mdx`.
- **`402 PLAN_LIMIT_REACHED`** is the developer's Meridian quota. Log it, never show it to a merchant.
