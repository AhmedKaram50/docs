---
title: Options to sync product data
description: >-
  Learn about the recommended ways to access and sync product data for your
  sales channel app.
source_url:
  html: 'https://shopify.dev/docs/apps/build/sales-channels/product-sync'
  md: 'https://shopify.dev/docs/apps/build/sales-channels/product-sync.md'
---

# Options to sync product data

Sales channel apps need access to merchant product data to syndicate catalogs to external platforms. Shopify provides several methods depending on your use case.

***

## Contextual product feeds (recommended)

[Contextual product feeds](https://shopify.dev/docs/apps/build/sales-channels/contextual-product-feeds) are the recommended way to sync product data for sales channel apps. Product feeds provide localized product data for each country and language pair that a merchant supports, and they keep your channel in sync through incremental updates.

Product feeds offer the following advantages:

* Localized product data (pricing, translations) per country and language
* Full sync to bootstrap your catalog and incremental sync for ongoing updates
* Automatic or manual feed management via the [channel config extension](https://shopify.dev/docs/apps/build/sales-channels/channel-config-extension)
* Webhooks for real-time notifications when products change

To get started, see the [contextual product feeds](https://shopify.dev/docs/apps/build/sales-channels/contextual-product-feeds) tutorial.

***

## Storefront API

The [Storefront API](https://shopify.dev/docs/api/storefront) is useful when your channel needs to look up product data on demand rather than maintaining a synced copy. When called in context of your sales channel app, the Storefront API returns products that are published to your channel.

This approach works well for:

* Real-time product lookups at checkout or browsing time
* Lightweight integrations that don't require a local product database

**Note:**

Make sure you have the correct [access scopes](https://shopify.dev/docs/api/usage/access-scopes). The Storefront API requires the `unauthenticated_read_product_listings` scope. Also consider [rate limits](https://shopify.dev/docs/api/usage/limits#rate-limits) when pulling data on demand.

***

## Other approaches

For specific use cases, you can also query products directly via the [GraphQL Admin API](https://shopify.dev/docs/api/admin-graphql) or use [bulk operation queries](https://shopify.dev/docs/api/usage/bulk-operations/queries) to download large datasets. However, contextual product feeds are the recommended path for sales channel apps because they handle localization, incremental updates, and publishing scope automatically.

***