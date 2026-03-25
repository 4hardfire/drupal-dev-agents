---
name: drupal-routing
description: Use when defining Drupal routes, creating controllers, implementing route subscribers, parameter upcasting, or custom access checks on routes.
version: 1.0.0
---

# Drupal Routing

## Overview

Drupal's routing system is built on top of the Symfony Routing component. It maps
HTTP requests to controller code by matching URL paths, HTTP methods, and other
request properties against a set of route definitions.

Key concepts:
- **Routes** are defined in `*.routing.yml` files inside modules.
- **Controllers** are PHP classes that produce the response for a matched route.
- **Access checks** determine whether the current user may access a route.
- **Parameter upcasting** automatically converts raw path parameters (e.g., a node ID) into loaded entity objects.
- **Route subscribers** allow modules to add or alter routes dynamically at runtime.

## Defining Routes in `*.routing.yml`

Every module can declare routes in a file named `<module_name>.routing.yml` placed
in the module root. Each top-level key is the **route name** (machine name).

### Minimal Route

```yaml
my_module.hello:
  path: '/hello'
  defaults:
    _controller: '\Drupal\my_module\Controller\HelloController::content'
    _title: 'Hello World'
  requirements:
    _permission: 'access content'
```

### Route Properties

| Property | Purpose |
|---|---|
| `path` | URL pattern. May contain parameters like `{node}`. |
| `defaults._controller` | Fully qualified class::method that returns the response. |
| `defaults._form` | Fully qualified form class (instead of `_controller`). |
| `defaults._title` | Static page title string. |
| `defaults._title_callback` | Method that returns a dynamic title. |
| `defaults._entity_form` | Entity form mode, e.g., `node.edit`. |
| `defaults._entity_list` | Entity type ID for the list builder. |
| `requirements._permission` | Permission string the user must have. |
| `requirements._role` | Role ID the user must possess. |
| `requirements._access` | Set to `'TRUE'` to grant unconditional access. |
| `requirements._custom_access` | Class::method returning AccessResultInterface. |
| `requirements._entity_access` | Entity operation check, e.g., `node.view`. |
| `requirements._format` | Restrict to a response format (`html`, `json`). |
| `requirements._csrf_token` | Set to `'TRUE'` to require a CSRF token. |
| `requirements._method` | Allowed HTTP methods, e.g., `GET|POST`. |
| `options._admin_route` | `TRUE` to use the admin theme. |
| `options.parameters` | Parameter type hints for upcasting. |
| `options.no_cache` | `TRUE` to disable page cache for this route. |

### Route with a Form

```yaml
my_module.settings:
  path: '/admin/config/my-module/settings'
  defaults:
    _form: '\Drupal\my_module\Form\SettingsForm'
    _title: 'My Module Settings'
  requirements:
    _permission: 'administer site configuration'
  options:
    _admin_route: TRUE
```

## Controllers

Controllers live in `src/Controller/` inside the module. They typically extend
`ControllerBase` for convenience (provides access to common services) and return
a render array or a `Response` object.

### Basic Controller with Dependency Injection

```php
<?php

namespace Drupal\my_module\Controller;

use Drupal\Core\Controller\ControllerBase;
use Drupal\Core\Database\Connection;
use Symfony\Component\DependencyInjection\ContainerInterface;

class HelloController extends ControllerBase {

  protected Connection $database;

  public function __construct(Connection $database) {
    $this->database = $database;
  }

  public static function create(ContainerInterface $container): static {
    return new static(
      $container->get('database'),
    );
  }

  public function content(): array {
    return [
      '#markup' => $this->t('Hello, world!'),
    ];
  }

}
```

### Returning a Response Object

```php
use Symfony\Component\HttpFoundation\JsonResponse;

public function apiEndpoint(): JsonResponse {
  return new JsonResponse(['status' => 'ok', 'time' => time()]);
}
```

## Entity Parameter Upcasting

When a route path contains a parameter that matches an entity type ID, Drupal
automatically loads the entity. For example, `{node}` is upcasted to a full
`NodeInterface` object.

### Route Definition

```yaml
my_module.node_info:
  path: '/node/{node}/info'
  defaults:
    _controller: '\Drupal\my_module\Controller\NodeInfoController::view'
    _title_callback: '\Drupal\my_module\Controller\NodeInfoController::title'
  requirements:
    _entity_access: 'node.view'
    node: \d+
  options:
    parameters:
      node:
        type: entity:node
```

