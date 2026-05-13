# Shopify Sales Channel — Catalog Sync (Products, Collections, Markets, Languages)

Extracted from Shopify Dev Assistant (shopify.dev) Q&A session.

## Sources

- [Apps as sales channels — architecture & product feeds](https://shopify.dev/docs/apps/build/sales-channels)
- [About Shopify Markets — concepts & Admin API objects](https://shopify.dev/docs/apps/build/markets)
- [Building localized experiences with the Storefront API & Markets](https://shopify.dev/docs/storefronts/headless/building-with-the-storefront-api/markets)
- [About catalogs with the Markets API (preview)](https://shopify.dev/docs/apps/build/markets/new-markets/catalogs)
- [Contextual product feeds](https://shopify.dev/docs/apps/build/sales-channels/contextual-product-feeds)
- [Managing channel connections](https://shopify.dev/docs/apps/build/sales-channels/channel-connections)
- [Options to sync product data](https://shopify.dev/docs/apps/build/sales-channels/product-sync)

---

## 1. Best way to fully sync catalog with markets & languages

For a sales channel that keeps its own catalog fully in sync with Shopify across markets and languages, lean on:

- Shopify's **Sales Channel + Product Feed pattern**
- **Shopify Markets** (regions / pricing)
- **Translations / localization APIs** (languages)
- **Webhooks + incremental sync**, not polling

### 1.1 High-level architecture

A proper sales channel app works like this:

1. **Install & connect the merchant account**
   - Merchant installs your app from Shopify.
   - OAuth, then connect the external account using the sales channel app flow.

2. **Create a channel connection and product feeds**
   - Call `channelCreate` (Admin GraphQL) to create a **channel connection** for the merchant.
   - Each channel connection defines **regions/markets** it serves; Shopify exposes **product feeds** for that configuration.
   - Your app **listens to those feeds** to stay in sync, instead of manually polling Admin API for every product.

3. **Your system becomes a read-optimized mirror of the Shopify catalog**
   - On initial setup, consume the full product feed (or run Admin GraphQL queries once to backfill).
   - Stay updated via:
     - Product feeds (if you fully embrace sales-channel model), or
     - Webhooks (`products/create`, `products/update`, `collections`, `inventory_levels`, `price_lists`, etc.).

4. **Maintain a per-market & per-language catalog**
   - For each **market**:
     - Which products/variants are **published** to that market.
     - Local prices (via Markets + PriceLists).
     - Availability/restrictions (shipping countries, currencies).
   - For each **language**:
     - Pull translations from Shopify, or
     - Manage translations separately but link to the right Shopify product/variant IDs.

> Key idea: **Shopify is source-of-truth; you keep an incremental, region-/language-aware mirror.**

### 1.2 Markets (regions, currencies, price lists)

For each Market sync:

- Market id, name, primary country/region, currencies.
- Which **products & variants** are available to that market.
- The **price list** that applies to that market (international pricing).
- Restrictions (only some collections, different prices by region).

Model:

- `Market` (mirrors Shopify Market)
- `ExternalCatalog` / `ExternalFeed` per `Market`
- `ExternalProductVariant` keyed by `(shopifyProductId, shopifyVariantId, marketId)`

Use Admin GraphQL Markets APIs to:

- Fetch list of markets.
- Get pricing info (via `PriceList` objects associated with a Market).
- Determine what to show in your channel per market.

### 1.3 Languages (translations / localization)

Two main options:

**Option A — Admin GraphQL Translations**

- Ask Shopify for translations of product/collection resources in a given language.
- Store keyed by `(resourceId, locale)`.
- Pros: strong fidelity with merchant admin; supports any configured language.
- Cons: more complex querying/storage.

**Option B — Storefront API as translation source (read-only)**

- Use Storefront GraphQL with `@inContext(language: X, country: Y)` to fetch localized titles/descriptions.
- Pros: very straightforward; respects all Shopify localization behavior.
- Cons: per-storefront tokens + rate limits; read-only.

For mostly-read channels, Option B works; for tight admin-workflow integration, Option A.

---

## 2. Best practices — initial sync of all markets, languages, currencies, products, collections

Initial sync must be:

- **Incremental** (resumable on failure)
- **Rate-limit aware**
- **Idempotent** (safe to re-run)
- **Structured around Markets and languages**

### 2.1 Overall strategy

**Treat initial sync as a pipeline, not a one-shot script.**

Stages:

1. Fetch base entities (products, variants, collections).
2. Fetch Markets and price lists.
3. Fetch listings/availability per Market.
4. Fetch localized content per language.
5. Materialize your own "external catalog" objects.

Each stage records **progress checkpoints** (e.g. last cursor synced, last product ID) so you can resume after:

- Job crash
- Rate-limit hit
- Large catalog (tens/hundreds of thousands of products)

**Do initial sync + continuous sync:**

- Initial sync only creates a baseline.
- Switch quickly to event/feeds-based updates:
  - Product feeds (sales channel using `channelCreate`)
  - Webhooks for products/collections/markets/price-lists.

### 2.2 Scope & ordering (what to sync first)

1. **Shop metadata** — store id, primary domain, primary locale, primary currency.
2. **Markets & pricing model** — all Markets (countries/regions, currencies); PriceLists linked to Markets if international pricing enabled.
3. **Base catalog (language-agnostic, market-agnostic)** — products, variants, options; collections (smart & custom) and membership.
4. **Market availability & catalog shape per Market** — which products/variants available per market; price list applied per market.
5. **Languages & translations** — full set of languages/locales; per product/collection, localized titles, descriptions, SEO fields per language.

> Markets and pricing define *where and how* the catalog is visible. Base objects are the foundation. Languages and prices are *attributes* on top.

### 2.3 Pagination, rate limiting, batching

**Cursor-based pagination everywhere:**

- Always request `first: N` (50, 100, 250).
- Use `pageInfo { hasNextPage, endCursor }` to page.
- Store last `endCursor` per job step for resume.
- For large catalogs use chunks (50–100 typical; 250 max for many connections).

**Throttling and backoff:**

- Respect API rate limits.
- Monitor `X-Shopify-Shop-Api-Call-Limit` (REST) / GraphQL cost; back off when close to limits.
- Exponential backoff + jitter on 429 / 5xx.

**Prefer fewer, deeper queries:**

- Inline related data (variants, collections, basic images).
- Avoid monster queries with all languages/markets in one go — split into base + follow-up batches for translations/market-specific data.

### 2.4 Data modeling: IDs & relationships

**Always store Shopify global IDs** (`gid://shopify/Product/1234567890`), not just numeric IDs, for:

- `Shop`
- `Market`
- `PriceList`
- `Product`
- `ProductVariant`
- `Collection`

**Separate base entities from localized/marketized entities:**

- `Product` — `id`, `shopifyProductId`, non-localized attributes (handle, status, tags).
- `ProductVariant` — `id`, `shopifyVariantId`, base SKU, base attributes.
- `ProductMarket` / `VariantMarket` — link to a specific Market: `available`, `price`, `currency`, `compareAtPrice`.
- `ProductTranslation` — `id`, `productId`, `languageCode/locale` (`en`, `fr-CA`), `title`, `description`, `metaTitle`, `metaDescription`.
- `Collection`, `CollectionProduct`, `CollectionTranslation` similarly.

### 2.5 Markets & currencies sync

1. Fetch all Markets:
   - id, name, primary country/region, supported currencies, PriceList vs auto-conversion.
2. For each Market determine:
   - Primary currency.
   - Whether multi-currency is in use.
   - Association to PriceLists.

For international pricing, sync **PriceLists**:

- PriceList id, currency, rules or fixed prices per variant.

For a channel needing fast/offline response: **materialize** per-market-per-variant price.

### 2.6 Languages & localized content

1. Detect languages configured for the shop.
2. Decide supported locales (not necessarily all).
3. For each product/collection, fetch localized fields per locale; store as `ProductTranslation` / `CollectionTranslation`.

Best practices:

- **Fallback**: record default locale (shop primary). If translation missing, fall back to product primary locale or negotiated fallback.
- **Minimal required fields**: `title`, `bodyHtml`/description, `seo` fields.
- **Incremental translation sync**: subscribe to translation update events or schedule periodic refresh.

### 2.7 Idempotency, consistency checks, safety

**Idempotent operations:**

- Unique constraints on Shopify IDs (e.g. `UNIQUE(shopifyProductId, shopId)`).
- Upserts instead of blind inserts.
- Safe to "rebuild" parts (re-sync all translations for a product) without breaking refs.

**Consistency checks after initial sync:**

- Count of products in Shopify vs your DB (rough match).
- Random product samples — markets match, languages have content.
- High mismatch → log, alert, or re-run affected sections.

**Live changes during initial sync:**

- Enable **webhooks early**, ideally before bulk sync starts.
- Log all webhook events received during sync.
- After bulk fetch finishes, **replay** webhook events that target already-synced objects.

### 2.8 Concurrency & multi-tenant

- **Per-shop sync queues** — one shop's huge catalog doesn't block others.
- Limit **concurrency per shop** to stay under rate limits; more concurrency across shops.
- Track sync state per shop:
  - `status`: `idle`, `initial_sync_in_progress`, `sync_failed`.
  - `lastSuccessfulSyncAt`.
  - `lastCursor` per entity type.

### 2.9 Initial sync checklist

**Setup:**
- [ ] Enable webhooks as early as possible.
- [ ] Prepare DB schema with base entities + Market + Translation tables.

**Markets & currencies:**
- [ ] Fetch all Markets with currencies (Admin GraphQL).
- [ ] Fetch all PriceLists and associate with Markets.
- [ ] Store primary currency and multi-currency info per Market.

**Base catalog:**
- [ ] Paginate through all products (with variants, basic info).
- [ ] Paginate through all collections.
- [ ] Store all `Product`, `ProductVariant`, `Collection`, `CollectionProduct` relationships.

**Market availability:**
- [ ] For each Market, determine which products/variants are available.
- [ ] Populate `ProductMarket` / `VariantMarket` with availability and prices per Market+currency.

**Languages:**
- [ ] Fetch shop's configured languages.
- [ ] For each language: fetch localized product fields → `ProductTranslation`; fetch localized collection fields → `CollectionTranslation`.
- [ ] Record fallbacks (primary locale for shop).

**Finalize & verify:**
- [ ] Replay any webhook events received during initial sync.
- [ ] Run consistency checks (counts, sample comparisons).
- [ ] Mark initial sync complete; switch to incremental-only (webhooks + feeds).

---

## 3. Managing the collection ↔ product relationship

Treat it as a **first-class many-to-many relationship** in your DB.

### 3.1 Conceptual model

In Shopify:

- A **product** can belong to many collections.
- A **collection** can contain many products.

Do NOT store a list of collection IDs on the Product row (or vice versa). Use:

1. `products` table
2. `collections` table
3. **Join table** `collection_products`

### 3.2 Example schema

```text
products
--------
id                     (PK in your DB)
shop_id                (your shop FK)
shopify_product_id     (GraphQL global ID or numeric; unique per shop)
handle
...other base fields...

collections
-----------
id                     (PK in your DB)
shop_id
shopify_collection_id  (GraphQL global ID or numeric; unique per shop)
handle
collection_type        (manual/custom vs smart)
...other base fields...

collection_products
-------------------
id                     (PK in your DB)
shop_id
collection_id          (FK to collections.id)
product_id             (FK to products.id)
position               (optional: sort position within the collection)
added_via              (optional: 'manual' | 'rule' | 'unknown')
```

Key points:

- `collection_products` = authoritative record of which products are in which collections.
- Normalized: a product/collection change touches the smallest necessary set of rows.
- "Which products are in collection X?" → join `collection_products` → `products`.
- "Which collections contain product Y?" → join `collection_products` → `collections`.

### 3.3 Manual (custom) vs smart collections

**Custom/manual:**
- Merchant explicitly adds/removes products.
- Membership stored explicitly.

**Smart:**
- Membership defined by rules (e.g. `tag = "sale"`, `vendor = "Nike"`).
- Membership computed by Shopify.
- You can fetch current membership as if explicit, but it's rule-based.

**Recommended approach — treat both types as "explicit membership":**

- During initial sync, for every collection (manual or smart):
  - Fetch list of currently included products.
  - Populate `collection_products` with one row per product–collection pair.
- For smart collections, optionally also store the **rules** in a `collection_rules` table for visibility, but don't rely on them for membership.

Pros: simple; smart/manual is just a flag. Cons: membership updates must be detected via webhooks or periodic re-sync.

Recomputing smart collections with your own logic is rarely worth it.

### 3.4 Markets & languages relative to the relation

**Collection–product relationships themselves are NOT per-language.** A product is either in a collection or not, regardless of language.

But collections and products have:

- **Languages** → titles, descriptions, SEO text.
- **Markets** → availability per region, prices/currencies.

**Markets & collections:**

- `collection_products` says "Product P is in Collection C".
- `product_markets` (separate table) says "Product P is available in Market M at price X".

You usually don't need `collection_market` unless modeling explicit "collection only visible in Market M". Often derive:

- Collection C shown in Market M if at least one of its products is available in Market M.
- Or apply your own rule (must have N products in that Market).

Possible extension:

```text
product_markets
---------------
id
shop_id
product_id
market_id
available                  (boolean)
default_price              (numeric/decimal)
currency                   (ISO code)
...other per-market fields...

collection_markets  (optional)
-------------------
id
shop_id
collection_id
market_id
explicitly_hidden          (boolean)  # if you support that logic
```

**Languages & collections:**

```text
product_translations
--------------------
id
shop_id
product_id
locale       (e.g. 'en', 'fr', 'de', 'fr-CA')
title
description
seo_title
seo_description
...

collection_translations
-----------------------
id
shop_id
collection_id
locale
title
description
seo_title
seo_description
...
```

When showing a collection to a user:

- Determine market and locale.
- Find products both in the collection (`collection_products`) AND available in that market (`product_markets`).
- Render collection title/description from `collection_translations(locale)`; fall back to default locale if missing.

### 3.5 Initial sync — building the mapping

**Order: products first, then collections.**

1. Full product sync first → base product data.
2. Sync collections:
   - Paginate all collections; store rows (id, handle, type, rules for smart).
   - Identify manual vs smart.
   - For each collection, paginate products via `collection.products(...)` connection.
   - Upsert into `collection_products` (use upsert / insert-ignore to avoid dupes).
   - Record `position` for custom ordering.

This order prevents ordering issues; missing product references signal a bug.

**Smart collection quirks:**

- Initial sync: store current membership like manual collections.
- Ongoing sync:
  - On product/collection change, re-evaluate membership for that single collection (or product).
  - Update `collection_products` (add or remove rows).
- Do NOT re-sync entire smart collection from scratch every time.
  - Listen to `collections/update` for the collection.
  - Listen to `products/update` for a product → fetch which smart collections it now belongs to → reconcile join table.

### 3.6 Ongoing sync — keeping relationships current

**Webhooks / feeds:** on product create/update/delete or collection create/update/delete:

1. Update relevant base entity.
2. Reconcile membership:
   - If webhook carries membership info, update `collection_products` directly.
   - Otherwise fetch membership via Admin GraphQL for the specific product/collection.

**Idempotent updates:** when a product or collection changes:

- Fetch current collection memberships from Shopify.
- Compute diff vs `collection_products`:
  - Add missing `(collection, product)` pairs.
  - Remove pairs that no longer exist.
- Idempotent: running twice yields same final state.

**Consistency checks** (e.g. nightly):

- Re-sync membership for a small random sample of collections.
- Ensure `collection_products` matches Shopify's list.
- On mismatch: schedule re-sync for those collections (or whole catalog for extreme cases).

### 3.7 Reading the catalog from the channel's perspective

For a given collection + market/language:

1. Find collection by handle/id → `collections` table.
2. Get localized fields for user's locale → `collection_translations`.
3. Get all products in the collection:
   ```sql
   SELECT product_id FROM collection_products WHERE collection_id = ?
   ```
4. For each product:
   - Join `product_markets` filtered by active Market → keep only `available = true`.
   - Join `product_translations` filtered by locale → get `title`, `description` with fallback if missing.
5. Apply per-market pricing (from `product_markets` or price lists).

Fast and deterministic because:

- Membership separate (`collection_products`).
- Market availability separate (`product_markets`).
- Language separate (`product_translations`).

### 3.8 Summary — best practices for product–collection relations

1. **Many-to-many join table** — dedicated `collection_products`. No flattened list fields.
2. **Normalize base vs overlays** — `products`, `collections` base; `collection_products` membership; `product_markets` per-market availability/pricing; `product_translations` + `collection_translations` localization.
3. **Treat smart collection membership as explicit** — store current membership from Shopify; optionally store rules for debug; update incrementally on events.
4. **Idempotent upserts on sync** — initial: upsert products, collections, `collection_products`; updates: reconcile current vs local + apply diffs.
5. **Webhooks + periodic validation** — events for live updates; sample re-checks to catch missed updates or bugs.

---

## 4. Sync options comparison (from Shopify "Options to sync product data")

### Contextual product feeds (recommended)

Recommended way to sync product data for sales channel apps.

Advantages:
- Localized product data (pricing, translations) per country and language.
- Full sync to bootstrap catalog + incremental sync for ongoing updates.
- Automatic or manual feed management via channel config extension.
- Webhooks for real-time notifications when products change.

### Storefront API

Useful when your channel needs to look up product data on demand rather than maintaining a synced copy. In context of your sales channel app, returns products published to your channel.

Works for:
- Real-time product lookups at checkout or browsing time.
- Lightweight integrations without local product database.

Requires `unauthenticated_read_product_listings` scope. Watch rate limits.

### Other approaches

For specific use cases:
- Query products directly via Admin GraphQL.
- Bulk operation queries to download large datasets.

Contextual product feeds are the recommended path for sales channel apps because they handle localization, incremental updates, and publishing scope automatically.

---

## 5. Contextual product feeds — flow summary

1. **Describe your channel** — add channel config extension; create `example-us-channel.toml`:

   ```toml
   handle = "example-us"
   label = "example.com"
   icon = "example-channel-icon.svg"
   productFeedManagement = "automatic"

   [capabilities]
   bundles = true
   digitalProducts = true

   [requirements]
   expectsOnlineStoreParity = false
   merchantOfRecord = "channel"

   [[countries]]
   code = "US"
   languages = [ "en" ]
   currency = "USD"
   ```

   Deploy: `shopify app deploy`.

2. **Establish channel connection** — `channelCreate` mutation.

3. **Subscribe to product feed webhooks:**
   - `PRODUCT_FEEDS_FULL_SYNC_FINISH` — bootstrap done signal.
   - `PRODUCT_FEEDS_INCREMENTAL_SYNC` — every product/variant/translation/price/publish change.

4. **Initiate full sync** — `channelFullSync` mutation.

```graphql
mutation {
  channelFullSync(channelId: "gid://shopify/Channel/...") {
    fullSyncTraceInfo { country language operationId }
    userErrors { field message }
  }
}
```

5. **Listen incremental webhook** — keep in sync.

### Typical integration flow

1. Merchant installs app, completes OAuth.
2. App presents external platform authentication.
3. Merchant authenticates with external account.
4. App calls `channelCreate` with specification handle + account details.
5. Shopify creates product feeds.
6. App subscribes to product feed webhooks + triggers full sync.
7. Product data flows to the app.

Repeat 2–6 for each additional channel connection (multi-channel apps).

### Full sync arguments

- `language` — scope to a language; omit for all.
- `country` — scope to a country; omit for all.
- `beforeUpdatedAt` — sync only products not changed since timestamp.
- `updatedAtSince` — sync only products changed since timestamp.

### Incremental sync triggers

- Product fields updated.
- Product variant fields added/updated/deleted.
- Product translations updated.
- Product market price updated.
- Products published / unpublished to the app.
