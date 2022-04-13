# Metrics

{% hint style="warning" %}
This feature is still in development and highly experimental. Existing interfaces might change, and we do not guarantee any stability yet.
{% endhint %}

As a merchant you'll most likely want to track what is happening in your shop.
How many orders were placed during the last 30 days?
What was the average cart amount of all those orders?
Is the average amount of line items higher after a marketing campaign?

To answer such questions it can be very helpful to collect all sort of information of an order and push it to an analytics service that helps you visualize and understand this data.
And this is where metrics come into play!

## Collecting Metrics in Shopware
There are several ways of collecting metrics by implementing different interfaces provided by Shopware.
For event based metrics, e.g. when an order is placed, a simple event subscriber can be used.
For more resource-intensive metrics it's better to use a collector, that is run once a day by scheduled task.

### Subscribers
With a metrics subscriber you can subscribe to every instance of `ShopwareEvent` and aggregate additional data that you want to have as part of your metric(s).

In this example we subscribe to the `CustomerRegisterEvent` and add the Sales Channel's identifier to it.

{% code title="custom/plugins/MetricsPlugin/src/Subscriber/CustomerRegisteredMetricsSubscriber" %}
```php
<?php declare(strict_types=1);

namespace MetricsPlugin\Subscriber;

use Shopware\Core\Checkout\Customer\Event\CustomerRegisterEvent;
use Shopware\Core\System\Metrics\AbstractMetricEventSubscriber;
use Shopware\Core\System\Metrics\MetricCollection;
use Shopware\Core\System\Metrics\MetricStruct;

class CustomerRegisteredMetricsSubscriber extends AbstractMetricEventSubscriber
{
    private const METRIC_NAME_CUSTOMER_REGISTERED = 'customer.registered';

    public function getSubscribedEvent(): string
    {
        CustomerRegisterEvent::class;
    }
    
    /**
     * @param CustomerRegisterEvent $event
     */
    public function getMetricsForEvent(ShopwareEvent $event): MetricCollection
    {
        $salesChannelId = $event->getSalesChannelContext()->getSalesChannelId();
        
        $metric = (new MetricStruct(self::METRIC_NAME_CUSTOMER_REGISTERED))
            ->withTags([
                'sales_channel_id' => $salesChannelId
            ]);
            
        return new MetricCollection([$metric]);
    }
}
```
{% endcode %}

### Collectors

### Dispatcher
The `MetricsDispatcher` is the central service used for dispatching metrics collected by subscribers and collectors.
It receives a `MetricStruct` and, enriches it with additional metadata and dispatches this metric to all registered and active metric clients.

### Metadata Providers
When collecting metrics, there are information that can be categorized as additional information that are the same for every metric.
Typically, these are information about the user who triggered an event, about a sales channel where an order was placed or instance data like the Shopware version.

So you don't have to fetch all this metadata every time you collect a metric you can create an `AbstractPartialMetadataProvider`.
Before dispatching a metric to all clients, the `MetricsDispatcher` first asks the `MetadataProvider` to provide all metadata by calling the partial metadata providers tagged with `shopware.metrics.metadata_provider`.

In this example we add the theme's technical name to the metadata, but only if the context's source is a `SalesChannelApiSource`.

{% code title="custom/plugins/MetricsPlugin/src/Core/System/MetadataProvider/CustomMetadataProvider.php" %}
```php
<?php declare(strict_types=1);

namespace MetricsPlugin\Core\System\MetadataProvider;

use Shopware\Core\System\Metrics\AbstractPartialMetadataProvider;
use Shopware\Core\Framework\Api\Context\SalesChannelApiSource;
use Shopware\Core\Framework\Context;
use Shopware\Core\Framework\DataAbstractionLayer\EntityRepositoryInterface;
use Shopware\Core\Framework\DataAbstractionLayer\Search\Criteria;
use Shopware\Core\Framework\DataAbstractionLayer\Search\Filter\EqualsFilter;

class SalesChannelThemeMetadataProvider extends AbstractPartialMetadataProvider
{
    public const KEY_SALES_CHANNEL_ASSIGNED_THEME = 'sales_channel_assigned_theme';

    private EntityRepositoryInterface $themeRepository;

    public function __construct(EntityRepositoryInterface $themeRepository)
    {
        $this->themeRepository = $themeRepository;
    }
    
    public function provide(Context $context): array
    {
        $source = $context->getSource();

        if (!$source instanceof SalesChannelApiSource) {
            return [];
        }
        
        $criteria = new Criteria();
        $criteria->addAssociation('salesChannel');
        $criteria->addFilter(new EqualsFilter('salesChannel.id', $source->getSalesChannelId()));
        
        $theme = $this->themeRepository->search($criteria, $context)->first();
        
        if (!$theme) {
            return [];
        }
        
        return [
            self::KEY_SALES_CHANNEL_ASSIGNED_THEME => $theme->getTechnicalName()
        ];
    }
}
```
{% endcode %}

