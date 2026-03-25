---
name: drupal-services
description: Use when defining Drupal services, implementing dependency injection, creating service decorators, using tagged services, or working with the Symfony service container in Drupal.
version: 1.0.0
---

# Drupal Services & Dependency Injection

## Overview

Drupal's service container is built on the Symfony DependencyInjection component. Every major Drupal subsystem (entity manager, database connection, current user, cache, routing, etc.) is registered as a service in the container. Custom modules define their own services in `mymodule.services.yml`.

Key principles:
- Services are lazy-loaded — instantiated only when first requested.
- Constructor injection is the preferred pattern everywhere.
- `\Drupal::service()` is a static wrapper around the container — use it only in procedural code (`.module` files, legacy hooks). Never use it inside classes that support DI.
- Drupal 11.1+ supports Symfony autowiring for simpler service definitions.

---

## Defining Services in `*.services.yml`

Service definitions live in `mymodule.services.yml` at the module root.

### Basic Service (No Dependencies)

```yaml
services:
  mymodule.greeting:
    class: Drupal\mymodule\GreetingService
```

### Service with Arguments

Reference other services with `@service_id` and parameters with `%parameter_name%`.

```yaml
services:
  mymodule.report_generator:
    class: Drupal\mymodule\ReportGenerator
    arguments:
      - '@entity_type.manager'
      - '@current_user'
      - '%mymodule.max_results%'
```

### Service with a Factory

```yaml
services:
  mymodule.connection:
    class: Drupal\mymodule\CustomConnection
    factory: ['Drupal\mymodule\CustomConnectionFactory', 'create']
    arguments:
      - '@config.factory'
```

### Autowired Service (Drupal 11.1+)

When autowire is enabled, the container resolves constructor type-hints automatically. You only need to list non-autowirable arguments.

```yaml
services:
  _defaults:
    autowire: true

  mymodule.report_generator:
    class: Drupal\mymodule\ReportGenerator
```

### Public vs Private Services

```yaml
services:
  mymodule.internal_helper:
    class: Drupal\mymodule\InternalHelper
    public: false   # Cannot be fetched from the container directly; only injected.
```

### Aliasing

```yaml
services:
  Drupal\mymodule\GreetingServiceInterface:
    alias: mymodule.greeting
```

---

## Constructor Injection Patterns

### In Controllers (ControllerBase::create)

Controllers that extend `ControllerBase` use a static `create()` factory.

```php
<?php

namespace Drupal\mymodule\Controller;

use Drupal\Core\Controller\ControllerBase;
use Drupal\Core\Session\AccountProxyInterface;
use Drupal\mymodule\ReportGenerator;
use Symfony\Component\DependencyInjection\ContainerInterface;

class ReportController extends ControllerBase {

  public function __construct(
    protected readonly ReportGenerator $reportGenerator,
    protected readonly AccountProxyInterface $currentUser,
  ) {}

  public static function create(ContainerInterface $container): static {
    return new static(
      $container->get('mymodule.report_generator'),
      $container->get('current_user'),
    );
  }

  public function view(): array {
    $report = $this->reportGenerator->generate($this->currentUser->id());
    return ['#markup' => $report];
  }

}
```

### In Forms (FormBase / ConfigFormBase)

```php
<?php

namespace Drupal\mymodule\Form;

use Drupal\Core\Form\ConfigFormBase;
use Drupal\Core\Config\ConfigFactoryInterface;
use Drupal\Core\Path\PathValidatorInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

class SettingsForm extends ConfigFormBase {

  public function __construct(
    ConfigFactoryInterface $config_factory,
    protected readonly PathValidatorInterface $pathValidator,
  ) {
    parent::__construct($config_factory);
  }

  public static function create(ContainerInterface $container): static {
    return new static(
      $container->get('config.factory'),
      $container->get('path.validator'),
    );
  }

  // ...
}
```

### In Block Plugins

Block plugins use `ContainerFactoryPluginInterface`.

```php
<?php

namespace Drupal\mymodule\Plugin\Block;

use Drupal\Core\Block\BlockBase;
use Drupal\Core\Plugin\ContainerFactoryPluginInterface;
use Drupal\mymodule\ReportGenerator;
use Symfony\Component\DependencyInjection\ContainerInterface;

#[\Drupal\Core\Block\Attribute\Block(
  id: 'mymodule_report_block',
  admin_label: new \Drupal\Core\StringTranslation\TranslatableMarkup('Report Block'),
)]
class ReportBlock extends BlockBase implements ContainerFactoryPluginInterface {

  public function __construct(
    array $configuration,
    $plugin_id,
    $plugin_definition,
    protected readonly ReportGenerator $reportGenerator,
  ) {
    parent::__construct($configuration, $plugin_id, $plugin_definition);
  }

  public static function create(
    ContainerInterface $container,
    array $configuration,
    $plugin_id,
    $plugin_definition,
  ): static {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition,
      $container->get('mymodule.report_generator'),
    );
  }

  public function build(): array {
    return ['#markup' => $this->reportGenerator->getSummary()];
  }

}
```

### In Event Subscribers

Event subscribers are registered as tagged services — no `create()` needed. The container injects constructor arguments directly.

