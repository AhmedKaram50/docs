# Shopify integration + migration from Bulk Operations to Product Feeds

**Date**: 2026-05-11 → 2026-05-12

## Goal

Wire the existing Shopify app (`/home/ahmed/work/rastova/mab/shopify-app`) to
the mab-api so that:

1. On Shopify install, the merchant creates a dashboard account
   (email + password).
2. Account creation kicks off a sync of products / variants / collections /
   localizations / per-market prices into our catalog.

## Phase 1 — Initial Shopify install + account flow

### Server (mab-api) additions

- **`POST /v1/auth/shopify/callback`** — internal-secret-protected
  (`X-Internal-Secret` header). Body `{ shopifyDomain, accessToken, scopes? }`.
  - First call: creates `Shop` row, encrypts/binds Shopify access token,
    creates trial Subscription, returns short-lived **install JWT**
    (`kind:'install'`, 1h, audience `mab:install`) + `needsPasswordSetup:true`.
  - Subsequent calls: rotates the encrypted Shopify access token, returns full
    access + refresh JWT pair.
- **`POST /v1/auth/set-password`** — bearer install JWT. Body `{ email, password }`.
  - Creates `Account` (or attaches existing if password matches), `Membership`,
    sets `shop.ownerAccountId`, issues full tokens, **fires the sync trigger**.
- **`POST /v1/sync/trigger`** (later replaced by `/v1/feeds/resync`).

### New env var

- `INTERNAL_API_SECRET` (≥16 chars). Mirrored from `shopify-app/.env` so the
  callback can authenticate.

### `TokenService` extended

Added `issueInstallToken`, `verifyAnyToken`. Install tokens use audience
`mab:install`, claim `{shopId, kind:'install'}`.

### Shop repo fix

`DrizzleShopRepository.save` now persists `shopifyAccessTokenEncrypted` /
`shopifyAccessTokenIv` on conflict update (previously dropped on update path).

### New port: `SyncTrigger`

Late-bound proxy in `IdentityModule` so `set-password` can kick off catalog
provisioning without identity having to know about `shopify-sync`. Wired in
`shopify-sync.module.ts` after the start-sync use case is constructed.

### Shopify-app changes

- `app/routes/app.tsx` + `app._index.tsx`: switched all paths from
  `/api/v1/...` to `/v1/...` and read `body.accessToken` directly (API doesn't
  wrap responses in `{data: …}`).
