---
title: About bundle products in product feeds
description: >-
  Learn how bundle products work with sales channel contextual product feed
  webhooks.
source_url:
  html: 'https://shopify.dev/docs/apps/build/sales-channels/about-publishing-bundles'
  md: >-
    https://shopify.dev/docs/apps/build/sales-channels/about-publishing-bundles.md
---

# About bundle products in product feeds

Product bundles allow merchants to group multiple products together and sell them as a single unit. By default, bundle products are excluded from sales channel feeds. When the **Sell bundles** feature is enabled for your sales channel, bundle products are included in contextual product feeds with an `isBundle` field to identify them.

***

## How bundle products work in feeds

After enabling bundle publishing, your contextual product feeds will include bundle products alongside regular products. The `isBundle` field distinguishes bundle from non-bundle products.

### Bundle product behavior

The behavior of bundle products in feeds depends on whether bundle publishing is enabled:

| Product type | Bundle publishing enabled | Bundle publishing disabled |
| - | - | - |
| Regular product | `"isBundle": false` | `"isBundle": null` |
| Bundle product | `"isBundle": true` | Entire webhook payload omitted |

**Note:**

When bundle publishing is disabled, webhooks for bundle products are not sent at all. Your app will not receive any data for bundle products unless the feature is enabled.

***

## Product feed webhook payloads

The [`PRODUCT_FEEDS_INCREMENTAL_SYNC`](https://shopify.dev/docs/api/admin-graphql/latest/enums/webhooksubscriptiontopic#value-productfeedsincrementalsync) and [`PRODUCT_FEEDS_FULL_SYNC`](https://shopify.dev/docs/api/admin-graphql/latest/enums/webhooksubscriptiontopic#value-productfeedsfullsync) webhooks include the `isBundle` field within the product object to identify whether a product is a bundle.

The following examples show the difference between bundle and regular products in webhook payloads:

### Bundle product (when enabled)

```json
{
  "metadata": {
    "action": "UPDATE",
    "type": "INCREMENTAL",
    "resource": "PRODUCT",
    "truncatedFields": [],
    "occurred_at": "2025-01-01T12:00:00-05:00"
  },
  "productFeed": {
    "id": "gid://shopify/ProductFeed/12345",
    "shop_id": "gid://shopify/Shop/12345",
    "country": "US",
    "language": "EN"
  },
  "product": {
    "id": "gid://shopify/Product/123",
    "title": "Team Fan Pack",
    "description": "Everything you need to support your favorite team",
    "onlineStoreUrl": "https://example.com/products/team-fan-pack",
    "createdAt": "2025-01-01T10:00:00-05:00",
    "updatedAt": "2025-01-01T12:00:00-05:00",
    "isPublished": true,
    "publishedAt": "2025-01-01T10:00:00-05:00",
    "productType": "Bundles",
    "vendor": "Acme Sports",
    "handle": "team-fan-pack",
    "isBundle": true,
    "images": {
      "edges": [
        {
          "node": {
            "id": "gid://shopify/ProductImage/1123",
            "url": "https://cdn.shopify.com/s/files/1/0262/9117/5446/products/fan-pack.jpg?v=1675101331",
            "height": 1200,
            "width": 1200
          }
        }
      ]
    },
    "options": [
      {
        "name": "Title",
        "values": ["Default Title"]
      }
    ],
    "seo": {
      "title": "Team Fan Pack - Complete Bundle",
      "description": "Get everything you need to support your team with our complete fan pack bundle"
    },
    "tags": ["bundle", "sports", "team"],
    "variants": {
      "edges": [
        {
          "node": {
            "id": "gid://shopify/ProductVariant/1231",
            "title": "Default Title",
            "price": {
              "amount": "59.99",
              "currencyCode": "USD"
            },
            "compareAtPrice": {
              "amount": "79.99",
              "currencyCode": "USD"
            },
            "sku": "FANPACK-001",
            "barcode": null,
            "quantityAvailable": 50,
            "availableForSale": true,
            "weight": 2.5,
            "weightUnit": "POUNDS",
            "requireShipping": true,
            "inventoryPolicy": "DENY",
            "createdAt": "2025-01-01T10:00:00-05:00",
            "updatedAt": "2025-01-01T12:00:00-05:00",
            "image": null,
            "selectedOptions": [
              {
                "name": "Title",
                "value": "Default Title"
              }
            ]
          }
        }
      ]
    }
  },
  "products": null
}
```

### Regular product

```json
{
  "metadata": {
    "action": "UPDATE",
    "type": "INCREMENTAL",
    "resource": "PRODUCT",
    "truncatedFields": [],
    "occurred_at": "2025-01-01T12:00:00-05:00"
  },
  "productFeed": {
    "id": "gid://shopify/ProductFeed/12345",
    "shop_id": "gid://shopify/Shop/12345",
    "country": "US",
    "language": "EN"
  },
  "product": {
    "id": "gid://shopify/Product/234",
    "title": "Baseball Cap",
    "description": "Premium quality baseball cap",
    "onlineStoreUrl": "https://example.com/products/baseball-cap",
    "createdAt": "2024-12-15T10:00:00-05:00",
    "updatedAt": "2025-01-01T12:00:00-05:00",
    "isPublished": true,
    "publishedAt": "2024-12-15T10:00:00-05:00",
    "productType": "Accessories",
    "vendor": "Acme Sports",
    "handle": "baseball-cap",
    "isBundle": false,
    "images": {
      "edges": [
        {
          "node": {
            "id": "gid://shopify/ProductImage/1234",
            "url": "https://cdn.shopify.com/s/files/1/0262/9117/5446/products/cap.jpg?v=1675101331",
            "height": 800,
            "width": 800
          }
        }
      ]
    },
    "options": [
      {
        "name": "Size",
        "values": ["S/M", "L/XL"]
      }
    ],
    "seo": {
      "title": "Baseball Cap - Team Colors",
      "description": "Show your team spirit with our premium baseball cap"
    },
    "tags": ["accessories", "sports", "headwear"],
    "variants": {
      "edges": [
        {
          "node": {
            "id": "gid://shopify/ProductVariant/2341",
            "title": "S/M",
            "price": {
              "amount": "19.99",
              "currencyCode": "USD"
            },
            "compareAtPrice": null,
            "sku": "CAP-SM-001",
            "barcode": "123456789012",
            "quantityAvailable": 100,
            "availableForSale": true,
            "weight": 0.2,
            "weightUnit": "POUNDS",
            "requireShipping": true,
            "inventoryPolicy": "DENY",
            "createdAt": "2024-12-15T10:00:00-05:00",
            "updatedAt": "2025-01-01T12:00:00-05:00",
            "image": null,
            "selectedOptions": [
              {
                "name": "Size",
                "value": "S/M"
              }
            ]
          }
        }
      ]
    }
  },
  "products": null
}
```

***

## Next steps

* [Enable bundle publishing](https://shopify.dev/docs/apps/build/product-merchandising/bundles/turn-on-publishing) for your sales channel
* Learn more about [contextual product feeds](https://shopify.dev/docs/apps/build/sales-channels/contextual-product-feeds)

***