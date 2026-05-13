---
title: Channel config extension
description: >-
  Learn how to use the channel config extension to configure your app as a sales
  channel.
source_url:
  html: 'https://shopify.dev/docs/apps/build/sales-channels/channel-config-extension'
  md: >-
    https://shopify.dev/docs/apps/build/sales-channels/channel-config-extension.md
---

# Channel config extension

**Tip:**

Ensure you have the [latest version of Shopify CLI](https://shopify.dev/docs/api/shopify-cli#upgrade) installed to access all channel config extension features.

***

## Overview

Channel config extension declares your app as a sales channel and configures its properties. Adding this extension to your app enables the following:

* Your app appears as a **sales channel** to merchants in the Shopify admin.
* Merchants can publish products to your channel.
* You control which products are published via channel specifications.
* You can receive [contextual product feeds](https://shopify.dev/docs/apps/build/sales-channels/contextual-product-feeds) for your channel's supported countries and languages.

You can scaffold an example channel config extension using Shopify CLI:

## Terminal

```bash
shopify app generate extension --template channel_config --name channel-config
```

**Note:**

If you've previously declared your app as a sales channel app using the deprecated Partner Dashboard, your app continues to be treated as a sales channel even if this extension isn't present. To migrate an existing app, follow [Migrating to a multi-channel app](https://shopify.dev/docs/apps/build/sales-channels/migrating-channel-connection-apis).

### Directory structure

## Channel config extension directory tree

```text
└── extensions
    └── channel-config
        ├── shopify.extension.toml
        └── specifications/
            ├── example-us.toml
            ├── example-us-icon.svg
            ├── example-de.toml
            └── example-de-icon.svg
```

***

## `shopify.extension.toml`

Like all app extensions, the primary configuration file for the channel config extension is `shopify.extension.toml`.

## Example shopify.extension.toml

```toml
[[extensions]]
uid = "..."
type = "channel_config"
handle = "channel-config"
```

***

## Channel specifications

Channel specifications describe a channel's capabilities, regional coverage, product schema, internationalization, and UI extensibility.

A sales channel app can define one or more specifications. Marketplaces like Amazon and eBay often create one specification per country because listing management is scoped to that region. Channels like Google and Meta might define one specification that covers multiple countries so a single connection can emit listings for each covered region.

Each specification has a unique handle. That handle becomes the `specificationHandle` you pass to [`channelCreate`](https://shopify.dev/docs/api/admin-graphql/unstable/mutations/channelCreate) to bind a merchant's external account to the correct channel configuration.

Specifications are declared as TOML or JSON files in the `specifications/` folder of the extension. Deploy specifications to Shopify using Shopify CLI:

## Terminal

```bash
shopify app deploy
```

### Specification format

## Example: Amazon US channel specification

```toml
handle = "amazon-us"
label = "Amazon - US"
icon = "amazon-us-icon.svg"


[capabilities]
bundles = true


[requirements]
expectsOnlineStoreParity = false
merchantOfRecord = "channel"


[[countries]]
code = "US"
languages = ["en"]
currency = "USD"
```

### Specification properties

#### Top-level properties

| Field | Type | Required | Description |
| - | - | - | - |
| `handle` | String | Yes | Unique human-friendly identifier for the channel specification. Referenced in code when calling `channelCreate`. |
| `label` | String | Yes | Display text for the channel as it appears in the Shopify admin, for example in Markets or the view-as context. Should match the application name, decorated with regional or channel-specific attributes: `"Amazon - US"`, `"Google Local Inventory"`, `"TikTok Shops"`. |
| `name` | String | Yes | Name of the extension. |
| `description` | String | No | Brief description of the channel specification. |
| `icon` | String | No | File name for an SVG icon to represent the channel within Markets and other admin surfaces. The viewbox of the SVG icon MUST be 20 x 20 with origin in 0. |
| `productFeedManagement` | Enum | No | `"automatic"` or `"manual"`. When automatic (default), product feeds are created and destroyed based on the channel's market configuration. When manual, the app controls feed lifecycle using the `productFeedCreate` API. |

#### Capabilities

Describes what types of product features and selling plans the channel supports. All capability fields default to `false` if omitted.

| Field | Type | Required | Description |
| - | - | - | - |
| `capabilities.bundles` | Boolean | No | Whether the channel can sell bundled products. Defaults to `false`. |
| `capabilities.digitalProducts` | Boolean | No | Whether the channel can sell products that don't require physical shipping. Defaults to `false`. |
| `capabilities.subscriptions` | Boolean | No | Whether the channel supports recurring subscription selling plans. Defaults to `false`. |
| `capabilities.unlistedProducts` | Boolean | No | Whether the channel can sell products that aren't publicly listed in search results. Defaults to `false`. |
| `capabilities.combinedListings` | Boolean | No | Whether the channel can sell [combined listings](https://shopify.dev/docs/apps/build/product-merchandising/combined-listings). Defaults to `false`. |
| `capabilities.scheduledPublishing` | Boolean | No | Whether the channel can support merchants [scheduling the publishing of their products](https://shopify.dev/docs/apps/build/sales-channels/scheduled-product-publishing). Defaults to `false`. |

#### Requirements

| Field | Type | Required | Description |
| - | - | - | - |
| `requirements.expectsOnlineStoreParity` | Boolean | No | Whether the channel expects product listings to match the merchant's online store pricing and availability. Channels like Google and Meta that drive traffic back to the merchant's store set this to `true`. Marketplace channels with their own checkout like Amazon and Walmart set this to `false`. Defaults to `false`. |
| `requirements.merchantOfRecord` | Enum | No | Identifies who processes the transaction. Set to `"channel"` when the external platform handles checkout and payment (marketplaces like Amazon and Walmart). Set to `"merchant"` when the merchant's own checkout handles the transaction (traffic-driving channels like Google and Meta). Defaults to `merchant` |

#### Countries

The `countries` array defines the regional coverage of the channel. Each entry describes a country the channel operates in and the languages and currency it supports for that country.

| Field | Type | Required | Description |
| - | - | - | - |
| `countries[].code` | String | Yes | ISO 3166-1 alpha-2 country code. |
| `countries[].languages` | Array | Yes | ISO 639-1 language codes. For each supported language that matches available translations on the store, the product feed payload incorporates available translations. |
| `countries[].currency` | String | Yes | ISO 4217 currency code. If the channel currency matches the market currency, no conversion is applied. If they differ, a currency conversion step is applied in the product feed. |

Multiple countries in a single specification mean each product generates multiple feed entries — one per country — assuming that the store has market coverage for those countries.

### Multiple specifications per app

A sales channel app can deploy multiple specifications to support different regional configurations or channel variants. For example, an Amazon app might deploy:

## Multiple specifications

```text
specifications/
├── amazon-us.toml    # Amazon.com (US, English, USD)
├── amazon-ca.toml    # Amazon.ca (CA, English + French, CAD)
├── amazon-de.toml    # Amazon.de (DE, German, EUR)
└── amazon-uk.toml    # Amazon.co.uk (GB, English, GBP)
```

Each specification has its own handle, and merchants can establish separate [channel connections](https://shopify.dev/docs/apps/build/sales-channels/channel-connections) for each.

***

## Existing channel apps

If you have an existing sales channel app that was created using the Partner Dashboard before the channel config extension was available, you'll need to set `create_legacy_channel_on_app_install = true` in your `shopify.extension.toml` to preserve backward compatibility.

## Legacy channel configuration

```toml
[[extensions]]
uid = "..."
type = "channel_config"
handle = "channel-config"
create_legacy_channel_on_app_install = true
```

For a complete migration guide, see [Migrating to a multi-channel app](https://shopify.dev/docs/apps/build/sales-channels/migrating-channel-connection-apis).

***

## Next steps

* **[Managing channel connections](https://shopify.dev/docs/apps/build/sales-channels/channel-connections)**: Create and manage channel connections using your specifications.
* **[Contextual product feeds](https://shopify.dev/docs/apps/build/sales-channels/contextual-product-feeds)**: Receive product data scoped to your channel's configuration.
* **[Migrating to a multi-channel app](https://shopify.dev/docs/apps/build/sales-channels/migrating-channel-connection-apis)**: Upgrade an existing single-channel app.

***