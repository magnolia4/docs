# App-Starter - Read and Write Data

This guide will show you how to set up an app server with our [app template](https://github.com/shopware/AppTemplate).
You will learn how to read and write data to the Shopware Admin API using an example of fetching dynamic translations for products when they are updated.

## Prerequisites

* Basic CLI usage (creating files, directories, running commands)
* Installed [shopware-cli](https://sw-cli.fos.gg/) tools
* Installed [symfony-cli](https://symfony.com/download)
* A running MariaDB or MySQL accessible to your development machine

## Setting up the app template

First we need to create a new Symfony project using Symfony-CLI

```sh
symfony new translator-app
```

The app template contains a basic Symfony application.

Now we need to install the Shopware App Bundle with Composer:

```sh
composer require shopware/app-bundle
```

Modify the `SHOPWARE_APP_NAME` and `SHOPWARE_APP_SECRET` in the env to your app name`./.env` to ensure the app can be installed in a store later.
Also, configure the `DATABASE_URL` to point to your database:

{% code title=".env" %}

```sh
....

###> shopware/app-bundle ###
SHOPWARE_APP_NAME=TestApp
SHOPWARE_APP_SECRET=TestSecret
###< shopware/app-bundle ###
```

{% endcode %}

You can now start the application with `symfony server:start -v`.

For now, your app server is currently only available locally.

{% hint style="info" %}
When you are using a local Shopware environment, you can skip to the [next chapter](#creating-the-manifest)
{% endhint %}

We need to expose your local app server to the internet. The easiest way to achieve that is using a tunneling service like [ngrok](https://ngrok.com/).

The setup is as simple as calling the following command (after installing ngrok)

```sh
ngrok http 8000
```

This will expose your Symfony server on a public URL, so the cloud store can communicate with your app.

## Creating the manifest

The `manifest.xml` is the main interface definition between stores and your app server.
It contains all the required information about your app.
Let's start by filling in all the meta information:

{% code title="release/manifest.xml" %}

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<manifest xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="https://raw.githubusercontent.com/shopware/platform/master/src/Core/Framework/App/Manifest/Schema/manifest-2.0.xsd">
    <meta>
        <name>product-translator</name>
        <label>Product translator</label>
        <description>App to translate product descriptions</description>
        <author>shopware AG</author>
        <copyright>(c) by shopware AG</copyright>
        <version>0.1.0</version>
        <license>MIT</license>
    </meta>
   </manifest>
```

{% endcode %}

{% hint style="warning" %}
Take care to use the same `<name>` as in the `.env` file. Otherwise, stores can't install the app.
{% endhint %}

### Setup hook

Next, we will define the `<setup>` part of the manifest. This part describes how the store will connect itself with the app server.

{% code title="release/manifest.xml" %}

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<manifest xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="https://raw.githubusercontent.com/shopware/platform/master/src/Core/Framework/App/Manifest/Schema/manifest-2.0.xsd">
    <meta>
    <!-- ... -->
    </meta>
    <setup>
        <registrationUrl>http://localhost:8000/app/lifecycle/register</registrationUrl>
        <secret>TestSecret</secret>
    </setup>
</manifest>
```

{% endcode %}

The `<registraionUrl>` is already implemented by the app template and is always `/app/lifecycle/register`, unless you modify `config/routes/shopware_app.yaml`.
The `<secret>` element is only present in development versions of the app. In production, the extension store will provide the secret to authenticate your app buyers.

### Permissions

Permissions are needed as this app will need to read product descriptions and translate them:

{% code title="release/manifest.xml" %}

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<manifest xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="https://raw.githubusercontent.com/shopware/platform/master/src/Core/Framework/App/Manifest/Schema/manifest-2.0.xsd">
    <meta>
    <!-- ... -->
    </meta>
    <setup>
    <!-- ... -->
    </setup>
    <permissions>
        <read>product</read>
        <read>product_translation</read>
        <read>language</read>
        <read>locale</read>
        <update>product</update>
        <update>product_translation</update>
        <create>product_translation</create>
    </permissions>
</manifest>
```

{% endcode %}

### Webhooks

Finally, your app needs to be notified every time a product description is modified.
The app system provides webhooks to subscribe your app server to any changes in the data
in its shops:

{% code title="release/manifest.xml" %}

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<manifest xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="https://raw.githubusercontent.com/shopware/platform/master/src/Core/Framework/App/Manifest/Schema/manifest-2.0.xsd">
    <meta>
    <!-- ... -->
    </meta>
    <setup>
    <!-- ... -->
    </setup>
    <permissions>
    <!-- ... -->
    </permissions>
    <webhooks>
        <webhook name="appActivated" url="http://localhost:8000/app/lifecycle/activate" event="app.activated"/>
        <webhook name="appDeactivated" url="http://localhost:8000/app/lifecycle/deactivate" event="app.deactivated"/>
        <webhook name="appDeleted" url="http://localhost:8000/app/lifecycle/delete" event="app.deleted"/>
        <webhook name="productWritten" url="http://localhost:8000/app/webhook" event="product.written"/>
    </webhooks>
</manifest>
```

{% endcode %}

{% hint style="info" %}
The timeout for the requests against the app server is 5 seconds.
{% endhint %}

These four webhooks are provided by the App Bundle,
so the Bundle does for you the complete lifecycle and handling of Webhooks.

## Handling shop events

To get started, let's write a simple Symfony event listener:

{% code title="src/EventListener/ProductWrittenWebhookListener.php" %}

```php
#[AsEventListener(event: 'webhook.product.written')]
class ProductWrittenWebhookListener
{
    public function __construct(private readonly ClientFactory $clientFactory, private readonly LoggerInterface $logger)
    {
    }

    public function __invoke(WebhookAction $action): void
    {
    }
}
```

{% endcode %}

### Creating a shop client

The Bundle verifies for you the Request and provides you the Webhook parsed together with the Shop it has requested it.
With the Shop, we can create a pre-authenticated PSR-18 Client to communicate with the Shop.
In this example, we will use the SimpleHttpClient which simples the usage of the PSR-18 Client.

{% code title="src/EventListener/ProductWrittenWebhookListener.php" %}

```php
    public function __invoke(WebhookAction $action): void
    {
        $client = $this->clientFactory->createSimpleClient($action->shop);
    }
```

{% endcode %}

Now we can inspect the event payload:

{% code title="src/EventListener/ProductWrittenWebhookListener.php" %}

```php
    public function __invoke(WebhookAction $action): void
    {
        //...

        $updatedFields = $action->payload[0]['updatedFields'];
        $id = $action->payload[0]['primaryKey'];

        if (!in_array('description', $updatedFields)) {
            return;
        }
    }
```

{% endcode %}

### Fetching data from the shop

All `$entity.written` events contain a list of fields that a written event has changed.
The code above uses this information to determine if the description of a product was changed.
If the change did not affect the description, the listener early returns because there is nothing else to do for this event.

Now that it is certain that the description of the product was changed, we fetch the description through the API of the shop:

{% code title="src/EventListener/ProductWrittenWebhookListener.php" %}

```php
    public function __invoke(WebhookAction $action): void
    {
        //...
        $response = $client->post(
            sprintf('%s/api/search/product', $action->shop->getShopUrl()),
            [
                'ids' => [$id],
                'associations' => [
                    'translations' => [
                        'associations' => [
                            'language' => [
                                'associations' => [
                                    'locale' => []
                                ]
                            ],
                        ]
                    ],
                ]
            ]
        );
        
        if (!$response->ok()) {
            $this->logger->error('Could not fetch product', ['response' => $response->json()]);
            return;
        }
    }
```

{% endcode %}

The request contains a criteria that fetches the product for which we received the event `'ids' => [$id]` and all translations and their associated languages `'associations' => 'language'`. Now we can retrieve the English description from the API response:

{% code title="src/EventListener/ProductWrittenWebhookListener.php" %}

```php
    public function __invoke(WebhookAction $action): void
    {
        //...
        $product = $response->json()['data'][0];
        $description = '';
        $name = '';
        foreach ($product['translations'] as $translation) {
            if ($translation['language']['locale']['code'] === 'en-GB') {
                $description = $translation['description'];
                $name = $translation['name'];
            }
        }
    }
```

{% endcode %}

{% hint style="info" %}
A common gotcha with `entity.written` webhooks is that they trigger themselves when you're performing write operations. Updating the description triggers another `entity.written` event. This again calls the webhook, which updates the description, and so on.
{% endhint %}

Because our goal is to write a French translation of the product, the app needs to take care to avoid endless loops.
To determine if the app has already written a translation once, it saves a hash of the original description.
We will get to the generation of the hash later, but we need to check it first:

{% code title="src/EventListener/ProductWrittenWebhookListener.php" %}

```php
    public function __invoke(WebhookAction $action): void
    {
        //...
        $lastHash = $product['customFields']['translator-last-translation-hash'] ?? '';
        if (md5($description) === $lastHash) {
            return;
        }
    }
```

{% endcode %}

### Writing a translated description

Now that the app can be sure the description has not been translated before it can write the new description like so:

{% code title="src/EventListener/ProductWrittenWebhookListener.php" %}

```php
    public function __invoke(WebhookAction $action): void
    {
        //...
        $response = $client->patch(sprintf('%s/api/product/%s', $action->shop->getShopUrl(), $id), [
            'translations' => [
                'en-GB' => [
                    'name' => $name,
                    'description' => $this->translate($description)
                ],
            ],
            'customFields' => [
                'translator-last-translation-hash' => md5($description)
            ]
        ]);

        if (!$response->ok()) {
            $this->logger->error('Could not update product', ['response' => $response->json()]);
        }
    }
```

{% endcode %}

Note that the hash of the original description gets saved as a value in the
custom fields of the product entity. This is possible without any further config since all custom fields are schema-less.

The implementation of the `translate` method is disregarded in this example. You might perform an additional lookup through a translation API service to implement it.

## Install the app

In this last step, we will install the app using the Shopware CLI tools.

{% hint style="info" %}
If this is your first time using the Shopware CLI, you have to [install](https://sw-cli.fos.gg/install/) it first. Next, configure it using the `shopware-cli project config init` command.
{% endhint %}

```sh
shopware-cli project extension upload ProductTranslator/release --activate --increase-version
```

This command will create a zip file from the specified extension directory and upload it to your configured store.
The `--increase-version` parameter increases the version specified in the `manifest.xml` file. This flag is required so Shopware picks up changes made to the `manifest.xml` since the last installation.
Once successfully installed, you will see the app in the extension manager.
And when you save a product, the description will automatically update.

## Where to continue

In this example, you have learned how to receive events and modify data through the app system. You can also:

* Add [new payments](../payment.md) as apps
* Write code that runs during checkout [app scripting](../app-scripts/cart-manipulation.md)
* Add new endpoints to the API [custom endpoints](./add-api-endpoint.md)
