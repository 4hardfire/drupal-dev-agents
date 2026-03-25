---
name: drupal-access
description: Use when implementing Drupal permissions, custom access checks, entity access control handlers, route access requirements, or AccessResult logic.
version: 1.0.0
---

# Drupal Access Control System

## Overview

Drupal's access system determines whether a user can perform an action — viewing a page, editing an entity, accessing a REST resource, or executing a route. It is built on three pillars:

1. **Permissions** — granular capabilities assigned to roles (`administer nodes`, `access content`).
2. **Route access requirements** — declarations in `*.routing.yml` that gate every route.
3. **Entity access handlers** — per-entity-type classes that control create / view / update / delete.

All access decisions produce an `AccessResult` object (allowed, forbidden, or neutral) that carries cacheability metadata. This ensures access outcomes integrate correctly with Drupal's render cache and Dynamic Page Cache.

Key principles:
- Access checking is **mandatory** — every route must declare at least one access requirement.
- `AccessResult::forbidden()` always wins. If any check returns forbidden, access is denied regardless of other results.
- `AccessResult::neutral()` means "I have no opinion." It does not grant access on its own.
- Always attach cache contexts and tags to access results so they vary and invalidate correctly.

---

## Permissions

### Static Permissions (*.permissions.yml)

Define permissions in `mymodule.permissions.yml` at the module root.

```yaml
administer mymodule settings:
  title: 'Administer My Module settings'
  description: 'Access the configuration page for My Module.'
  restrict access: true

view mymodule reports:
  title: 'View My Module reports'
  description: 'Allow users to view generated reports.'
```

The `restrict access: true` flag causes Drupal to display a security warning on the permissions page for that permission.

### Dynamic Permissions (Callback)

For permissions that depend on configuration or content (e.g., one permission per vocabulary), use a callback.

```yaml
permission_callbacks:
  - Drupal\mymodule\MymodulePermissions::permissions
```

```php
<?php

namespace Drupal\mymodule;

use Drupal\Core\StringTranslation\StringTranslationTrait;
use Drupal\taxonomy\Entity\Vocabulary;

class MymodulePermissions {

  use StringTranslationTrait;

  public function permissions(): array {
    $permissions = [];
    foreach (Vocabulary::loadMultiple() as $vocabulary) {
      $permissions['edit terms in ' . $vocabulary->id()] = [
        'title' => $this->t('Edit terms in %name', ['%name' => $vocabulary->label()]),
      ];
    }
    return $permissions;
  }

}
```

---

## Route Access Requirements

Access requirements are declared in `mymodule.routing.yml` under the `requirements` key.

### _permission

Checks that the current user has a specific permission.

```yaml
mymodule.reports:
  path: '/admin/reports/mymodule'
  defaults:
    _controller: '\Drupal\mymodule\Controller\ReportController::list'
    _title: 'My Module Reports'
  requirements:
    _permission: 'view mymodule reports'
```

Combine multiple permissions with `+` (AND) or `,` (OR):

```yaml
requirements:
  _permission: 'access content,view mymodule reports'   # OR — either permission grants access
  _permission: 'access content+view mymodule reports'    # AND — both required
```

### _role

Checks that the user has a specific role. Prefer `_permission` over `_role` for finer-grained control.

```yaml
requirements:
  _role: 'administrator'
```

### _access

Unconditional access — `'TRUE'` allows everyone (including anonymous), `'FALSE'` denies everyone.

```yaml
requirements:
  _access: 'TRUE'
```

### _entity_access

Checks entity-level access for an entity parameter in the route.

```yaml
mymodule.node_report:
  path: '/node/{node}/report'
  defaults:
    _controller: '\Drupal\mymodule\Controller\ReportController::nodeReport'
  requirements:
    _entity_access: 'node.view'
  options:
    parameters:
      node:
        type: entity:node
```

The format is `{parameter_name}.{operation}` where operation is `view`, `update`, `delete`, or a custom operation string.

### _custom_access

Points to a method that returns an `AccessResultInterface`.

```yaml
mymodule.protected:
  path: '/mymodule/protected/{item}'
  defaults:
    _controller: '\Drupal\mymodule\Controller\ProtectedController::view'
  requirements:
    _custom_access: '\Drupal\mymodule\Access\ItemAccessCheck::access'
```

The method receives the route parameters (upcasted) plus `AccountInterface $account`:

```php
<?php

namespace Drupal\mymodule\Access;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Access\AccessResultInterface;
use Drupal\Core\Session\AccountInterface;
use Drupal\mymodule\Entity\Item;

class ItemAccessCheck {

  public function access(AccountInterface $account, Item $item = NULL): AccessResultInterface {
    if ($item === NULL) {
      return AccessResult::forbidden('Item not found.');
    }
    return AccessResult::allowedIf($item->getOwnerId() === (int) $account->id())
      ->addCacheableDependency($item)
      ->cachePerUser();
  }

}
```

---

## Custom Access Checker Service

For reusable access logic tied to a route requirement key, implement `AccessInterface` and register a tagged service.

### Service Definition