You can override the metadata of other providers by setting a lower priority on your service definition and putting your value for the same key.
The partial metadata of all providers will be merged inside the `MetadataProvider`.
Please note that the `MetadataProvider` goes through all partial metadata providers only once.
To force it to collect all partial metadata again, you need to call its `::reset()` method.

{% code title="custom/plugins/MetricsPlugin/src/Resources/config/services.xml" %}
```xml
<services>
    <service id="MetricsPlugin\Core\System\MetadataProvider\SalesChannelThemeMetadataProvider">    
        <tag name="shopware.metrics.metadata_provider" priority="20"/>
    </service>
</services>
```
{% endcode %}

### Clients
Metric clients are the connectors to different backends that receive all the different metrics and data you collect.
Common backend services could be DataDog, PostHog, Amazon Timestream, InfluxDB or Prometheus - to name only a few.
They accept a `MetricStruct` and convert all the data contained in it into a format which its backend can handle.

To create your own metric client you simply extend the `AbstractMetricClient`.
All metric clients are tagged with `shopware.metrics.client` and are injected into the `MetricsDispatcher` that calls each client for every metric.

The metric clients are called at a time when the response has already been streamed to the user, so there's no necessity to make sure they're not blocking the request-response-cycle.
Ideally however, the clients are implemented in such a way, that metrics are sent in a batch rather than sending a request for every single metric.
In most cases, this is already done in underlying libraries.

{% code title="custom/plugins/MetricsPlugin/src/Client/InfluxDbClient.php" %}
```php
<?php declare(strict_types=1);

namespace MetricsPlugin\Client;

use InfluxDB\Database;
use InfluxDB\Point;
use Shopware\Core\System\Metrics\AbstractMetricClient;
use Shopware\Core\System\Metrics\MetricStruct;

class InfluxDbClient extends AbstractMetricsClient
{
    private Database $database;

    public function capture(MetricStruct $metric, Context $context): void
    {
        $point = new Point(
            $metric->getName(),
            $metric->getValue(),
            $metric->getTags(),
            $metric->getMetadata()
        );
        
        $this->database->writePoints([$point]);
    }
}
```
{% endcode %}

When tagging the service, you add a `client` attribute to it, which is used as an index when injecting the client into the `MetricsDispatcher`.
Clients that are not configured inside the `shopware.yaml` configuration file, will be removed from the `MetricsDispatcher` and won't receive any metrics.

{% code title="custom/plugins/MetricsPlugin/src/Resources/config/services.xml" %}
```xml
<services>
    <service id="MetricsPlugin\Client\InfluxDbClient">    
        <tag name="shopware.metrics.client" client="InfluxDb"/>
    </service>
</services>
```
{% endcode %}

## Configuration
{% hint style="info" %}
The shop's administrator has to opt-in to allow tracking of metrics. If the opt-in is not active, no metrics will be collected.
{% endhint %}

You can enable metric clients by specifying them under `shopware.metrics.clients` in the `shopware.yaml` file.
Only metric clients tagged with `shopware.metrics.client` **and** activated there will be injected into the `MetricsDispatcher`.

In this example we activate an additional `InfluxDb` client that we will be implemented later.
Please notice that the client's name in `shopware.yaml` matches the tag's `client` attribute when defining the PHP service.

{% code title="custom/pluings/MetricsPlugin/src/Resources/config/packages/shopware.yaml" %}
```yaml
shopware:
    metrics:
        clients: ["PostHog", "InfluxDb"]
```
{% endcode %}
