---
name: drupal-hooks
description: Use when implementing Drupal hooks, both legacy procedural hooks (hook_form_alter, hook_entity_presave, hook_theme) and Drupal 11+ OOP hooks using the #[Hook] attribute.
version: 1.0.0
---

# Drupal Hooks

## Overview

Drupal's hook system is the primary extension mechanism that allows modules to alter
and extend core and contributed module behavior without modifying their source code.
When Drupal reaches certain points during execution (rendering a page, saving an
entity, processing a form, running cron), it invokes hooks — named extension points
that any module can implement.

### How Hooks Are Discovered and Invoked

**Legacy procedural hooks** are discovered by naming convention. Drupal's module handler
scans enabled modules for functions matching the pattern `{modulename}_{hookname}()`.
For example, if a module named `mymodule` implements `hook_form_alter`, Drupal looks
for a function called `mymodule_form_alter()` in `mymodule.module`.

The `ModuleHandlerInterface::invokeAll()` method is responsible for calling all
implementations of a given hook across all enabled modules. The invocation order is
determined by module weight (set in the `system` table) and alphabetical order for
modules with equal weight.

**OOP hooks** (Drupal 11+) are discovered via PHP attributes. Drupal scans classes in
each module's `src/Hook/` directory for methods annotated with the `#[Hook]` attribute.
These are registered during container compilation and invoked through the same module
handler system.

### Hook Invocation Flow

1. Core or a module calls `\Drupal::moduleHandler()->invokeAll('hook_name', $args)`.
2. The module handler iterates through all enabled modules.
3. For each module, it checks for a procedural function or a registered OOP hook.
4. Each implementation is called with the provided arguments.
5. Some hooks collect return values; alter hooks modify arguments by reference.

---

## Legacy Procedural Hooks

Legacy procedural hooks are defined as functions in a module's `.module` file (or in
some cases `.install`, `.theme`, or other specific files). The function name follows
the pattern `{modulename}_{hookname}()`.

### hook_form_alter / hook_form_FORM_ID_alter

Allows modules to alter any form before it is rendered.

```php
/**
 * Implements hook_form_alter().
 */
function mymodule_form_alter(array &$form, \Drupal\Core\Form\FormStateInterface $form_state, string $form_id): void {
  if ($form_id === 'node_article_edit_form') {
    $form['title']['widget'][0]['value']['#description'] = t('Enter a descriptive title.');
  }
}

/**
 * Implements hook_form_FORM_ID_alter() for node_article_edit_form.
 */
function mymodule_form_node_article_edit_form_alter(array &$form, \Drupal\Core\Form\FormStateInterface $form_state, string $form_id): void {
  // This is only called for the node_article_edit_form form.
  $form['field_summary']['#access'] = FALSE;
}
```

### hook_entity_presave / hook_entity_insert / hook_entity_update / hook_entity_delete

React to entity lifecycle events.

```php
/**
 * Implements hook_entity_presave().
 */
function mymodule_entity_presave(\Drupal\Core\Entity\EntityInterface $entity): void {
  if ($entity->getEntityTypeId() === 'node' && $entity->bundle() === 'article') {
    // Set a field value before the entity is saved.
    $entity->set('field_last_reviewed', date('Y-m-d'));
  }
}

/**
 * Implements hook_entity_insert().
 */
function mymodule_entity_insert(\Drupal\Core\Entity\EntityInterface $entity): void {
  if ($entity->getEntityTypeId() === 'node') {
    \Drupal::logger('mymodule')->notice('New node created: @title', [
      '@title' => $entity->label(),
    ]);
  }
}

/**
 * Implements hook_entity_update().
 */
function mymodule_entity_update(\Drupal\Core\Entity\EntityInterface $entity): void {
  // React after an entity has been updated.
}

/**
 * Implements hook_entity_delete().
 */
function mymodule_entity_delete(\Drupal\Core\Entity\EntityInterface $entity): void {
  // Clean up related data when an entity is deleted.
}
```

### hook_theme

Registers theme implementations (templates and theme functions).

```php
/**
 * Implements hook_theme().
 */
function mymodule_theme(): array {
  return [
    'mymodule_custom_block' => [
      'variables' => [
        'title' => NULL,
        'content' => NULL,
        'attributes' => [],
      ],
      'template' => 'mymodule-custom-block',
    ],
  ];
}
```

The corresponding Twig template goes in `templates/mymodule-custom-block.html.twig`.

