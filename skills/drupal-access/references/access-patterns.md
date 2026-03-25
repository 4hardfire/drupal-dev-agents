# Advanced Access Patterns in Drupal

## Field-Level Access Control

Drupal provides `hook_entity_field_access()` to control access to individual fields on an entity. This is distinct from entity-level access — a user might be able to view a node but not see a specific field on it.

### hook_entity_field_access()

```php
<?php

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Access\AccessResultInterface;
use Drupal\Core\Field\FieldDefinitionInterface;
use Drupal\Core\Field\FieldItemListInterface;
use Drupal\Core\Session\AccountInterface;

/**
 * Implements hook_entity_field_access().
 */
function mymodule_entity_field_access(string $operation, FieldDefinitionInterface $field_definition, AccountInterface $account, ?FieldItemListInterface $items = NULL): AccessResultInterface {
  // Restrict access to the 'field_internal_notes' field.
  if ($field_definition->getName() === 'field_internal_notes') {
    return AccessResult::forbiddenIf(
      !$account->hasPermission('view internal notes'),
      'User lacks permission to view internal notes.'
    )->addCacheContexts(['user.permissions']);
  }

  // No opinion on other fields.
  return AccessResult::neutral();
}
```

The `$operation` parameter is one of `view`, `edit`, or `delete`. The `$items` parameter is NULL when checking access to the field definition rather than a specific field value.

### Field Access in Entity Access Handlers

You can also override `fieldAccess()` in your entity class for per-entity-type field access logic:

```php
<?php

namespace Drupal\mymodule\Entity;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Access\AccessResultInterface;
use Drupal\Core\Entity\ContentEntityBase;
use Drupal\Core\Field\FieldDefinitionInterface;
use Drupal\Core\Field\FieldItemListInterface;
use Drupal\Core\Session\AccountInterface;

class Item extends ContentEntityBase {

  protected function fieldAccess(string $operation, FieldDefinitionInterface $field_definition, ?AccountInterface $account = NULL, ?FieldItemListInterface $items = NULL): AccessResultInterface {
    if ($field_definition->getName() === 'field_secret' && $operation === 'view') {
      $account = $account ?: \Drupal::currentUser();
      return AccessResult::allowedIfHasPermission($account, 'view secret fields');
    }
    return parent::fieldAccess($operation, $field_definition, $account, $items);
  }

}
```

---

## Node Grants System

The node grants system is Drupal's most powerful (and complex) access mechanism for node entities. It works at the database query level, filtering node listings without loading every entity.

### How Node Grants Work

1. Modules implement `hook_node_access_records()` to assign **grant records** to each node.
2. Modules implement `hook_node_grants()` to assign **grant IDs** to each user.
3. Drupal's node access system matches user grants against node grant records to determine access.

### hook_node_grants()

Returns grants for a given user and operation.

```php
<?php

use Drupal\Core\Session\AccountInterface;

/**
 * Implements hook_node_grants().
 */
function mymodule_node_grants(AccountInterface $account, string $op): array {
  $grants = [];

  if ($op === 'view') {
    // Every authenticated user gets a "public" grant.
    if ($account->isAuthenticated()) {
      $grants['mymodule_access'] = [1];
    }

    // Users with 'view premium content' get a "premium" grant.
    if ($account->hasPermission('view premium content')) {
      $grants['mymodule_access'] = array_merge($grants['mymodule_access'] ?? [], [2]);
    }
  }

  return $grants;
}
```

### hook_node_access_records()

Returns grant records for a given node.

```php
<?php

use Drupal\node\NodeInterface;

/**
 * Implements hook_node_access_records().
 */
function mymodule_node_access_records(NodeInterface $node): array {
  $grants = [];

  if ($node->isPublished()) {
    $grants[] = [
      'realm' => 'mymodule_access',
      'gid' => 1,
      'grant_view' => 1,
      'grant_update' => 0,
      'grant_delete' => 0,
    ];
  }

  if ($node->hasField('field_premium') && $node->get('field_premium')->value) {
    // Only premium grant holders can view this node.
    $grants = [];
    $grants[] = [
      'realm' => 'mymodule_access',
      'gid' => 2,
      'grant_view' => 1,
      'grant_update' => 0,
      'grant_delete' => 0,
    ];
  }

  return $grants;
}
```