- Resync button disabled when `needsPasswordSetup: true` (install token
  can't reach sync routes).

### Logging

`src/shared/infrastructure/logger/pino.ts` now writes to both stdout and
`logs/api.log` (pino multi-target). Used heavily for debugging downstream.

## Phase 2 — Bulk-operations sync hits the wall

The original `shopify-sync` module submitted Admin GraphQL `bulkOperationRunQuery`
per resource type (products / collections / markets / localizations /
collection_products), polled via webhook, downloaded JSONL, staged in
`external_entities`, then resolved + imported.

Two persistent failures surfaced:

1. **Field-removal whack-a-mole**.
   - `ProductVariant.requiresShipping` removed in Admin 2024-10+ → fixed by
     querying `inventoryItem.requiresShipping` instead, mapper updated to read
     from either path.
   - `translations` now requires a `locale:` argument, but bulk operations are
     one-shot so we can't iterate locales — hardcoded `"en"` as a stop-gap.
2. **Localization can't fit bulk-ops semantics** at all.

### Shopify API version aligned to `2026-01`

Was previously inconsistent across:

- mab-api `.env` `SHOPIFY_API_VERSION` → `2026-01`
- shopify-app `app/shopify.server.ts` → `ApiVersion.October25`
- shopify-app `shopify.app.toml` → `2026-04`

All three now `2026-01`.

## Phase 3 — Migration plan: Bulk Operations → Product Feeds

Cross-referenced the sister project at
`/home/ahmed/work/rastova/mobile-app-builder` (their
`packages/shopify-client/src/admin/product-feeds.ts`,
`apps/api/src/modules/webhooks/product-feed-webhook.service.ts`,
`apps/api/src/modules/sync/product-feed-setup.ts`, plus the 5-phase plan docs at
`docs/plans/shopify-sales-channel-product-feeds-sync/`).

Decision: **replace `shopify-sync` in-place** with a Product Feeds
implementation. Plan at
`/home/ahmed/.claude/plans/yes-bro-i-need-iterative-grove.md`.

### Schema

Migration `migrations/0004_product_feeds.sql`:

- **Drop**: `sync_runs`, `external_entities`, `external_entity_map`,
  `unresolved_references` + their enums (`sync_run_status`, `staging_status`,
  `unresolved_status`).
- **Create**:
  - `product_feeds(id, shop_id, shopify_feed_id, country, language, status,
    is_primary, last_full_sync_at, last_full_sync_id, …)` with unique
    `(shop_id, country, language)` and `(shop_id, shopify_feed_id)`.
  - `product_feed_syncs(id, shop_id, feed_id, shopify_full_sync_id, status,
    products_received, products_expected, started_at, completed_at, …)`.
  - Enums `product_feed_status` (ACTIVE / INACTIVE), `full_sync_status`
    (PENDING / IN_PROGRESS / COMPLETED / ERROR).
- Catalog tables already had `shop_markets`, `product_localizations`,
  `variant_market_prices` — reused as-is via the existing `CatalogUpsertPort`.

### Domain layer (rebuilt)

- `ProductFeed` aggregate — country/language VOs, `status` transitions,
  `recordFullSync(...)`, `markPrimary(...)`.
- `ProductFeedSync` aggregate — PENDING → IN_PROGRESS → COMPLETED / ERROR,
  `incrementReceived`, `complete`, `fail`.
- Repository interfaces in `domain/`, Drizzle impls in
  `infrastructure/persistence/`.

### Application layer

- **Ports**: `ShopifyFeedClient`, `FeedJobEnqueuer`, `Clock`.
- **Setup**: `RunFeedSetupUseCase` — install-time orchestration:
  1. Decrypt shop access token (AES-GCM with `SHOPIFY_TOKEN_ENCRYPTION_KEY`).
  2. `fetchMarkets` (Admin GraphQL `MARKETS_QUERY`).
  3. Upsert `shop_markets` via `CatalogUpsertPort.upsertShopMarkets`.
  4. Enumerate `(country, language)` pairs from each market's webPresences,
     plus a **shop primary locale fallback** so source-locale products still
     flow into markets configured only for non-source languages.
  5. List existing `productFeeds` from Shopify (idempotency); **adopt any
     pre-existing feeds** we didn't enumerate (e.g. feeds left behind by
     another integration on the same shop).
  6. For each pair → create or reuse `productFeed`, persist `ProductFeed` row.
  7. Subscribe 7 webhook topics idempotently.
  8. Trigger `productFullSync` per feed, persist `ProductFeedSync` row.
- **Webhook handlers**:
  - `HandleProductSyncUseCase` — handles CREATE / UPDATE (upsert product +
    variants + localization + per-market price) and DELETE (unpublish
    localization, drop product if no published localizations remain).
  - `HandleFullSyncFinishUseCase` — mark sync COMPLETED, update feed
    `lastFullSyncAt`, mark stale localizations unpublished
    (`syncedAt < sync.startedAt`).
  - `HandleFeedUpdateUseCase` — update feed status from feed-update payload.
  - `HandleMarketChangeUseCase` — re-runs feed setup on any market
    create/update/delete (conservative; per-action diffing deferred).
- **Status use cases**: `ListFeedsUseCase`, `GetFeedStatusUseCase`.
- **Parsers** (verbatim ports of mobile-app-builder):
  - `parseProductFeedPayload` — handles `requireShipping` / `requiresShipping`
    fallback, price-object → string flattening
    (`{amount: "10.00", currencyCode}` → `"10.00"`), unwraps GraphQL
    edges/nodes.
  - `parseFullSyncFinishPayload`, `parseFeedUpdatePayload`.

### Infrastructure layer

- `HttpShopifyFeedClient` — fetch-based Admin GraphQL client. Implements
  `fetchMarkets`, `listProductFeeds`, `createProductFeed`, `deleteProductFeed`,
  `triggerFullSync`, `listWebhookSubscriptions`, `subscribeWebhook`
  (idempotent: list-by-topic first, only create if missing).
- GraphQL queries in `shopify-feed.queries.ts` — `MARKETS_QUERY`,
  `PRODUCT_FEED_*`, `WEBHOOK_*`. Note: `WEBHOOK_SUBSCRIPTION_CREATE_MUTATION`
  uses the new `endpoint { ... on WebhookHttpEndpoint { callbackUrl } }` shape.