### hook_cron

Executes periodic tasks during cron runs.

```php
/**
 * Implements hook_cron().
 */
function mymodule_cron(): void {
  // Perform periodic cleanup or synchronization tasks.
  $storage = \Drupal::entityTypeManager()->getStorage('node');
  $query = $storage->getQuery()
    ->condition('type', 'temporary')
    ->condition('created', strtotime('-30 days'), '<')
    ->accessCheck(FALSE);
  $nids = $query->execute();

  if ($nids) {
    $nodes = $storage->loadMultiple($nids);
    $storage->delete($nodes);
    \Drupal::logger('mymodule')->notice('Deleted @count old temporary nodes.', [
      '@count' => count($nids),
    ]);
  }
}
```

### hook_install / hook_uninstall

Run when a module is installed or uninstalled. Defined in `mymodule.install`.

```php
/**
 * Implements hook_install().
 */
function mymodule_install(): void {
  // Set default configuration values on install.
  \Drupal::configFactory()->getEditable('mymodule.settings')
    ->set('enabled', TRUE)
    ->set('max_items', 50)
    ->save();
}

/**
 * Implements hook_uninstall().
 */
function mymodule_uninstall(): void {
  // Clean up any data created by this module.
  \Drupal::state()->delete('mymodule.last_run');
}
```

---

## Drupal 11+ OOP Hooks

Starting with Drupal 11, hooks can be implemented as methods on classes using PHP
attributes. This approach brings full dependency injection support, better IDE
integration, and clearer code organization.

### How It Works

1. Create a class in your module's `src/Hook/` directory.
2. Annotate methods with the `#[Hook('hook_name')]` attribute.
3. Drupal discovers these classes automatically during container compilation.
4. Dependencies are injected through the constructor (autowired by the container).

### Basic OOP Hook Example

```php
<?php

declare(strict_types=1);

namespace Drupal\mymodule\Hook;

use Drupal\Core\Hook\Attribute\Hook;
use Drupal\Core\Entity\EntityInterface;
use Psr\Log\LoggerInterface;

/**
 * Entity hook implementations for mymodule.
 */
class EntityHooks {

  public function __construct(
    protected readonly LoggerInterface $logger,
  ) {}

  #[Hook('entity_presave')]
  public function entityPresave(EntityInterface $entity): void {
    if ($entity->getEntityTypeId() === 'node' && $entity->bundle() === 'article') {
      $entity->set('field_last_reviewed', date('Y-m-d'));
    }
  }

  #[Hook('entity_insert')]
  public function entityInsert(EntityInterface $entity): void {
    if ($entity->getEntityTypeId() === 'node') {
      $this->logger->notice('New node created: @title', [
        '@title' => $entity->label(),
      ]);
    }
  }

}
```

### Form Alter Hooks in OOP Style

```php
<?php

declare(strict_types=1);

namespace Drupal\mymodule\Hook;

use Drupal\Core\Hook\Attribute\Hook;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\Session\AccountProxyInterface;

/**
 * Form hook implementations for mymodule.
 */
class FormHooks {

  public function __construct(
    protected readonly AccountProxyInterface $currentUser,
  ) {}

  #[Hook('form_alter')]
  public function formAlter(array &$form, FormStateInterface $form_state, string $form_id): void {
    if ($form_id === 'node_article_edit_form' && !$this->currentUser->hasPermission('administer nodes')) {
      $form['field_internal_notes']['#access'] = FALSE;
    }
  }

  #[Hook('form_node_article_edit_form_alter')]
  public function articleFormAlter(array &$form, FormStateInterface $form_state, string $form_id): void {
    $form['title']['widget'][0]['value']['#description'] = t('Enter a descriptive title.');
  }

}
```

### Multiple Hooks on One Class

A single class can implement many hooks. Group related hooks together for cohesion.

```php
<?php

declare(strict_types=1);

namespace Drupal\mymodule\Hook;

use Drupal\Core\Hook\Attribute\Hook;

/**
 * Theme and render hook implementations.
 */
class ThemeHooks {

  #[Hook('theme')]
  public function theme(): array {
    return [
      'mymodule_dashboard' => [
        'variables' => ['items' => [], 'title' => NULL],
        'template' => 'mymodule-dashboard',
      ],
    ];
  }

  #[Hook('preprocess_node')]
  public function preprocessNode(array &$variables): void {
    $node = $variables['node'];
    if ($node->bundle() === 'article') {
      $variables['custom_class'] = 'article--featured';
    }
  }

}
```