```php
<?php

namespace Drupal\mymodule\EventSubscriber;

use Drupal\Core\Messenger\MessengerInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\KernelEvents;

class MaintenanceSubscriber implements EventSubscriberInterface {

  public function __construct(
    protected readonly MessengerInterface $messenger,
  ) {}

  public static function getSubscribedEvents(): array {
    return [
      KernelEvents::REQUEST => ['onRequest', 100],
    ];
  }

  public function onRequest(RequestEvent $event): void {
    $this->messenger->addWarning('Scheduled maintenance tonight.');
  }

}
```

```yaml
services:
  mymodule.maintenance_subscriber:
    class: Drupal\mymodule\EventSubscriber\MaintenanceSubscriber
    arguments:
      - '@messenger'
    tags:
      - { name: event_subscriber }
```

### Using ContainerInjectionInterface Directly

For any class that is not a controller, form, or plugin but still needs DI. This is the most generic approach.

```php
<?php

namespace Drupal\mymodule;

use Drupal\Core\DependencyInjection\ContainerInjectionInterface;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

class NodeProcessor implements ContainerInjectionInterface {

  public function __construct(
    protected readonly EntityTypeManagerInterface $entityTypeManager,
  ) {}

  public static function create(ContainerInterface $container): static {
    return new static(
      $container->get('entity_type.manager'),
    );
  }

}
```

---

## ContainerInjectionInterface vs ControllerBase::create()

| Approach | When to use |
|----------|-------------|
| `ControllerBase::create()` | Route controllers — inherits helper methods (`entityTypeManager()`, `config()`, etc.) |
| `ContainerInjectionInterface` | Any standalone class that Drupal instantiates via the container's `create()` convention |
| `ContainerFactoryPluginInterface` | Plugin classes (blocks, field formatters, queue workers, etc.) |
| Direct constructor (no interface) | Services defined in `*.services.yml` — the container handles injection via `arguments:` |

---

## When to Use `\Drupal::service()` vs Proper DI

### Acceptable (procedural / `.module` context)

```php
// In mymodule.module — no class, no DI available.
function mymodule_cron(): void {
  $queue = \Drupal::queue('mymodule_tasks');
  $queue->createItem(['action' => 'cleanup']);
}
```

### Avoid (inside any class)

```php
// BAD — do not call \Drupal::service() inside a class.
class BadExample {
  public function process(): void {
    $manager = \Drupal::service('entity_type.manager'); // Anti-pattern.
  }
}
```

Instead, inject the service through the constructor.

---

## Common Service Tags

| Tag | Purpose | Interface |
|-----|---------|-----------|
| `event_subscriber` | React to kernel / Drupal events | `EventSubscriberInterface` |
| `access_check` | Custom route access logic | `AccessInterface` |
| `path_processor_inbound` | Alter inbound path before routing | `InboundPathProcessorInterface` |
| `path_processor_outbound` | Alter generated URLs | `OutboundPathProcessorInterface` |
| `breadcrumb_builder` | Custom breadcrumbs | `BreadcrumbBuilderInterface` |
| `theme_negotiator` | Select active theme dynamically | `ThemeNegotiatorInterface` |
| `route_enhancer` | Add defaults / parameters to matched routes | `RouteEnhancerInterface` |
| `cache.context` | Custom cache context | `CacheContextInterface` |

See `references/tagged-services.md` for full patterns.

---

## Scaffolding with Drush

Generate a service skeleton quickly:

```bash
# Generate a custom service class + services.yml entry
ddev drush gen service:custom

# Generate a full service provider
ddev drush gen service
```

Drush `gen` will prompt for module name, service name, and class. It creates the class file and appends to `mymodule.services.yml`.

For non-interactive (CI-friendly) usage:

```bash
ddev drush gen service:custom --answers='{"module":"mymodule","service_name":"mymodule.processor","class":"Processor"}'
```

---

## ServiceProvider and ServiceModifier

Modules can alter the container at build time by providing a `ServiceProvider` class.

### Registering a ServiceProvider

Place `MymoduleServiceProvider.php` in the module's `src/` directory. The filename must follow the pattern `{ModuleName}ServiceProvider.php`.

```php
<?php

namespace Drupal\mymodule;

use Drupal\Core\DependencyInjection\ContainerBuilder;
use Drupal\Core\DependencyInjection\ServiceProviderBase;

class MymoduleServiceProvider extends ServiceProviderBase {

  public function register(ContainerBuilder $container): void {
    // Register dynamic services.
  }

  public function alter(ContainerBuilder $container): void {
    // Modify existing service definitions.
    $definition = $container->getDefinition('some.service');
    $definition->setClass(MyCustomOverride::class);
  }

}
```

---

## Related Skills

- **drupal-events** — Event subscribers (which are tagged services) and custom event dispatching.
- **drupal-plugins** — Plugin classes that need `ContainerFactoryPluginInterface` for DI.
- **drupal-hooks** — OOP hooks in Drupal 11+ that may need service injection.
- **drupal-routing** — Route controllers and access checkers that rely on injected services.
- **drupal-config** — Config factory service and config overrides via service providers.
- **drush-generate** — `drush gen service:custom` and related generators.
