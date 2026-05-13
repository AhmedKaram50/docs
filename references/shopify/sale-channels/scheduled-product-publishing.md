---
title: Scheduled publishing
description: >-
  Learn how to integrate your sales channel app with scheduled product
  publishing.
source_url:
  html: >-
    https://shopify.dev/docs/apps/build/sales-channels/scheduled-product-publishing
  md: >-
    https://shopify.dev/docs/apps/build/sales-channels/scheduled-product-publishing.md
---

# Scheduled publishing

Merchants can schedule products to be published to a sales channel at a specific datetime. Common use cases include product drops and timed sales, where merchants want to reserve sufficient inventory to meet spikes in customer demand.

***

## Validation workflow

If your channel app requires products to meet requirements before they can display to customers, then you need to validate products before their scheduled publication datetime. This gives merchants time to update products and resubmit them for validation. Otherwise, the channel validates the products at the scheduled datetime, and any updates required might delay their publication.

The following is the recommended workflow for channels that include validation:

1. A merchant schedules a product to be published on the channel app at a specified datetime.
2. Shopify sends a `scheduled_product_listings/add` event to the channel app.
3. The channel app validates the product against requirements for displaying it on the channel.
4. If the product fails validation, then the channel app sends feedback to the merchant using the `ResourceFeedback` object.
5. The merchant updates the product to meet the requirements.
6. Shopify sends a `scheduled_product_listings/update` event to the channel app.
7. The channel app validates the product against the requirements.
8. At the scheduled datetime, Shopify sends a `product_listing/add` event.
9. The channel app reads the product data and displays the product.

![An image of the scheduled publishing workflow](https://shopify.dev/assets/assets/images/apps/scheduled-publishing-validation-DPu_9Gvu.png)

***

## Requirements

* You've completed the [getting started tutorials](https://shopify.dev/docs/apps/build/sales-channels/start-building).
* Your app requires the `read_product_listings` [access scope](https://shopify.dev/docs/api/usage/access-scopes).

***

## Step 1: Identify products that are scheduled for publication

Request `resourcePublicationOnCurrentPublication` on the [GraphQL Admin API's Product](https://shopify.dev/docs/api/admin-graphql/latest/objects/product) object. A `publishDate` in the future indicates that the product is scheduled to display on the channel at the specified datetime.

## POST /api/{api\_version}/graphql.json

##### GraphQL query

```graphql
{
  products(first: 10) {
    edges {
      node {
        title
        resourcePublicationOnCurrentPublication {
          publication {
            name
            id
          }
          publishDate
          isPublished
        }
      }
    }
  }
}
```

##### JSON response

```json
{
   "data":{
      "products":{
         "edges":[
            {
               "node":{
                  "title":"Baseball",
                  "resourcePublicationOnCurrentPublication":{
                     "publication":{
                        "name":"Sales Channel",
                        "id":"gid://shopify/Publication/2"
                     },
                     "publishDate":"2021-09-23T17:30:00Z",
                     "isPublished":false
                  }
               }
            },
            {
               "node":{
                  "title":"Soccer ball",
                  "resourcePublicationOnCurrentPublication":{
                     "publication":{
                        "name":"Sales Channel",
                        "id":"gid://shopify/Publication/2"
                     },
                     "publishDate":"2021-09-10T17:30:00Z",
                     "isPublished":true
                  }
               }
            }
         ]
      }
   }
}
```

***

## Step 2: Subscribe to webhooks

**Note:**

This step is only required if the channel [validates products against requirements before displaying them](#validation-workflow).

Shopify's [`SCHEDULED_PRODUCT_LISTINGS`](https://shopify.dev/docs/api/admin-graphql/latest/enums/webhooksubscriptiontopic) webhooks notify channels when products are scheduled to be published.

The following is an example payload:

## Example webhook payload

```json
{
  "product_listing": {
    "product_id": 788032119674292922,
    "created_at": null,
    "updated_at": "2021-10-01T17:12:26-04:00",
    "body_html": null,
    "handle": "example-t-shirt",
    "product_type": "Shirts",
    "title": "Example T-Shirt",
    "vendor": "Acme",
    "available": false,
    "tags": "example, mens, t-shirt",
    "publish_at": "2021-10-01T17:12:26-04:00",
    "variants": [
      {
        "id": 642667041472713922,
        "title": "",
        "option_values": [
          {
            "option_id": 527050010214937811,
            "name": "Title",
            "value": "Small"
          }
        ],
        "price": "19.99",
        "formatted_price": "$19.99",
        "compare_at_price": "24.99",
        "grams": 200,
        "requires_shipping": true,
        "sku": "example-shirt-s",
        "barcode": null,
        "taxable": true,
        "position": 0,
        "available": false,
        "inventory_policy": "deny",
        "inventory_quantity": 0,
        "inventory_management": "shopify",
        "fulfillment_service": "manual",
        "weight": 200.0,
        "weight_unit": "g",
        "image_id": null,
        "created_at": null,
        "updated_at": null
      }
    ],
    "images": [
      {
        "id": 539438707724640965,
        "created_at": null,
        "position": 0,
        "updated_at": null,
        "product_id": 788032119674292922,
        "src": "//cdn.shopify.com/shopifycloud/shopify/assets/shopify_shirt-39bb555874ecaeed0a1170417d58bbcf792f7ceb56acfe758384f788710ba635.png",
        "variant_ids": [],
        "width": 323,
        "height": 434
      }
    ],
    "options": [
      {
        "id": 527050010214937811,
        "name": "Title",
        "product_id": 788032119674292922,
        "position": 1,
        "values": [
          "Small",
          "Medium",
          "Large"
        ]
      }
    ]
  }
}
```

***