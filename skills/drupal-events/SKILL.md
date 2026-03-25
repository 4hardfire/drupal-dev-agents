---
name: drupal-events
description: Use when creating Drupal event subscribers, dispatching custom events, listening to kernel events, entity events, or configuration events.
version: 1.0.0
---

# Drupal Event System

## Overview

Drupal uses the Symfony EventDispatcher component to allow modules to react to things that happen in the system. Events provide a modern, object-oriented alternative to hooks for decoupled communication between components.

### When to Use Events vs Hooks

| Use Events When | Use Hooks When |
|-----------------|----------------|
| Reacting to kernel-level operations (request, response, exceptions) | Altering forms (`hook_form_alter`) |
| Subscribing to configuration changes | Defining theme implementations (`hook_theme`) |
| Reacting to entity lifecycle operations (Drupal 10.3+) | Providing data to Views (`hook_views_data`) |
| Dispatching your own decoupled notifications | Altering existing plugin definitions (`hook_X_alter`) |
| Needing priority-based ordering of listeners | Working with legacy module APIs |

### Core Concepts

- **Event**: An object that carries data about something that happened. It extends `Drupal\Component\EventDispatcher\Event`.
- **Event Dispatcher**: The service (`event_dispatcher`) that dispatches event objects to registered subscribers.
- **Event Subscriber**: A class implementing `Symfony\Component\EventDispatcher\EventSubscriberInterface` that declares which events it listens to.
- **Priority**: An integer that controls the order in which subscribers are called. Higher numbers run first. Default is `0`.

---

## Event Subscriber

Event subscribers are the primary way to listen for events in Drupal. You need a class and a service definition.

### Complete Example

**File: `src/EventSubscriber/MyModuleSubscriber.php`**

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\EventSubscriber;

use Drupal\Core\Messenger\MessengerInterface;
use Drupal\Core\Session\AccountProxyInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\Event\ResponseEvent;
use Symfony\Component\HttpKernel\KernelEvents;

/**
 * Subscribes to kernel events for my_module.
 */
class MyModuleSubscriber implements EventSubscriberInterface {

  public function __construct(
    protected readonly AccountProxyInterface $currentUser,
    protected readonly MessengerInterface $messenger,
  ) {}

  /**
   * {@inheritdoc}
   */
  public static function getSubscribedEvents(): array {
    // Keys are event names (constants); values are method names or
    // [method, priority] arrays. Higher priority runs first.
    return [
      KernelEvents::REQUEST => ['onRequest', 100],
      KernelEvents::RESPONSE => [
        ['onResponseEarly', 10],
        ['onResponseLate', -10],
      ],
    ];
  }

  /**
   * Reacts to the kernel request event.
   */
  public function onRequest(RequestEvent $event): void {
    // Only act on the main request, not subrequests.
    if (!$event->isMainRequest()) {
      return;
    }

    if ($this->currentUser->isAnonymous()) {
      // Perform logic for anonymous users.
    }
  }

  /**
   * Runs early during response processing.
   */
  public function onResponseEarly(ResponseEvent $event): void {
    $response = $event->getResponse();
    $response->headers->set('X-My-Module', 'active');
  }

  /**
   * Runs late during response processing.
   */
  public function onResponseLate(ResponseEvent $event): void {
    // Additional response processing with lower priority.
  }

}
```

### Service Definition

**File: `my_module.services.yml`**

```yaml
services:
  my_module.event_subscriber:
    class: Drupal\my_module\EventSubscriber\MyModuleSubscriber
    arguments:
      - '@current_user'
      - '@messenger'
    tags:
      - { name: event_subscriber }
```

The `event_subscriber` tag is what tells Drupal's service container to register this class with the event dispatcher. Without it, the subscriber is never called.

### Dependency Injection in Subscribers

Subscribers are standard services, so inject dependencies through the constructor. Common injected services:

```yaml
services:
  my_module.config_subscriber:
    class: Drupal\my_module\EventSubscriber\ConfigSubscriber
    arguments:
      - '@entity_type.manager'
      - '@logger.factory'
      - '@cache_tags.invalidator'
      - '@config.factory'
    tags:
      - { name: event_subscriber }
```

---

## Custom Event

Create custom events when your module needs to notify other modules about something that happened.

### Step 1: Define the Event Class

**File: `src/Event/OrderCompleteEvent.php`**

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Event;

use Drupal\Component\EventDispatcher\Event;
use Drupal\Core\Entity\EntityInterface;

/**
 * Dispatched when an order is completed.
 */
class OrderCompleteEvent extends Event {

  /**
   * The event name.
   */
  const EVENT_NAME = 'my_module.order_complete';

  public function __construct(
    protected readonly EntityInterface $order,
    protected readonly float $totalAmount,
  ) {}

  /**
   * Gets the completed order entity.
   */
  public function getOrder(): EntityInterface {
    return $this->order;
  }

  /**
   * Gets the order total amount.
   */
  public function getTotalAmount(): float {
    return $this->totalAmount;
  }

}
```

