# The SDK: plans, pricing, gating, usage

`@the-meridian/sdk` goes inside the Shopify app. It targets the Shopify React Router app template (Remix-shaped loaders work the same way). Two entry points:

- `@the-meridian/sdk`: browser code (`MeridianProvider`, `useMeridian`, `MeridianPricingPage`, `MeridianAccountPage`).
- `@the-meridian/sdk/server`: Node only (`createMeridianApp`, webhook verification). The only place the API key is used.

Before writing code, get the real keys: the MCP prompt `integrate_sdk` returns a walkthrough pre-filled with the app's features, events and plans (optional `surface`: `provider`, `pricing`, `usage` or `server`). Or call `features_list`, `events_list` and `plans_list`. For any detail, `docs_get` on `sdk-quickstart.mdx`, `sdk-app.mdx`, `sdk-use-meridian.mdx` or `sdk-errors.mdx` (without MCP, the same pages as `.md` on help.the-meridian.ai).

## Credentials

| Variable | Where it comes from |
| --- | --- |
| `MERIDIAN_APP_ID` | Dashboard, the app's Settings, SDK page. Not a secret. |
| `MERIDIAN_API_KEY` | Same page, **Create API key** (shown once, starts `mrd_sk_`). Server only. The user creates it and puts it in `.env` themselves. |
| `MERIDIAN_WEBHOOK_SECRET` | Only to verify forwarded webhooks. Pub/Sub, Webhooks page. |
| `MERIDIAN_API_URL` | Leave unset. |

On Meridian hosting these are injected at every deploy (check with `managed_variables_list`), so they only need to be in the local `.env`.

## Wire it in

1. `npm install @the-meridian/sdk`
2. `app/meridian.server.ts`:

   ```ts
   import { createMeridianApp } from "@the-meridian/sdk/server"

   export const meridian = createMeridianApp()
   ```

   It never throws. Unconfigured, `meridian.configured` is `false` and every method returns its empty result.
3. `app/shopify.server.ts`, in `shopifyApp({ hooks })`:

   ```ts
   afterAuth: async ({ session, request }) => {
     await meridian.afterAuth(session, { request })
   },
   ```

   This registers the webhooks, reports the install (automations fire), syncs Shopify subscriptions and trials, and attributes a referral.
4. `app/routes/app.tsx` loader: `const shopToken = await meridian.mintShopToken({ shop: session.shop })`, returned to the component. `null` means "render without Meridian", not an error.
5. Same file, inside `<AppProvider embedded>`: `shopToken ? <MeridianProvider shopToken={shopToken}><Outlet /></MeridianProvider> : <Outlet />`.
6. Pricing route `app/routes/app.pricing.tsx`: build `returnUrl` in the loader as `` `https://${session.shop}/admin/apps/${process.env.SHOPIFY_API_KEY}/app/pricing` `` and render `<s-page><s-section><MeridianPricingPage returnUrl={returnUrl} /></s-section></s-page>`.
7. Account route: `<MeridianAccountPage onChangePlan={() => navigate("/app/pricing")} />`.

## Gate and track

```tsx
const { isEnabled, getLimit, loading, track } = useMeridian()
if (loading) return null
if (!isEnabled("advanced_reports")) return <UpgradePrompt />
await track({ eventKey: "api_calls", quantity: 1, idempotencyKey: requestId })
```

Server side: `meridian.track({ shop, eventKey, quantity, idempotencyKey })`, or `meridian.trackMany({ shop, events })` for up to 50 events in one request. `meridian.gatesForShop(shop)` returns the gates for a loader or a job.

## Pitfalls

- **Billing needs Meridian hosting.** `subscribe`, `cancelSubscription` and `updateUsageCap` work only when the app runs on Meridian with a managed database, or locally under `meridian dev`. Otherwise they fail with `403 PLAN_BUILDER_UNAVAILABLE` and `err.details.plan_builder_billing_status` names the cause. Two things fail quietly instead: usage overage is never charged and prices fall back to USD. See `sdk-hosting.mdx`.
- **SQLite breaks billing** with `session_table_missing`. Prisma must use MySQL (`references/cli.md`).
- **`returnUrl`** must be the `https` shop-admin URL of the pricing route itself. One built from `request.url`, `window.location` or the app's own origin is rejected (`INVALID_RETURN_URL`).
- **Gates fail open.** While `loading` is true, or after an error, everything reads as enabled. Check `loading` first, or paying merchants see an upgrade prompt flash. An unknown key reads `false` with no error.
- **Keys, not ids.** `track` and `updateUsageCap` take the event key. A UUID is refused with 422.
- **Idempotency.** Pass `idempotencyKey` on every `track`: retries on 429 and 5xx only happen with one. A backdated `timestamp` requires one.
- **Rate limits.** Reads and tracking share 120 requests a minute per shop token, billing writes 30. Batch with `trackMany`, never `Promise.all` over `track`.
- **The pages need App Bridge.** The pricing and account pages render Polaris web components, so they live in the embedded admin.
- **Serverless hosting** starts every instance with a cold token cache. Persist the token with `mintShopTokenDetailed`.
- **Webhook topics** added in the dashboard reach installed stores at their next `afterAuth`. `meridian.registerWebhooks({ shop, accessToken })` applies them at once.
- **Forwarded webhooks**: verify with `verifyMeridianWebhookDetailed` against the raw body, and take the shop and topic from what it returns, not from a header. See `sdk-webhooks.mdx`.
- **`402 PLAN_LIMIT_REACHED`** is the developer's Meridian quota. Log it, never show it to a merchant.
