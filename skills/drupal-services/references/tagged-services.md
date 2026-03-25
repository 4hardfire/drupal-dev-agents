# Tagged Services in Drupal

## How Service Tags Work

Service tags are metadata labels in `*.services.yml` that tell the container a service belongs to a specific category. Drupal (and Symfony) uses compiler passes to collect all services with a given tag and pass them to a "collector" service.

```yaml
services:
  mymodule.my_subscriber:
    class: Drupal\mymodule\EventSubscriber\MySubscriber
    tags:
      - { name: event_subscriber }
```

The tag `event_subscriber` causes the event dispatcher to discover and register this subscriber automatically.

---

## Common Tags and Their Interfaces

### event_subscriber

Reacts to kernel, entity, config, and custom events.

```yaml
services:
  mymodule.route_subscriber:
    class: Drupal\mymodule\EventSubscriber\RouteSubscriber
    tags:
      - { name: event_subscriber }
```

```php
<?php

namespace Drupal\mymodule\EventSubscriber;

use Drupal\Core\Routing\RouteBuildEvent;
use Drupal\Core\Routing\RoutingEvents;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

class RouteSubscriber implements EventSubscriberInterface {

  public static function getSubscribedEvents(): array {
    return [
      RoutingEvents::ALTER => ['onRouteAlter', -100],
    ];
  }

  public function onRouteAlter(RouteBuildEvent $event): void {
    $collection = $event->getRouteCollection();
    if ($route = $collection->get('user.login')) {
      $route->setDefault('_title', 'Sign In');
    }
  }

}
```

### access_check

Custom route access logic. The `applies_to` attribute links the checker to routes that declare a specific requirement key.

```yaml
services:
  mymodule.access_checker.premium:
    class: Drupal\mymodule\Access\PremiumAccessCheck
    tags:
      - { name: access_check, applies_to: _premium_access }
```

```php
<?php

namespace Drupal\mymodule\Access;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Access\AccessResultInterface;
use Drupal\Core\Routing\Access\AccessInterface;
use Drupal\Core\Session\AccountInterface;

class PremiumAccessCheck implements AccessInterface {

  public function access(AccountInterface $account): AccessResultInterface {
    return AccessResult::allowedIf(
      $account->hasPermission('access premium content')
    )->cachePerPermissions();
  }

}
```

Route definition that uses it:

```yaml
mymodule.premium_page:
  path: '/premium'
  defaults:
    _controller: '\Drupal\mymodule\Controller\PremiumController::view'
  requirements:
    _premium_access: 'TRUE'
```

### path_processor_inbound / path_processor_outbound

Alter paths before routing (inbound) or when generating URLs (outbound).

```yaml
services:
  mymodule.path_processor:
    class: Drupal\mymodule\PathProcessor\LegacyRedirectProcessor
    tags:
      - { name: path_processor_inbound, priority: 200 }
```

```php
<?php

namespace Drupal\mymodule\PathProcessor;

use Drupal\Core\PathProcessor\InboundPathProcessorInterface;
use Symfony\Component\HttpFoundation\Request;

class LegacyRedirectProcessor implements InboundPathProcessorInterface {

  public function processInbound(string $path, Request $request): string {
    if (str_starts_with($path, '/old-blog/')) {
      return str_replace('/old-blog/', '/blog/', $path);
    }
    return $path;
  }

}
```

### breadcrumb_builder

Custom breadcrumb logic. Return `applies()` to indicate which routes this builder handles.

```yaml
services:
  mymodule.breadcrumb_builder:
    class: Drupal\mymodule\Breadcrumb\ProductBreadcrumbBuilder
    tags:
      - { name: breadcrumb_builder, priority: 100 }
```

```php
<?php

namespace Drupal\mymodule\Breadcrumb;

use Drupal\Core\Breadcrumb\Breadcrumb;
use Drupal\Core\Breadcrumb\BreadcrumbBuilderInterface;
use Drupal\Core\Link;
use Drupal\Core\Routing\RouteMatchInterface;

class ProductBreadcrumbBuilder implements BreadcrumbBuilderInterface {

  public function applies(RouteMatchInterface $route_match): bool {
    return $route_match->getRouteName() === 'entity.node.canonical'
      && $route_match->getParameter('node')?->bundle() === 'product';
  }

  public function build(RouteMatchInterface $route_match): Breadcrumb {
    $breadcrumb = new Breadcrumb();
    $breadcrumb->addLink(Link::createFromRoute('Home', '<front>'));
    $breadcrumb->addLink(Link::createFromRoute('Products', 'view.products.page_1'));
    $breadcrumb->addCacheContexts(['route']);
    return $breadcrumb;
  }

}
```

### theme_negotiator

Dynamically select the active theme based on route or request context.

```yaml
services:
  mymodule.theme_negotiator:
    class: Drupal\mymodule\Theme\AdminThemeNegotiator
    tags:
      - { name: theme_negotiator, priority: 10 }
```

```php
<?php

namespace Drupal\mymodule\Theme;

use Drupal\Core\Routing\RouteMatchInterface;
use Drupal\Core\Theme\ThemeNegotiatorInterface;

class AdminThemeNegotiator implements ThemeNegotiatorInterface {

  public function applies(RouteMatchInterface $route_match): bool {
    return str_starts_with($route_match->getRouteName() ?? '', 'mymodule.admin');
  }

  public function determineActiveTheme(RouteMatchInterface $route_match): ?string {
    return 'claro';
  }

}
```

