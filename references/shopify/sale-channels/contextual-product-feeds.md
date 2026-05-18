---
title: Managing channel connections
description: >-
  Learn how to create and manage channel connections using the channel
  connection APIs to support multiple channels per app.
source_url:
  html: 'https://shopify.dev/docs/apps/build/sales-channels/channel-connections'
  md: 'https://shopify.dev/docs/apps/build/sales-channels/channel-connections.md'
---

# Managing channel connections

Channel connections represent authenticated links between a Shopify store and external platforms. Each connection ties a [channel specification](https://shopify.dev/docs/apps/build/sales-channels/channel-config-extension#channel-specifications) to a specific seller account on the external platform. A single app can manage multiple channel connections — for example, an Amazon app can connect `Amazon US`, `Amazon CA`, and `Amazon DE` as separate connections, each with its own specification, account, and product feeds.

***

## What you'll learn

In this guide, you'll learn how to do the following tasks:

* Create a channel connection using the [`channelCreate`](https://shopify.dev/docs/api/admin-graphql/unstable/mutations/channelCreate) mutation
* Update and delete channel connections using [`channelUpdate`](https://shopify.dev/docs/api/admin-graphql/unstable/mutations/channelUpdate) and [`channelDelete`](https://shopify.dev/docs/api/admin-graphql/unstable/mutations/channelDelete)
* Query channel connections for your app
* Trigger a full product sync using [`channelFullSync`](https://shopify.dev/docs/api/admin-graphql/unstable/mutations/channelFullSync)

***

## Requirements

* Your app is registered as a sales channel.
* You have deployed a [channel config extension](https://shopify.dev/docs/apps/build/sales-channels/channel-config-extension) with at least one [channel specification](https://shopify.dev/docs/apps/build/sales-channels/channel-config-extension#channel-specifications).
* Your app has the `read_product_listings` [access scope](https://shopify.dev/docs/api/usage/access-scopes).

***

## How channel connections work

When you call [`channelCreate`](https://shopify.dev/docs/api/admin-graphql/unstable/mutations/channelCreate) with a specification handle and account details, Shopify:

1. Reads the specification to determine regional coverage (countries, languages, currencies)
2. Finds the store's markets that intersect with the specification's covered countries
3. Creates product feeds for each covered country, which immediately begin emitting events to your app's webhook subscriptions

When a channel connection is deleted via [`channelDelete`](https://shopify.dev/docs/api/admin-graphql/unstable/mutations/channelDelete), all associated product feeds are removed.

***

## Create a channel connection

After a merchant completes authentication with their external platform account, call [`channelCreate`](https://shopify.dev/docs/api/admin-graphql/unstable/mutations/channelCreate) to establish the connection:

## POST https://\{shop\}.myshopify.com/admin/api/\{api\_version\}/graphql.json

## channelCreate mutation

```graphql
mutation ChannelCreate($input: ChannelCreateInput!) {
  channelCreate(input: $input) {
    channel {
      id
      handle
    }
    userErrors {
      field
      message
      code
    }
  }
}
```

## Input variables

```json
{
  "input": {
    "handle": "amazon-us-A1B2C3D4E5F6G7",
    "specificationHandle": "amazon-us",
    "accountId": "A1B2C3D4E5F6G7",
    "accountName": "My Amazon Store"
  }
}
```

## JSON response

```json
{
  "data": {
    "channelCreate": {
      "channel": {
        "id": "gid://shopify/Channel/1068177798",
        "handle": "amazon-us-A1B2C3D4E5F6G7"
      },
      "userErrors": []
    }
  }
}
```

### Input fields

| Field | Type | Required | Description |
| - | - | - | - |
| `handle` | String | No | A unique identifier for the channel connection, for example `"amazon-us-A1B2C3D4E5F6G7"`. If not provided, a handle is auto-generated from the specification handle and account ID. |
| `specificationHandle` | String | Yes | The handle of a channel specification deployed via the [channel config extension](https://shopify.dev/docs/apps/build/sales-channels/channel-config-extension). |
| `accountId` | String | Yes | A unique identifier for the merchant's account on the external platform. |
| `accountName` | String | Yes | A human-readable name for the merchant's account, displayed in the Shopify admin when referencing the connection. |

### Online store parity requirements

If the channel specification declares `requirements.expectsOnlineStoreParity = true` (traffic-driving channels like Google or Meta), `channelCreate` requires the store to have at least one region market with a valid web presence covering the specification's countries. If no matching markets exist, the mutation returns an error.

***

## Update a channel connection

Use [`channelUpdate`](https://shopify.dev/docs/api/admin-graphql/unstable/mutations/channelUpdate) to modify account information or reassign a channel specification:

## POST https://\{shop\}.myshopify.com/admin/api/\{api\_version\}/graphql.json

## channelUpdate mutation

```graphql
mutation ChannelUpdate($id: ID!, $input: ChannelUpdateInput!) {
  channelUpdate(id: $id, input: $input) {
    channel {
      id
      handle
    }
    userErrors {
      field
      message
    }
  }
}
```

## Input variables

```json
{
  "id": "gid://shopify/Channel/10079785100",
  "input": {
    "accountName": "Updated Store Name"
  }
}
```

## JSON response

```json
{
  "data": {
    "channelUpdate": {
      "channel": {
        "id": "gid://shopify/Channel/10079785100",
        "handle": "amazon-us-A1B2C3D4E5F6G7"
      },
      "userErrors": []
    }
  }
}
```

**Tip:**

`channelUpdate` is also used to migrate legacy automatic channel records to explicit connections. See [Migrating to a multi-channel app](https://shopify.dev/docs/apps/build/sales-channels/migrating-channel-connection-apis) for details.

***

## Delete a channel connection

When a merchant disconnects their external account or a channel connection is no longer valid, remove it with [`channelDelete`](https://shopify.dev/docs/api/admin-graphql/unstable/mutations/channelDelete). This removes all associated product feeds.

## POST https://\{shop\}.myshopify.com/admin/api/\{api\_version\}/graphql.json

## channelDelete mutation

```graphql
mutation ChannelDelete($id: ID!) {
  channelDelete(id: $id) {
    deletedId
    userErrors {
      field
      message
    }
  }
}
```

## Input variables

```json
{
  "id": "gid://shopify/Channel/10079785100"
}
```

***

## Query channel connections

### Query a single channel by ID

## channel query

```graphql
query Channel($id: ID!) {
  channel(id: $id) {
    id
    handle
    accountId
    accountName
    specificationHandle
  }
}
```

For the full list of fields available on `Channel`, refer to the [Channel object](https://shopify.dev/docs/api/admin-graphql/unstable/objects/channel) in the GraphQL Admin API reference.

### Query a channel by handle

## channelByHandle query

```graphql
query ChannelByHandle($handle: String!) {
  channelByHandle(handle: $handle) {
    id
    handle
    accountId
    accountName
  }
}
```

### List all channels for the app

## channels query

```graphql
query {
  channels(first: 10) {
    nodes {
      id
      handle
      accountId
      accountName
      specificationHandle
    }
  }
}
```

**Note:**

A sales channel app cannot query channels belonging to other apps.

***

## Trigger a full sync for a channel

After establishing a channel connection and subscribing to [product feed webhooks](https://shopify.dev/docs/apps/build/sales-channels/contextual-product-feeds#step-3-subscribe-to-product-feed-webhooks), trigger a full sync to receive the merchant's published product catalog. A full sync is triggered by default when your feed is first created, but if you need a full reconciliation you can trigger a full sync manually using the following mutation:

## POST https://\{shop\}.myshopify.com/admin/api/\{api\_version\}/graphql.json

## channelFullSync mutation

```graphql
mutation ChannelFullSync($channelId: ID!, $language: LanguageCode, $country: CountryCode) {
  channelFullSync(channelId: $channelId, language: $language, country: $country) {
    fullSyncTraceInfo {
      country
      language
      operationId
    }
    userErrors {
      message
    }
  }
}
```

## Input variables

```json
{
  "channelId": "gid://shopify/Channel/1",
  "language": "en",
  "country": "CA"
}
```

## JSON response

```json
{
  "data": {
    "channelFullSync": {
      "fullSyncTraceInfo": [
        {
          "country": "CA",
          "language": "en",
          "operationId": "gid://shopify/ProductFullSync/31987319841212"
        }
      ],
      "userErrors": []
    }
  }
}
```

The `country` and `language` parameters are optional. If omitted, all feeds for the channel are synced.

If your channel manages feeds manually, create the feeds first and trigger [`productFullSync`](https://shopify.dev/docs/api/admin-graphql/latest/mutations/productFullSync) for each feed instead. See [Contextual product feeds](https://shopify.dev/docs/apps/build/sales-channels/contextual-product-feeds#step-4-optional-create-feeds-manually).

***

## Typical integration flow

A typical sales channel app integrates channel connections as part of its post-authentication flow:

1. Merchant installs the app and completes OAuth
2. App presents the external platform authentication (for example, Amazon Seller Central login)
3. Merchant authenticates with their external account
4. App calls `channelCreate` with the appropriate specification handle and account details
5. Shopify creates product feeds
6. App subscribes to product feed webhooks and triggers a full sync
7. Product data begins flowing to the app

For apps supporting multiple channel connections, repeat steps 2–6 for each additional connection.

***

## Next steps

* **[Channel config extension](https://shopify.dev/docs/apps/build/sales-channels/channel-config-extension)**: Set up the extension and describe your channel's capabilities.
* **[Contextual product feeds](https://shopify.dev/docs/apps/build/sales-channels/contextual-product-feeds)**: Subscribe to product feed events.
* **[Migrating to a multi-channel app](https://shopify.dev/docs/apps/build/sales-channels/migrating-channel-connection-apis)**: Upgrade an existing single-channel app.

***