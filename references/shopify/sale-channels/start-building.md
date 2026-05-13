---
title: Start building a sales channel app
description: Learn how to turn an app into a sales channel app.
source_url:
  html: 'https://shopify.dev/docs/apps/build/sales-channels/start-building'
  md: 'https://shopify.dev/docs/apps/build/sales-channels/start-building.md'
---

# Start building a sales channel app

To build a sales channel app, you need to scaffold an app and turn it into a sales channel app.

***

## What you'll learn

After finishing this tutorial, you'll know how to do the following tasks:

* Turn an app into a sales channel app
* Add the channel config extension to your app
* Define channel specifications for your app

***

## Requirements

* You've created a [Partner account](https://www.shopify.com/partners) and a [dev store](https://shopify.dev/docs/apps/build/dev-dashboard/development-stores#create-a-development-store-to-test-your-app).

***

## Step 1: Create a public app

Create a public app by [scaffolding with Shopify CLI](https://shopify.dev/docs/apps/build/scaffold-app) or [creating an app in the Dev Dashboard](https://shopify.dev/docs/apps/build/dev-dashboard/create-apps-using-dev-dashboard).

***

## Step 2: Add the channel config extension

From your app's root directory, declare your app as a sales channel by adding the channel config extension using Shopify CLI:

## Terminal

```bash
shopify app generate extension --template channel_config --name channel-config
```

This creates a `channel-config` extension in your app's `extensions/` directory. The extension includes a `shopify.extension.toml` configuration file and a `specifications/` folder where you define your [channel specifications](https://shopify.dev/docs/apps/build/sales-channels/channel-config-extension#channel-specifications).

Deploy the extension to register your app as a sales channel:

## Terminal

```bash
shopify app deploy
```

After deploying, you can install the app on your dev store through the [CLI](https://shopify.dev/docs/api/shopify-cli/commands/app/app-dev) or the [Dev Dashboard](https://shopify.dev/docs/apps/launch/app-store-review/pass-app-review#install-your-app-on-a-development-store).

For more details on configuring the extension and writing channel specifications, refer to [Channel config extension](https://shopify.dev/docs/apps/build/sales-channels/channel-config-extension).

***

## Step 3: Define channel specifications

After generating the extension, add one or more channel specifications to the `specifications/` directory to describe your channel's supported countries, languages, currencies, and capabilities.

For details on writing specifications, refer to [Channel config extension](https://shopify.dev/docs/apps/build/sales-channels/channel-config-extension#channel-specifications).

**Note:**

If you're migrating an existing sales channel app that was previously declared through the Partner Dashboard, follow [Migrating to a multi-channel app](https://shopify.dev/docs/apps/build/sales-channels/migrating-channel-connection-apis).

***