- `BullMqFeedEnqueuer` + `product-feed.worker.ts` — single BullMQ worker
  dispatching by job name (`feed.product-sync`, `feed.full-sync-finish`,
  `feed.feed-update`, `feed.market-{create,update,delete}`).

### HTTP routes

- `webhooks.routes.ts` — 7 routes under `/v1/shopify/webhooks/...`, each
  HMAC-verified, fire-and-forget enqueue, respond 200 (Shopify times out
  at 5s):
  - `POST /v1/shopify/webhooks/product-feeds/full-sync`
  - `POST /v1/shopify/webhooks/product-feeds/incremental-sync`
  - `POST /v1/shopify/webhooks/product-feeds/full-sync-finish`
  - `POST /v1/shopify/webhooks/product-feeds/update`
  - `POST /v1/shopify/webhooks/markets/create`
  - `POST /v1/shopify/webhooks/markets/update`
  - `POST /v1/shopify/webhooks/markets/delete`
- `feeds.routes.ts` — auth-gated `GET /v1/feeds`, `GET /v1/feeds/:id`,
  `POST /v1/feeds/resync` (replaces old `/v1/sync/trigger`).
- `verify-shopify-hmac.plugin.ts` — preserved from old module
  (raw-body content-type parser + base64 HMAC compare).

### Catalog port additions

`CatalogUpsertPort` extended with:

- `markStaleProductLocalizations(shopId, country, language, syncStartedAt)` —
  bulk-unpublish localizations not seen during a full sync.
- `unpublishProductLocalization(shopId, productId, country, language)`.
- `hasAnyPublishedProductLocalization(shopId, productId)`.
- `findProductIdByShopifyGid` / `findVariantIdByShopifyGid`.
- `setProductPublished(shopId, productId, isPublished)`.
- `listVariantIdsForProduct(shopId, productId)`.

Wired in `catalog.module.ts`. Backed by a new `Product.setPublished` repo
method and three new methods on `ProductLocalizationRepository`
(`markStaleUnpublished`, `markUnpublishedByProduct`, `hasAnyPublished`).

### Env changes

- Added `SHOPIFY_WEBHOOK_BASE_URL` (publicly reachable HTTPS root, e.g.
  `https://unfull-remissly-jax.ngrok-free.dev` from the user's stable
  ngrok tunnel `vector`).
- Removed: `JSONL_DOWNLOAD_TIMEOUT_MS`, `STAGING_RETENTION_DAYS`,
  `SYNC_BATCH_SIZE`, `SYNC_RESOLVE_BATCH_SIZE` (bulk-op leftovers).
- Updated `.env`, `.env.example`, `tests/setup.ts`, `tests/global-setup.ts`,
  `tests/helpers/db.ts` (truncate list).

### Tests

All bulk-op specs deleted. 63 tests passing, typecheck clean. New
integration/e2e specs for feeds were scoped in the plan but not added in
this session (deferred — happy-path verification ran against the live
test store).

## Phase 4 — Live debugging on `rastova-test.myshopify.com`

After deploying, hit a chain of issues uncovered via `logs/api.log`:

| # | Symptom | Cause | Fix |
|---|---|---|---|
| 1 | `relation "product_feeds" does not exist` | Migrations not applied to dev DB | `npm run db:migrate` |
| 2 | `webhookSubscriptionCreate: Address protocol http:// is not supported` | `SHOPIFY_WEBHOOK_BASE_URL=http://localhost:3001` | Switched to existing stable ngrok tunnel `vector` (`https://unfull-remissly-jax.ngrok-free.dev`) |
| 3 | `INVALID_HMAC` 401 on incoming webhooks | `SHOPIFY_WEBHOOK_SECRET=dev-webhook-secret-change-me` (Shopify signs with the **app**'s API secret) | Set `SHOPIFY_WEBHOOK_SECRET` to the value of `SHOPIFY_API_SECRET` from `shopify-app/.env` (`shpss_…`) |
| 4 | `productFeedCreate: shop doesn't support this country and language context` for some pairs (DZ/AR etc.) | Real — that market isn't configured for that locale | Already log-and-continue. Harmless. |

After all fixes, install + setup completed cleanly:

- 4 markets fetched, persisted to `shop_markets`.
- 6 feeds in DB (CA/AR, CA/EN, EG/AR, EG/EN, US/AR, US/EN) — EN ones adopted
  from a previous integration on the same shop (the "adopt pre-existing
  feeds" branch fired for all three).
- 7 webhooks subscribed.
- 6 `productFullSync` triggered.
- All 6 sync records reached `COMPLETED` via `full_sync_finish` webhooks.

## Phase 5 — `productsExpected: 0` mystery

Every full-sync-finish reported `productsExpected: 0` despite the test store
having 98 products. Initial diagnosis (web presences missing on the markets)
turned out to be **wrong** — corrected after the user shared the
[Shopify docs on contextual product feeds](https://shopify.dev/docs/apps/build/sales-channels/contextual-product-feeds).

### Actual root cause

`ProductFeed` only delivers products that are **published to the channel's
publication**. Verified via:

```graphql
{
  publication(id: "gid://shopify/Publication/120435572791") {
    name        # "Mobile App Builder 2"
    autoPublish # false
    products(first: 5) { edges { node { id } } }  # []
  }
}
```

→ zero products published to our app's publication, and `autoPublish: false`
so new products won't be added either.

### Audit against the docs

| Doc requirement | Status |
|---|---|
| `read_product_listings` access scope | ✅ in `shopify.app.toml` |
| Subscribe to `PRODUCT_FEEDS_FULL_SYNC` / `INCREMENTAL_SYNC` / `FULL_SYNC_FINISH` / `UPDATE` | ✅ done |
| Use `productFeedCreate` + `productFullSync` (manual mode) | ✅ done |
| Webhook payload field naming (`requireShipping`, `quantityAvailable`) | ✅ parser handles both spellings |
| **Products published to our channel's publication** | ❌ zero published |
| Channel config extension (`productFeedManagement = "automatic"`) | ❌ not set up — optional, manual mode is allowed |

### Fix options (parked at end of session)

- **A. Manual** — Shopify admin → Products → select all → Add to channels →
  check "Mobile App Builder 2" → save. Re-trigger sync.
- **B. Programmatic in `RunFeedSetupUseCase`** — query
  `currentAppInstallation { publication { id } }`, run
  `publicationUpdate(id, input: { autoPublish: true })`, iterate existing
  products and call `publishablePublish(id: productId, input: [{publicationId}])`.

User to decide before next session.

## Files created / modified

### Created
- `src/modules/shopify-sync/domain/product-feed/{product-feed.aggregate,product-feed-id.vo,product-feed-status.vo,product-feed.repository}.ts`
- `src/modules/shopify-sync/domain/product-feed-sync/{product-feed-sync.aggregate,product-feed-sync-id.vo,full-sync-status.vo,product-feed-sync.repository}.ts`
- `src/modules/shopify-sync/application/setup/run-feed-setup.usecase.ts`
- `src/modules/shopify-sync/application/webhooks/{handle-product-sync,handle-full-sync-finish,handle-feed-update,handle-market-change}.usecase.ts`
- `src/modules/shopify-sync/application/parsers/{product-feed-payload,full-sync-finish-payload,feed-update-payload}.parser.ts`
- `src/modules/shopify-sync/application/status/{list-feeds,get-feed-status}.usecase.ts`
- `src/modules/shopify-sync/application/ports/{shopify-feed-client,feed-job-enqueuer,clock}.port.ts`
- `src/modules/shopify-sync/infrastructure/persistence/{product-feed,product-feed-sync}.drizzle-repo.ts`
- `src/modules/shopify-sync/infrastructure/persistence/mappers/{product-feed,product-feed-sync}.mapper.ts`
- `src/modules/shopify-sync/infrastructure/persistence/schema/{product-feeds,product-feed-syncs,_enums}.schema.ts`
- `src/modules/shopify-sync/infrastructure/adapters/shopify/{shopify-feed.client,shopify-feed.queries}.ts`
- `src/modules/shopify-sync/infrastructure/adapters/bullmq-feed-enqueuer.adapter.ts`
- `src/modules/shopify-sync/infrastructure/workers/product-feed.worker.ts`
- `src/modules/shopify-sync/interfaces/http/{webhooks.routes,feeds.routes}.ts`
- `src/modules/shopify-sync/interfaces/http/plugins/verify-shopify-hmac.plugin.ts`
- `src/modules/identity/application/auth/{shopify-callback,set-password}.usecase.ts`
- `src/modules/identity/application/ports/sync-trigger.port.ts`
- `migrations/0004_product_feeds.sql`
- `docs/sessions/2026-05-12-shopify-product-feeds-migration.md` (this file)

### Rewritten
- `src/modules/shopify-sync/shopify-sync.module.ts` (composition root)
- `src/modules/identity/identity.module.ts` (returns `IdentityModule` with `setSyncTrigger` setter)
- `src/modules/identity/interfaces/http/auth.routes.ts` (added shopify-callback + set-password routes)
- `src/modules/identity/interfaces/http/schemas/auth.schema.ts`
- `src/shared/infrastructure/crypto/jwt.service.ts` (install token support)
- `src/shared/infrastructure/queue/workers/index.ts` (registers product-feed worker)
- `src/shared/infrastructure/logger/pino.ts` (file + stdout)
- `src/modules/identity/infrastructure/persistence/shop.drizzle-repo.ts` (persist Shopify token columns on update)
- `src/modules/catalog/application/ports/catalog-upsert.port.ts` (new methods)
- `src/modules/catalog/catalog.module.ts` (wire new port methods)
- `src/modules/catalog/domain/product/product.repository.ts` (`setPublished`)
- `src/modules/catalog/infrastructure/persistence/product.drizzle-repo.ts`
- `src/modules/catalog/domain/product-localization/product-localization.repository.ts`
- `src/modules/catalog/infrastructure/persistence/product-localization.drizzle-repo.ts`
- `src/config/env.ts`, `.env`, `.env.example`
- `tests/global-setup.ts`, `tests/setup.ts`, `tests/helpers/db.ts`
- `mab/shopify-app/app/routes/{app,app._index}.tsx` (path + token-shape fixes)
- `mab/shopify-app/app/shopify.server.ts` (`ApiVersion.January26`)
- `mab/shopify-app/shopify.app.toml` (`api_version = "2026-01"`)

### Deleted
- `src/modules/shopify-sync/{domain/sync-run,domain/external-entity,domain/external-entity-map,domain/unresolved-reference,domain/events}/`
- `src/modules/shopify-sync/application/{start-sync,pipeline,maintenance,status,webhooks,ports}/` (all bulk-op)
- `src/modules/shopify-sync/infrastructure/persistence/{sync-run,external-entity,external-entity-map,unresolved-reference}.drizzle-repo.ts`
- `src/modules/shopify-sync/infrastructure/adapters/shopify/{shopify-http.client,shopify.queries,shopify.mappers,shopify.types,jsonl-parser}.ts`
- `src/modules/shopify-sync/sync.constants.ts`
- `src/modules/shopify-sync/interfaces/http/{webhook,sync}.routes.ts` (old)
- `tests/shopify-sync/{domain,integration}/*` (old bulk-op specs)

## Verification

- `npm run typecheck` — clean.
- `npm test` — 63 / 63 pass.
- Live install on `rastova-test.myshopify.com`: setup completes, all 6
  syncs reach COMPLETED, but `productsExpected: 0` until products are
  published to the app's publication (Phase 5 above).

## Open follow-ups

1. **Publish products to our app's publication** (option A or B from Phase 5).
2. `RunFeedSetupUseCase` heuristic for `shop_market.isPrimary` is naive
   (`m.type === 'PRIMARY' || m.status === 'PRIMARY'`) — verify against
   real Admin response shapes.
3. `HandleProductSyncUseCase.resolveMarketCurrency` is stubbed to `'USD'`.
   Add `findMarketCurrencyByCountry(shopId, country)` on
   `ShopMarketRepository` + expose via `CatalogUpsertPort`.
4. `HandleMarketChangeUseCase` re-runs the entire setup on any market
   change. Tighten to per-action diffing later.
5. Migration `0004` snapshot was hand-copied from `0003` (drizzle-kit
   prompts interactively and we couldn't pipe answers). Regenerate the
   snapshot from a clean state before adding migration `0005`.
6. New integration / e2e specs for feed handlers + webhook routes were
   scoped in the plan but not implemented this session.
7. Stale 404s in the log from a different integration's old webhook
   subscription (`/api/v1/webhooks/shopify`). Harmless; clean up via
   `webhookSubscriptionDelete` if it's noise.

## Reference

- Plan: `~/.claude/plans/yes-bro-i-need-iterative-grove.md`
- Reference implementation: `/home/ahmed/work/rastova/mobile-app-builder`
- Shopify docs: <https://shopify.dev/docs/apps/build/sales-channels/contextual-product-feeds>