### Hook Ordering and Priority

You can control hook execution order using the `order` parameter.

The order parameter accepts the following simple ordering options.
`\Drupal\Core\Hook\Order\Order::First`
`\Drupal\Core\Hook\Order\Order::Last`

The order parameter also accepts the following complex ordering options which also accept parameters.
`\Drupal\Core\Hook\Order\OrderBefore`
`\Drupal\Core\Hook\Order\OrderAfter`

`OrderBefore()` and `OrderAfter()` accept the following parameters:
modules: an array of modules to order before or after.
classesAndMethods: an array of arrays of classes and methods to order before or after.

```php
#[Hook('entity_presave', order: Order::first)]
public function latePresave(EntityInterface $entity): void {
  // Runs before other entity_presave hooks.
}

#[Hook('entity_presave', order: Order::last)]
public function latePresave(EntityInterface $entity): void {
  // Runs after other entity_presave hooks.
}

#[Hook('entity_presave', order: new OrderBefore(['other_module']))]
public function latePresave(EntityInterface $entity): void {
  // Runs before before another module's implementation.
}

#[Hook('entity_presave', order: new OrderAfter(['other_module']))]
public function latePresave(EntityInterface $entity): void {
  // Runs after another module's implementation.
}

```

### Dependency Injection in Hook Classes

Hook classes are registered as services in the container. Constructor arguments are
autowired automatically — you do not need to define the service manually in
`mymodule.services.yml`. Any service available in the container can be injected.

```php
<?php

declare(strict_types=1);

namespace Drupal\mymodule\Hook;

use Drupal\Core\Hook\Attribute\Hook;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Config\ConfigFactoryInterface;
use Drupal\Core\Messenger\MessengerInterface;
use Psr\Log\LoggerInterface;

class CronHooks {

  public function __construct(
    protected readonly EntityTypeManagerInterface $entityTypeManager,
    protected readonly ConfigFactoryInterface $configFactory,
    protected readonly MessengerInterface $messenger,
    protected readonly LoggerInterface $logger,
  ) {}

  #[Hook('cron')]
  public function cron(): void {
    $config = $this->configFactory->get('mymodule.settings');
    $max_age = $config->get('max_age') ?? 30;

    $storage = $this->entityTypeManager->getStorage('node');
    $nids = $storage->getQuery()
      ->condition('type', 'temporary')
      ->condition('created', strtotime("-{$max_age} days"), '<')
      ->accessCheck(FALSE)
      ->execute();

    if ($nids) {
      $storage->delete($storage->loadMultiple($nids));
      $this->logger->notice('Deleted @count expired temporary nodes.', [
        '@count' => count($nids),
      ]);
    }
  }

}
```

---

## When to Use Hooks vs Event Subscribers

| Use Case | Mechanism |
|---|---|
| Altering forms, entities, render arrays | Hooks (alter hooks) |
| Responding to entity CRUD operations | Hooks or entity events |
| Responding to kernel events (request, response, exception) | Event subscribers |
| Reacting to route-level events (access, matching) | Event subscribers |
| Modifying configuration on import/export | Config events (event subscribers) |
| Integrating with Symfony-level systems | Event subscribers |

**Rule of thumb**: If Drupal core invokes it via `ModuleHandler::invokeAll()` or
`ModuleHandler::alter()`, it is a hook. If it is dispatched via Symfony's
`EventDispatcherInterface`, use an event subscriber. When both are available for the
same operation (e.g., entity events), prefer whichever gives you cleaner code and
better DI — in Drupal 11+, OOP hooks close the DI gap.

---

## Scaffolding with Drush

Use Drush code generator to scaffold hook implementations quickly:

```bash
# Generate a procedural hook implementation
ddev drush generate hook

# You will be prompted to select the hook and target module.
# Drush generates the function signature with proper docblock.
```

For OOP hooks, you can scaffold the class manually or use contributed generators
that support the `#[Hook]` attribute pattern.

---

## Related Skills

- **drupal-module-development** — Module structure, services, routing, and controllers.
- **drupal-forms** — Building and altering forms with the Form API.
- **drupal-entities** — Entity types, fields, storage, and entity API.
- **drupal-events** — Symfony event subscribers in Drupal.
- **drupal-theming** — Theme system, Twig templates, and render arrays.
- **drupal-cron** — Cron system, queue workers, and scheduled tasks.