### Controller Receiving the Entity

```php
<?php

namespace Drupal\my_module\Controller;

use Drupal\Core\Controller\ControllerBase;
use Drupal\node\NodeInterface;

class NodeInfoController extends ControllerBase {

  public function view(NodeInterface $node): array {
    return [
      '#theme' => 'my_module_node_info',
      '#node' => $node,
    ];
  }

  public function title(NodeInterface $node): string {
    return $this->t('Info for @title', ['@title' => $node->label()]);
  }

}
```

### Custom Entity Type Upcasting

For non-core entity types or custom parameters, declare the type explicitly:

```yaml
my_module.product_view:
  path: '/products/{commerce_product}'
  defaults:
    _controller: '\Drupal\my_module\Controller\ProductController::view'
    _title: 'Product'
  requirements:
    _permission: 'access content'
    commerce_product: \d+
  options:
    parameters:
      commerce_product:
        type: entity:commerce_product
```

## Custom Access Checks

### Inline Custom Access

```yaml
my_module.restricted:
  path: '/restricted-page'
  defaults:
    _controller: '\Drupal\my_module\Controller\RestrictedController::content'
    _title: 'Restricted'
  requirements:
    _custom_access: '\Drupal\my_module\Controller\RestrictedController::access'
```

```php
use Drupal\Core\Access\AccessResult;
use Drupal\Core\Session\AccountInterface;

public function access(AccountInterface $account): AccessResult {
  return AccessResult::allowedIf($account->hasPermission('access content')
    && $account->getEmail() !== NULL);
}
```

## Dynamic Routes with RouteSubscriberBase

Use a route subscriber when you need to add routes programmatically or alter
existing routes.

### Service Definition (`my_module.services.yml`)

```yaml
services:
  my_module.route_subscriber:
    class: Drupal\my_module\Routing\MyModuleRouteSubscriber
    tags:
      - { name: event_subscriber }
```

### Route Subscriber Class

```php
<?php

namespace Drupal\my_module\Routing;

use Drupal\Core\Routing\RouteSubscriberBase;
use Symfony\Component\Routing\RouteCollection;

class MyModuleRouteSubscriber extends RouteSubscriberBase {

  protected function alterRoutes(RouteCollection $collection): void {
    // Alter an existing route.
    if ($route = $collection->get('user.login')) {
      $route->setDefault('_title', 'Sign In');
    }

    // Add a dynamic route.
    $route = new \Symfony\Component\Routing\Route(
      '/dynamic-page',
      [
        '_controller' => '\Drupal\my_module\Controller\DynamicController::content',
        '_title' => 'Dynamic Page',
      ],
      [
        '_permission' => 'access content',
      ]
    );
    $collection->add('my_module.dynamic_page', $route);
  }

}
```

## Menu Links

### Menu Links (`my_module.links.menu.yml`)

```yaml
my_module.admin:
  title: 'My Module'
  description: 'Configure My Module settings.'
  route_name: my_module.settings
  parent: system.admin_config
  weight: 10
```

### Local Tasks / Tabs (`my_module.links.task.yml`)

```yaml
my_module.settings_tab:
  title: 'Settings'
  route_name: my_module.settings
  base_route: my_module.settings
```

### Local Actions (`my_module.links.action.yml`)

```yaml
my_module.add_item:
  title: 'Add item'
  route_name: my_module.item_add
  appears_on:
    - my_module.item_list
```

## Scaffolding with Drush

Generate boilerplate quickly:

```bash
# Generate a controller with route definition
ddev drush gen controller

# Generate a route subscriber service
ddev drush gen route-subscriber
```

These commands create the routing YAML entry, the PHP class, and the service
definition (when applicable) in one step.

## Related Skills

- **drupal-forms** -- Building forms referenced by `_form` in routes.
- **drupal-access** -- Detailed access check services and policies.
- **drupal-services** -- Dependency injection and service definitions.
- **drupal-entities** -- Entity types, storage, and parameter upcasting details.
- **drupal-render** -- Render arrays returned by controllers.
- **drupal-hooks** -- Hook-based alterations that complement route subscribers.
