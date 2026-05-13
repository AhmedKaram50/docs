---
title: Migrating to a multi-channel app
description: Learn how to turn an existing sales channel app into a multi-channel app.
source_url:
  html: >-
    https://shopify.dev/docs/apps/build/sales-channels/migrating-channel-connection-apis
  md: >-
    https://shopify.dev/docs/apps/build/sales-channels/migrating-channel-connection-apis.md
---

# Migrating to a multi-channel app

**Caution:**

This guide is for existing sales channel apps that were created before explicit channel connection management was available. New sales channel apps should use the [`channelCreate`](https://shopify.dev/docs/api/admin-graphql/unstable/mutations/channelCreate) and [`channelUpdate`](https://shopify.dev/docs/api/admin-graphql/unstable/mutations/channelUpdate) APIs from the start.

When you first install a sales channel app, Shopify automatically creates a channel connection record on your behalf. This legacy behavior limits apps to a single channel connection per shop and prevents you from providing explicit connection metadata.

By migrating to explicit channel connections, you can:

* Provide meaningful account identifiers and names for merchant visibility
* Attach channel specifications that describe your channel's capabilities
* Support multiple channel connections per shop in the future

***

## How it works

The migration process updates your app's authorization flow to explicitly manage channel connections using the [`channelCreate`](https://shopify.dev/docs/apps/build/sales-channels/channel-connections#create-a-channel-connection) and [`channelUpdate`](https://shopify.dev/docs/apps/build/sales-channels/channel-connections#update-a-channel-connection) mutations. This approach ensures your code works correctly regardless of whether automatic channel records are being created.

***

## Requirements

* An existing sales channel app
* A [channel specification](https://shopify.dev/docs/apps/build/sales-channels/channel-config-extension#channel-specifications) uploaded via Shopify CLI
* Familiarity with the GraphQL Admin API

***

## Step 1: Add your channel config app extension

Deploy a [channel config extension](https://shopify.dev/docs/apps/build/sales-channels/channel-config-extension) with at least one [channel specification](https://shopify.dev/docs/apps/build/sales-channels/channel-config-extension#channel-specifications) that describes your channel's capabilities and regional coverage.

**Caution:**

Ensure that `create_legacy_channel_on_app_install = true` is set in your `shopify.extension.toml`. Without it, channels won't be created when merchants install your app.

***

## Step 2: Update your authorization flow

Modify your app's authorization completion logic to handle both legacy automatic records and explicit channel connections.

### Query for existing channel records

After a successful authorization, query channels to check if an automatic channel record exists:

## channels query

```graphql
query {
  channels(first: 10) {
    edges {
      node {
        id
        handle
        specificationHandle
      }
    }
  }
}
```

Automatic channel records lack explicit fields such as `specificationHandle`.

### Update or create the channel connection

Convert automatic records to an explicit connection by calling `channelUpdate`:

## channelUpdate mutation

```graphql
mutation ChannelUpdate($id: ID!, $input: ChannelUpdateInput!) {
  channelUpdate(id: $id, input: $input) {
    channel {
      id
      handle
    }
  }
}
```

Provide the following input fields:

| Field | Description |
| - | - |
| `accountId` | The unique identifier for the merchant's account on your platform. |
| `accountName` | A human-readable name for the merchant's account. |
| `specificationHandle` | The handle of one of your uploaded channel specifications. |

If no record exists, call `channelCreate` to establish a new explicit connection:

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
    }
  }
}
```

**Tip:**

This branching logic makes your authorization code resilient to both legacy and modern states. The same code path works before and after you disable automatic channel creation.

***

## Step 3: Disable automatic channel creation

Once your updated authorization flow is deployed and you've verified it works correctly, disable the legacy automatic channel creation behavior.

In your `shopify.extension.toml`, remove `create_legacy_channel_on_app_install = true`.

Then redeploy your app:

```bash
shopify app deploy
```

After deployment, Shopify will no longer create automatic channel records when merchants install your app.

**Note:**

Unless or until you upgrade all legacy merchants, you should keep the authorization code that uses `channelUpdate`.

***

## Step 4: Clean up existing legacy records

After disabling automatic channel creation, some merchants will still have legacy automatic records if they haven't re-authorized with your app. You can clean these up with a batch process using the following options:

1. Convert legacy records. [Query for shops with automatic records and call `channelUpdate`](#query-for-existing-channel-records) to convert them to explicit connections.
2. Remove stale records. If a channel connection is no longer valid, delete the legacy record:

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

***

## Step 5: (Optional) Add multiple connection support

With explicit channel management in place, you can extend your app to support multiple channel connections per shop. Use `channelCreate` to establish additional connections as needed by your platform.

Learn more about [managing multiple channel connections](https://shopify.dev/docs/apps/build/sales-channels/channel-connections).

***