### route_enhancer

Add defaults or parameters to a matched route before the controller executes.

```yaml
services:
  mymodule.route_enhancer:
    class: Drupal\mymodule\Routing\FormatEnhancer
    tags:
      - { name: route_enhancer }
```

---

## Service Priorities

Tags accept a `priority` attribute. Higher values run first. Default is `0`.

```yaml
tags:
  - { name: event_subscriber }          # priority 0 (default)
  - { name: breadcrumb_builder, priority: 100 }  # runs before lower-priority builders
  - { name: path_processor_inbound, priority: -50 }  # runs after default processors
```

For event subscribers, priority can also be set in `getSubscribedEvents()`:

```php
public static function getSubscribedEvents(): array {
  return [
    KernelEvents::REQUEST => ['onRequest', 300],  // 300 = priority
  ];
}
```

---

## Service Decoration

Service decoration replaces an existing service while retaining access to the original via the `inner` keyword. This is the proper way to override core services.

```yaml
services:
  mymodule.custom_path_validator:
    class: Drupal\mymodule\CustomPathValidator
    decorates: path.validator
    arguments:
      - '@mymodule.custom_path_validator.inner'
      - '@logger.channel.mymodule'
```

```php
<?php

namespace Drupal\mymodule;

use Drupal\Core\Path\PathValidatorInterface;
use Psr\Log\LoggerInterface;

class CustomPathValidator implements PathValidatorInterface {

  public function __construct(
    protected readonly PathValidatorInterface $inner,
    protected readonly LoggerInterface $logger,
  ) {}

  public function isValid(string $path): bool {
    $this->logger->debug('Validating path: @path', ['@path' => $path]);
    return $this->inner->isValid($path);
  }

  public function getUrlIfValid(string $path): mixed {
    return $this->inner->getUrlIfValid($path);
  }

  public function getUrlIfValidWithoutAccessCheck(string $path): mixed {
    return $this->inner->getUrlIfValidWithoutAccessCheck($path);
  }

}
```

### Decoration Priority

When multiple modules decorate the same service, `decoration_priority` controls the order. Higher values wrap the outer layer.

```yaml
services:
  mymodule.custom_path_validator:
    class: Drupal\mymodule\CustomPathValidator
    decorates: path.validator
    decoration_priority: 10
    arguments:
      - '@mymodule.custom_path_validator.inner'
```

---

## Service Collectors (Tagged Service Aggregation)

A service collector gathers all services with a specific tag and passes them as an iterable.

### Defining a Collector

```yaml
services:
  mymodule.calculator_manager:
    class: Drupal\mymodule\CalculatorManager
    tags:
      - { name: service_collector, tag: mymodule_calculator, call: addCalculator }
```

Every service tagged with `mymodule_calculator` will have its instance passed to the `addCalculator()` method at container build time.

```yaml
  mymodule.calculator.tax:
    class: Drupal\mymodule\Calculator\TaxCalculator
    tags:
      - { name: mymodule_calculator, priority: 10 }

  mymodule.calculator.discount:
    class: Drupal\mymodule\Calculator\DiscountCalculator
    tags:
      - { name: mymodule_calculator, priority: 20 }
```

```php
<?php

namespace Drupal\mymodule;

class CalculatorManager {

  protected array $calculators = [];

  public function addCalculator(CalculatorInterface $calculator): void {
    $this->calculators[] = $calculator;
  }

  public function calculateTotal(float $amount): float {
    foreach ($this->calculators as $calculator) {
      $amount = $calculator->apply($amount);
    }
    return $amount;
  }

}
```

---

## ServiceModifierInterface

`ServiceModifierInterface` allows a module to modify the container after all services are registered. Implement it in your `ServiceProvider` class.

```php
<?php

namespace Drupal\mymodule;

use Drupal\Core\DependencyInjection\ContainerBuilder;
use Drupal\Core\DependencyInjection\ServiceModifierInterface;

class MymoduleServiceProvider implements ServiceModifierInterface {

  public function alter(ContainerBuilder $container): void {
    // Change the class of an existing service.
    if ($container->hasDefinition('path.validator')) {
      $definition = $container->getDefinition('path.validator');
      $definition->setClass(CustomPathValidator::class);
    }
  }

}
```

**Note:** Prefer service decoration over `ServiceModifierInterface` when you need to wrap behavior. Use `ServiceModifierInterface` only when you need to fundamentally change a service definition (swap class, remove tags, change arguments).

---

## Compiler Passes

For advanced container manipulation, register a compiler pass in your `ServiceProvider`.

```php
<?php

namespace Drupal\mymodule;

use Drupal\Core\DependencyInjection\ContainerBuilder;
use Drupal\Core\DependencyInjection\ServiceProviderBase;
use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;

class MymoduleServiceProvider extends ServiceProviderBase implements CompilerPassInterface {

  public function register(ContainerBuilder $container): void {
    $container->addCompilerPass($this);
  }

  public function process(ContainerBuilder $container): void {
    // Find all services tagged 'mymodule_calculator' and sort by priority.
    $tagged = $container->findTaggedServiceIds('mymodule_calculator');
    foreach ($tagged as $id => $tags) {
      // Custom processing logic.
    }
  }

}
```

Compiler passes run during the container build phase and give full access to all service definitions before the container is compiled and dumped to PHP.