```yaml
services:
  mymodule.access_checker.premium:
    class: Drupal\mymodule\Access\PremiumAccessCheck
    arguments:
      - '@current_user'
      - '@entity_type.manager'
    tags:
      - { name: access_check, applies_to: _mymodule_premium_access }
```

### Access Checker Class

```php
<?php

namespace Drupal\mymodule\Access;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Access\AccessResultInterface;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Routing\Access\AccessInterface;
use Drupal\Core\Session\AccountInterface;
use Symfony\Component\Routing\Route;

class PremiumAccessCheck implements AccessInterface {

  public function __construct(
    protected readonly AccountInterface $currentUser,
    protected readonly EntityTypeManagerInterface $entityTypeManager,
  ) {}

  public function access(Route $route, AccountInterface $account): AccessResultInterface {
    if ($account->hasPermission('access premium content')) {
      return AccessResult::allowed()
        ->addCacheContexts(['user.permissions']);
    }
    return AccessResult::neutral('User does not have premium access.')
      ->addCacheContexts(['user.permissions']);
  }

}
```

### Using the Checker in Routing

```yaml
mymodule.premium_page:
  path: '/premium'
  defaults:
    _controller: '\Drupal\mymodule\Controller\PremiumController::view'
  requirements:
    _mymodule_premium_access: 'TRUE'
```

---

## Entity Access Control Handler

Each entity type can specify an access handler. Extend `EntityAccessControlHandler` and override `checkAccess()` and `checkCreateAccess()`.

### Declaring the Handler in the Entity Annotation / Attribute

```php
#[\Drupal\Core\Entity\Attribute\ContentEntityType(
  id: 'mymodule_item',
  label: new \Drupal\Core\StringTranslation\TranslatableMarkup('Item'),
  handlers: [
    'access' => \Drupal\mymodule\Access\ItemAccessControlHandler::class,
    // ... other handlers
  ],
)]
```

### Implementing the Handler

```php
<?php

namespace Drupal\mymodule\Access;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Access\AccessResultInterface;
use Drupal\Core\Entity\EntityAccessControlHandler;
use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Session\AccountInterface;

class ItemAccessControlHandler extends EntityAccessControlHandler {

  protected function checkAccess(EntityInterface $entity, string $operation, AccountInterface $account): AccessResultInterface {
    switch ($operation) {
      case 'view':
        return AccessResult::allowedIfHasPermission($account, 'view mymodule_item');

      case 'update':
        return AccessResult::allowedIf($entity->getOwnerId() === (int) $account->id())
          ->addCacheableDependency($entity)
          ->cachePerUser()
          ->orIf(AccessResult::allowedIfHasPermission($account, 'administer mymodule_item'));

      case 'delete':
        return AccessResult::allowedIfHasPermission($account, 'administer mymodule_item');

      default:
        return AccessResult::neutral();
    }
  }

  protected function checkCreateAccess(AccountInterface $account, array $context, ?string $entity_bundle = NULL): AccessResultInterface {
    return AccessResult::allowedIfHasPermission($account, 'create mymodule_item');
  }

}
```

---

## AccessResult API

All methods live in `Drupal\Core\Access\AccessResult`.

### Factory Methods

| Method | Returns | Meaning |
|--------|---------|---------|
| `AccessResult::allowed()` | `AccessResultAllowed` | Grants access (can be overridden by forbidden) |
| `AccessResult::forbidden($reason)` | `AccessResultForbidden` | Denies access — always wins |
| `AccessResult::neutral($reason)` | `AccessResultNeutral` | No opinion — does not grant access alone |
| `AccessResult::allowedIfHasPermission($account, $permission)` | `AccessResultInterface` | Allowed if account has permission, neutral otherwise |
| `AccessResult::allowedIf($condition)` | `AccessResultInterface` | Allowed if boolean is true, neutral otherwise |
| `AccessResult::forbiddenIf($condition, $reason)` | `AccessResultInterface` | Forbidden if boolean is true, neutral otherwise |

### Combining Results

| Method | Logic |
|--------|-------|
| `$a->orIf($b)` | Allowed if either is allowed (unless one is forbidden) |
| `$a->andIf($b)` | Allowed only if both are allowed |

### Cacheability

Access results implement `CacheableDependencyInterface`. Always attach cacheability metadata.

```php
AccessResult::allowed()
  ->addCacheContexts(['user.permissions'])   // Varies by permission set
  ->addCacheTags(['node:42'])                // Invalidates when node 42 changes
  ->cachePerUser()                           // Shorthand for user cache context
  ->addCacheableDependency($entity);         // Merges entity's cache metadata
```

### Checking Results

```php
$result->isAllowed();    // true only for AccessResultAllowed
$result->isForbidden();  // true only for AccessResultForbidden
$result->isNeutral();    // true only for AccessResultNeutral
```

---

## Related Skills

- **drupal-services** — Registering access checker services with dependency injection.
- **drupal-routing** — Route definitions, parameters, and access requirement declarations.
- **drupal-hooks** — `hook_entity_access`, `hook_node_access`, and other access-related hooks.
- **drupal-plugins** — Entity type plugins that declare access handlers.
- **drupal-config** — Permission configuration and role management.