> **Important**: Extend `Drupal\Component\EventDispatcher\Event`, not `Symfony\Contracts\EventDispatcher\Event` directly. Drupal's base class ensures compatibility across Symfony versions.

### Step 2: Dispatch the Event from a Service

**File: `src/Service/OrderProcessor.php`**

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Service;

use Drupal\Core\Entity\EntityInterface;
use Drupal\my_module\Event\OrderCompleteEvent;
use Symfony\Contracts\EventDispatcher\EventDispatcherInterface;

/**
 * Processes orders and dispatches completion events.
 */
class OrderProcessor {

  public function __construct(
    protected readonly EventDispatcherInterface $eventDispatcher,
  ) {}

  /**
   * Completes an order and notifies subscribers.
   */
  public function completeOrder(EntityInterface $order, float $total): void {
    // ... business logic to finalize the order ...

    // Dispatch the event so other modules can react.
    $event = new OrderCompleteEvent($order, $total);
    $this->eventDispatcher->dispatch($event, OrderCompleteEvent::EVENT_NAME);

    // Subscribers may have modified the event. You can inspect it here
    // if your event has mutable properties.
  }

}
```

Service definition for the dispatcher injection:

```yaml
services:
  my_module.order_processor:
    class: Drupal\my_module\Service\OrderProcessor
    arguments:
      - '@event_dispatcher'
```

### Step 3: Subscribe to the Custom Event

**File: `src/EventSubscriber/OrderNotificationSubscriber.php`**

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\EventSubscriber;

use Drupal\Core\Logger\LoggerChannelFactoryInterface;
use Drupal\my_module\Event\OrderCompleteEvent;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

/**
 * Sends notifications when orders are completed.
 */
class OrderNotificationSubscriber implements EventSubscriberInterface {

  public function __construct(
    protected readonly LoggerChannelFactoryInterface $loggerFactory,
  ) {}

  /**
   * {@inheritdoc}
   */
  public static function getSubscribedEvents(): array {
    return [
      OrderCompleteEvent::EVENT_NAME => ['onOrderComplete', 0],
    ];
  }

  /**
   * Logs and notifies on order completion.
   */
  public function onOrderComplete(OrderCompleteEvent $event): void {
    $order = $event->getOrder();
    $this->loggerFactory->get('my_module')->info(
      'Order @id completed for @amount.',
      [
        '@id' => $order->id(),
        '@amount' => $event->getTotalAmount(),
      ]
    );
  }

}
```

```yaml
services:
  my_module.order_notification_subscriber:
    class: Drupal\my_module\EventSubscriber\OrderNotificationSubscriber
    arguments:
      - '@logger.factory'
    tags:
      - { name: event_subscriber }
```

---

## Common Core Events

| Category | Key Constants | When They Fire |
|----------|--------------|----------------|
| **Kernel** | `KernelEvents::REQUEST`, `RESPONSE`, `TERMINATE`, `EXCEPTION`, `CONTROLLER`, `VIEW` | HTTP request lifecycle |
| **Entity** | `EntityTypeEvents::CREATE`, `UPDATE`, `DELETE` | Entity type definition changes |
| **Config** | `ConfigEvents::SAVE`, `DELETE`, `RENAME`, `COLLECTION_INFO` | Configuration changes |
| **Routing** | `RoutingEvents::ALTER`, `DYNAMIC`, `FINISHED` | Route collection building |

For a comprehensive catalog of every core event, see `references/core-events.md`.

---

## Scaffolding with Drush

Use `ddev drush generate` (Drush 12+) to scaffold event subscriber code:

```bash
# Generate an event subscriber
ddev drush gen event-subscriber
```

The generator will prompt for:
- Module name
- Event to subscribe to
- Subscriber class name

After generating, always clear the cache:

```bash
ddev drush cr
```

---

## Key Principles

- **Always clear cache** after adding or modifying an event subscriber service definition (`ddev drush cr`).
- **Use `isMainRequest()`** in kernel event subscribers to avoid acting on subrequests (ESI, render arrays, etc.).
- **Set appropriate priorities** -- higher numbers run first. Use priorities to ensure your subscriber runs before or after other modules.
- **Extend `Drupal\Component\EventDispatcher\Event`** for custom events, not the Symfony class directly.
- **Use constructor promotion** (PHP 8.1+) for clean dependency injection in subscriber classes.
- **Multiple methods per event** are supported by returning an array of `[method, priority]` arrays in `getSubscribedEvents()`.
- **Stop propagation** with `$event->stopPropagation()` to prevent lower-priority subscribers from receiving the event.

---

## Related Skills

- **drupal-services** -- How to define services and inject dependencies into event subscribers.
- **drupal-hooks** -- Hook system reference for cases where events are not yet available.
- **drupal-plugins** -- Plugin system for swappable components (different pattern from events).
- **drupal-cache** -- Cache invalidation via events and cache tag management.
- **drupal-config** -- Configuration system events for import/export workflows.