After modifying grant logic, rebuild permissions:

```bash
ddev drush node-access-rebuild
```

---

## Checking Access Programmatically

### Entity Access

The most common way to check access is via `EntityInterface::access()`.

```php
use Drupal\node\Entity\Node;

$node = Node::load(42);

// Returns a boolean.
if ($node->access('view')) {
  // Current user can view this node.
}

// Returns an AccessResultInterface (for cacheability).
$access_result = $node->access('update', NULL, TRUE);
if ($access_result->isAllowed()) {
  // Current user can edit this node.
}

// Check against a specific account.
$other_user = \Drupal\user\Entity\User::load(10);
$can_delete = $node->access('delete', $other_user);
```

The third parameter (`$return_as_object`) is critical: when `TRUE`, you get an `AccessResultInterface` with cacheability metadata; when `FALSE` (default), you get a plain boolean.

### Using the Access Manager

For route-level access checks in code:

```php
$access_manager = \Drupal::service('access_manager');

// Check access to a route by name.
$access = $access_manager->checkNamedRoute('entity.node.canonical', ['node' => 42]);
// Returns TRUE or FALSE.

// Get the full AccessResultInterface.
$result = $access_manager->checkNamedRoute('entity.node.canonical', ['node' => 42], NULL, TRUE);
```

### Using EntityTypeManager for Access Checks

```php
$access_handler = \Drupal::entityTypeManager()->getAccessControlHandler('node');

// Check create access.
$can_create = $access_handler->createAccess('article', NULL, [], TRUE);

// Check entity access.
$can_view = $access_handler->access($node, 'view', NULL, TRUE);
```

---

## AccountInterface Methods

The `AccountInterface` (and its proxy `AccountProxyInterface`) provides methods for checking user capabilities.

```php
use Drupal\Core\Session\AccountInterface;

function mymodule_check_user(AccountInterface $account): void {
  // Get user ID.
  $uid = $account->id();

  // Check a single permission.
  $has_perm = $account->hasPermission('access content');

  // Check if the user is authenticated.
  $is_logged_in = $account->isAuthenticated();

  // Check if the user is anonymous.
  $is_anon = $account->isAnonymous();

  // Get all roles.
  $roles = $account->getRoles();

  // Check for a specific role (prefer hasPermission over this).
  $is_admin = in_array('administrator', $account->getRoles(), TRUE);

  // Get the account name.
  $name = $account->getAccountName();

  // Get the email.
  $email = $account->getEmail();
}
```

### Current User

```php
// In procedural code (.module files):
$current_user = \Drupal::currentUser();

// In classes, inject 'current_user' service (AccountProxyInterface).
// The service returns an AccountProxyInterface which wraps AccountInterface.
```

---

## Role-Based vs Permission-Based Access

### Permission-Based (Preferred)

Permissions are the recommended approach because they decouple access logic from the role structure. Site builders can assign permissions to any role without code changes.

```php
// Good: checks a capability, not a role.
AccessResult::allowedIfHasPermission($account, 'administer mymodule settings')
  ->addCacheContexts(['user.permissions']);
```

### Role-Based (Use Sparingly)

Checking roles directly is brittle — it breaks when site builders rename or restructure roles. Use it only when the business rule genuinely ties to a role identity (rare).

```php
// Avoid when possible: ties logic to a specific role name.
$is_editor = in_array('editor', $account->getRoles(), TRUE);
AccessResult::allowedIf($is_editor)
  ->addCacheContexts(['user.roles']);
```

Note the different cache contexts: `user.permissions` for permission checks, `user.roles` for role checks.

---

## Custom Access Policies

For complex access logic that combines multiple conditions, encapsulate the logic in a dedicated service.

```php
<?php

namespace Drupal\mymodule\Access;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Access\AccessResultInterface;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Session\AccountInterface;
use Drupal\mymodule\Entity\Item;

class ItemAccessPolicy {

  public function __construct(
    protected readonly EntityTypeManagerInterface $entityTypeManager,
  ) {}

  public function canView(Item $item, AccountInterface $account): AccessResultInterface {
    // Administrators always have access.
    $admin_access = AccessResult::allowedIfHasPermission($account, 'administer mymodule_item');
    if ($admin_access->isAllowed()) {
      return $admin_access;
    }

    // Owners can always view their own items.
    $is_owner = AccessResult::allowedIf($item->getOwnerId() === (int) $account->id())
      ->addCacheableDependency($item)
      ->cachePerUser();

    // Published items are viewable by anyone with 'view mymodule_item'.
    $published_access = AccessResult::allowedIf($item->isPublished())
      ->addCacheableDependency($item)
      ->andIf(AccessResult::allowedIfHasPermission($account, 'view mymodule_item'));

    return $is_owner->orIf($published_access);
  }

  public function canEdit(Item $item, AccountInterface $account): AccessResultInterface {
    return AccessResult::allowedIf($item->getOwnerId() === (int) $account->id())
      ->addCacheableDependency($item)
      ->cachePerUser()
      ->orIf(AccessResult::allowedIfHasPermission($account, 'administer mymodule_item'));
  }

}
```

Register the policy as a service and inject it into access handlers or controllers:

```yaml
services:
  mymodule.item_access_policy:
    class: Drupal\mymodule\Access\ItemAccessPolicy
    arguments:
      - '@entity_type.manager'
```

Then use it in the entity access handler:

```php
<?php

namespace Drupal\mymodule\Access;

use Drupal\Core\Access\AccessResultInterface;
use Drupal\Core\Entity\EntityAccessControlHandler;
use Drupal\Core\Entity\EntityHandlerInterface;
use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Entity\EntityTypeInterface;
use Drupal\Core\Session\AccountInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

class ItemAccessControlHandler extends EntityAccessControlHandler implements EntityHandlerInterface {

  public function __construct(
    EntityTypeInterface $entity_type,
    protected readonly ItemAccessPolicy $accessPolicy,
  ) {
    parent::__construct($entity_type);
  }

  public static function createInstance(ContainerInterface $container, EntityTypeInterface $entity_type): static {
    return new static(
      $entity_type,
      $container->get('mymodule.item_access_policy'),
    );
  }

  protected function checkAccess(EntityInterface $entity, string $operation, AccountInterface $account): AccessResultInterface {
    /** @var \Drupal\mymodule\Entity\Item $entity */
    return match ($operation) {
      'view' => $this->accessPolicy->canView($entity, $account),
      'update' => $this->accessPolicy->canEdit($entity, $account),
      'delete' => $this->accessPolicy->canEdit($entity, $account),
      default => parent::checkAccess($entity, $operation, $account),
    };
  }

  protected function checkCreateAccess(AccountInterface $account, array $context, ?string $entity_bundle = NULL): AccessResultInterface {
    return \Drupal\Core\Access\AccessResult::allowedIfHasPermission($account, 'create mymodule_item');
  }

}
```

---

## Access in Entity Queries

When building entity queries, access checking is off by default. Always call `accessCheck()` explicitly.

```php
$query = \Drupal::entityTypeManager()->getStorage('node')->getQuery();

// Explicitly enable access checking (filters results to what the current user can see).
$query->accessCheck(TRUE);
$nids = $query->condition('type', 'article')
  ->condition('status', 1)
  ->execute();

// Disable access checking (admin context, cron, migrations).
$query = \Drupal::entityTypeManager()->getStorage('node')->getQuery();
$query->accessCheck(FALSE);
$all_nids = $query->execute();
```

Failing to call `accessCheck()` triggers a deprecation notice in Drupal 10+ and will become a hard error in future versions. Always be explicit.

---

## Common Pitfalls

1. **Forgetting cache contexts** — An access result without `user.permissions` or `user.roles` cache context will be cached for the first user and served to everyone.
2. **Using `allowed()` when `neutral()` is correct** — If your module has no opinion, return neutral. Returning allowed prevents other modules from denying access.
3. **Checking roles instead of permissions** — Ties your code to specific role names. Use permissions for flexibility.
4. **Not calling `accessCheck()` on entity queries** — Results may include entities the user cannot see, or trigger deprecation warnings.
5. **Returning boolean from `_custom_access` callbacks** — Always return `AccessResultInterface`, never a raw boolean.
