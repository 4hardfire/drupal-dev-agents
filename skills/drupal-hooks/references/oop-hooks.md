# Drupal 11+ OOP Hooks Reference

The OOP hook system introduced in Drupal 11 allows hook implementations as class
methods annotated with the `#[Hook]` PHP attribute. Classes are placed in the
module's `src/Hook/` directory and are autowired as services automatically.

---

## The Hook Attribute

The `#[Hook]` attribute is defined in `Drupal\Core\Hook\Attribute\Hook`. It accepts
the following parameters:

```php
#[Hook(
  hook: 'hook_name',       // Required. The hook name without the module prefix.
  module: 'mymodule',      // Optional. Override the module this hook belongs to.
  priority: 0,             // Optional. Higher values execute first. Default is 0.
)]
```

---

## Full Class Examples for Common Hooks

### Entity Hooks

```php
<?php

declare(strict_types=1);

namespace Drupal\mymodule\Hook;

use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Hook\Attribute\Hook;
use Psr\Log\LoggerInterface;

/**
 * Hook implementations for entity operations.
 */
class EntityHooks {

  public function __construct(
    protected readonly EntityTypeManagerInterface $entityTypeManager,
    protected readonly LoggerInterface $logger,
  ) {}

  #[Hook('entity_presave')]
  public function presave(EntityInterface $entity): void {
    if ($entity->getEntityTypeId() === 'node' && $entity->bundle() === 'article') {
      // Auto-populate a computed field before save.
      $title = $entity->label();
      if (empty($entity->get('field_slug')->value)) {
        $entity->set('field_slug', strtolower(preg_replace('/[^a-z0-9]+/i', '-', $title)));
      }
    }
  }

  #[Hook('entity_insert')]
  public function insert(EntityInterface $entity): void {
    if ($entity->getEntityTypeId() === 'node') {
      $this->logger->info('Created @type: @title', [
        '@type' => $entity->bundle(),
        '@title' => $entity->label(),
      ]);
    }
  }

  #[Hook('entity_update')]
  public function update(EntityInterface $entity): void {
    if ($entity->getEntityTypeId() === 'node') {
      $this->logger->info('Updated @type: @title', [
        '@type' => $entity->bundle(),
        '@title' => $entity->label(),
      ]);
    }
  }

  #[Hook('entity_delete')]
  public function delete(EntityInterface $entity): void {
    if ($entity->getEntityTypeId() === 'node') {
      // Clean up related custom storage.
      $this->entityTypeManager->getStorage('mymodule_tracking')
        ->delete(
          $this->entityTypeManager->getStorage('mymodule_tracking')
            ->loadByProperties(['node_id' => $entity->id()])
        );
    }
  }

  #[Hook('entity_access')]
  public function access(EntityInterface $entity, string $operation, \Drupal\Core\Session\AccountInterface $account): \Drupal\Core\Access\AccessResultInterface {
    if ($entity->getEntityTypeId() === 'node' && $entity->bundle() === 'restricted') {
      if ($operation === 'view' && !$account->hasPermission('view restricted content')) {
        return \Drupal\Core\Access\AccessResult::forbidden('No permission to view restricted content.')
          ->addCacheContexts(['user.permissions']);
      }
    }
    return \Drupal\Core\Access\AccessResult::neutral();
  }

}
```

### Form Hooks

```php
<?php

declare(strict_types=1);

namespace Drupal\mymodule\Hook;

use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\Hook\Attribute\Hook;
use Drupal\Core\Session\AccountProxyInterface;
use Drupal\Core\StringTranslation\StringTranslationTrait;

/**
 * Form alter hook implementations.
 */
class FormHooks {

  use StringTranslationTrait;

  public function __construct(
    protected readonly AccountProxyInterface $currentUser,
  ) {}

  #[Hook('form_alter')]
  public function alter(array &$form, FormStateInterface $form_state, string $form_id): void {
    // Add a CSS class to all forms.
    $form['#attributes']['class'][] = 'mymodule-form';
  }

  #[Hook('form_node_article_edit_form_alter')]
  public function articleEditFormAlter(array &$form, FormStateInterface $form_state, string $form_id): void {
    if (!$this->currentUser->hasPermission('administer nodes')) {
      $form['field_internal_notes']['#access'] = FALSE;
    }

    // Add a custom validation handler.
    $form['#validate'][] = [$this, 'validateArticleForm'];
  }

  /**
   * Custom validation handler for the article edit form.
   */
  public function validateArticleForm(array &$form, FormStateInterface $form_state): void {
    $title = $form_state->getValue(['title', 0, 'value']);
    if ($title && strlen($title) < 10) {
      $form_state->setErrorByName('title', $this->t('Title must be at least 10 characters.'));
    }
  }

}
```

### Theme and Preprocess Hooks

```php
<?php

declare(strict_types=1);

namespace Drupal\mymodule\Hook;

use Drupal\Core\Hook\Attribute\Hook;
use Drupal\Core\Routing\RouteMatchInterface;

/**
 * Theme system hook implementations.
 */
class ThemeHooks {

  public function __construct(
    protected readonly RouteMatchInterface $routeMatch,
  ) {}

  #[Hook('theme')]
  public function theme(): array {
    return [
      'mymodule_card' => [
        'variables' => [
          'title' => NULL,
          'body' => NULL,
          'image_url' => NULL,
          'link' => NULL,
        ],
        'template' => 'mymodule-card',
      ],
      'mymodule_listing' => [
        'variables' => [
          'items' => [],
          'empty_message' => NULL,
        ],
        'template' => 'mymodule-listing',
      ],
    ];
  }

  #[Hook('preprocess_node')]
  public function preprocessNode(array &$variables): void {
    $node = $variables['node'];
    $variables['is_front'] = $this->routeMatch->getRouteName() === 'view.frontpage.page_1';
    if ($node->bundle() === 'article' && $node->hasField('field_reading_time')) {
      $variables['reading_time'] = $node->get('field_reading_time')->value;
    }
  }

  #[Hook('theme_suggestions_node')]
  public function themeSuggestionsNode(array $variables): array {
    $suggestions = [];
    $node = $variables['elements']['#node'];
    $view_mode = $variables['elements']['#view_mode'];
    $suggestions[] = 'node__' . $node->bundle() . '__' . $view_mode;
    return $suggestions;
  }

  #[Hook('page_attachments')]
  public function pageAttachments(array &$attachments): void {
    $attachments['#attached']['library'][] = 'mymodule/global';
  }

}
```

### Cron Hook

```php
<?php

declare(strict_types=1);

namespace Drupal\mymodule\Hook;

use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Hook\Attribute\Hook;
use Drupal\Core\State\StateInterface;
use Psr\Log\LoggerInterface;

class CronHooks {

  public function __construct(
    protected readonly EntityTypeManagerInterface $entityTypeManager,
    protected readonly StateInterface $state,
    protected readonly LoggerInterface $logger,
  ) {}

  #[Hook('cron')]
  public function cron(): void {
    $last_run = $this->state->get('mymodule.last_cleanup', 0);

    // Only run once per day.
    if ((\Drupal::time()->getRequestTime() - $last_run) < 86400) {
      return;
    }

    $storage = $this->entityTypeManager->getStorage('node');
    $nids = $storage->getQuery()
      ->condition('type', 'temporary')
      ->condition('created', strtotime('-30 days'), '<')
      ->accessCheck(FALSE)
      ->execute();

    if ($nids) {
      $storage->delete($storage->loadMultiple($nids));
      $this->logger->notice('Cleaned up @count expired nodes.', ['@count' => count($nids)]);
    }

    $this->state->set('mymodule.last_cleanup', \Drupal::time()->getRequestTime());
  }

}
```

---

## Migrating from Procedural to OOP Hooks

### Before (procedural in mymodule.module)

```php
/**
 * Implements hook_entity_presave().
 */
function mymodule_entity_presave(\Drupal\Core\Entity\EntityInterface $entity): void {
  if ($entity->getEntityTypeId() === 'node') {
    \Drupal::logger('mymodule')->info('Saving node: @title', [
      '@title' => $entity->label(),
    ]);
  }
}
```

### After (OOP in src/Hook/EntityHooks.php)

```php
<?php

declare(strict_types=1);

namespace Drupal\mymodule\Hook;

use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Hook\Attribute\Hook;
use Psr\Log\LoggerInterface;

class EntityHooks {

  public function __construct(
    protected readonly LoggerInterface $logger,
  ) {}

  #[Hook('entity_presave')]
  public function presave(EntityInterface $entity): void {
    if ($entity->getEntityTypeId() === 'node') {
      $this->logger->info('Saving node: @title', [
        '@title' => $entity->label(),
      ]);
    }
  }

}
```

### Migration steps

1. Create a class in `src/Hook/` with a descriptive name (e.g., `EntityHooks`, `FormHooks`).
2. Add `use Drupal\Core\Hook\Attribute\Hook;` to imports.
3. Move the function body into a class method.
4. Add `#[Hook('hook_name')]` to the method (hook name without the module prefix).
5. Replace static `\Drupal::service()` calls with constructor-injected dependencies.
6. Remove the procedural function from `.module`.
7. Clear caches: `ddev drush cr`.

**Important**: Do not implement the same hook both procedurally and via OOP in the
same module. This will cause the hook to fire twice.

---

## Hook Attribute Parameters

### hook (required)

The hook name, matching what core invokes. Do not include the module name prefix.

```php
// For hook_form_alter, use 'form_alter':
#[Hook('form_alter')]

// For hook_form_FORM_ID_alter, include the form ID:
#[Hook('form_user_login_form_alter')]

// For hook_ENTITY_TYPE_presave, include the entity type:
#[Hook('node_presave')]
```

### module (optional)

Override which module this hook belongs to. Useful for organizing hooks in a
submodule or implementing a hook on behalf of another module.

```php
#[Hook('form_alter', module: 'other_module')]
public function formAlter(array &$form, FormStateInterface $form_state, string $form_id): void {}
```

### priority (optional)

Controls execution order. Higher values run first. Default is `0`.

```php
// Runs early, before most other implementations.
#[Hook('entity_presave', priority: 100)]
public function earlyPresave(EntityInterface $entity): void {}

// Runs late, after most other implementations.
#[Hook('form_alter', priority: -50)]
public function lateFormAlter(array &$form, FormStateInterface $form_state, string $form_id): void {}
```

---

## Multiple Hooks per Class vs Single Responsibility

### Multiple hooks per class (grouped by domain)

Best when hooks are logically related and share dependencies.

```php
class NodeHooks {

  public function __construct(
    protected readonly EntityTypeManagerInterface $entityTypeManager,
    protected readonly LoggerInterface $logger,
  ) {}

  #[Hook('node_presave')]
  public function presave(NodeInterface $node): void { /* ... */ }

  #[Hook('node_insert')]
  public function insert(NodeInterface $node): void { /* ... */ }

  #[Hook('node_delete')]
  public function delete(NodeInterface $node): void { /* ... */ }

}
```

### Single hook per class

Best for complex hooks that have significant logic and unique dependencies.

```php
class ArticlePresaveHook {

  public function __construct(
    protected readonly SlugGeneratorInterface $slugGenerator,
    protected readonly TaxonomyMapperInterface $taxonomyMapper,
    protected readonly LoggerInterface $logger,
  ) {}

  #[Hook('node_presave')]
  public function __invoke(NodeInterface $node): void {
    if ($node->bundle() !== 'article') {
      return;
    }
    // Complex presave logic with multiple dependencies.
  }

}
```

---

## Alter Hooks in OOP Style

Alter hooks work the same way as regular hooks in OOP. The first argument is passed
by reference, allowing you to modify it.

```php
<?php

declare(strict_types=1);

namespace Drupal\mymodule\Hook;

use Drupal\Core\Hook\Attribute\Hook;

class AlterHooks {

  #[Hook('theme_suggestions_alter')]
  public function themeSuggestionsAlter(array &$suggestions, array $variables, string $hook): void {
    if ($hook === 'page') {
      $suggestions[] = 'page__custom_layout';
    }
  }

  #[Hook('views_data_alter')]
  public function viewsDataAlter(array &$data): void {
    if (isset($data['node_field_data'])) {
      $data['node_field_data']['custom_area'] = [
        'title' => t('Custom area'),
        'help' => t('A custom area handler.'),
        'area' => ['id' => 'mymodule_custom_area'],
      ];
    }
  }

  #[Hook('page_attachments_alter')]
  public function pageAttachmentsAlter(array &$attachments): void {
    // Remove a library added by another module.
    if (isset($attachments['#attached']['library'])) {
      $key = array_search('other_module/unwanted-library', $attachments['#attached']['library']);
      if ($key !== FALSE) {
        unset($attachments['#attached']['library'][$key]);
      }
    }
  }

  #[Hook('menu_links_discovered_alter')]
  public function menuLinksAlter(array &$links): void {
    // Change the title of an existing menu link.
    if (isset($links['system.admin'])) {
      $links['system.admin']['title'] = t('Dashboard');
    }
  }

}
```

---

## Testing OOP Hooks

### Unit Testing

Hook classes with dependency injection can be unit tested by mocking their dependencies.

```php
<?php

namespace Drupal\Tests\mymodule\Unit\Hook;

use Drupal\Core\Entity\EntityInterface;
use Drupal\mymodule\Hook\EntityHooks;
use PHPUnit\Framework\TestCase;
use Psr\Log\LoggerInterface;
use Drupal\Core\Entity\EntityTypeManagerInterface;

class EntityHooksTest extends TestCase {

  public function testPresaveSetsSlug(): void {
    $entityTypeManager = $this->createMock(EntityTypeManagerInterface::class);
    $logger = $this->createMock(LoggerInterface::class);

    $hooks = new EntityHooks($entityTypeManager, $logger);

    $entity = $this->createMock(EntityInterface::class);
    $entity->method('getEntityTypeId')->willReturn('node');
    $entity->method('bundle')->willReturn('article');
    $entity->method('label')->willReturn('My Test Article');

    $field = $this->createMock(\Drupal\Core\Field\FieldItemListInterface::class);
    $field->value = NULL;
    $entity->method('get')->with('field_slug')->willReturn($field);

    $entity->expects($this->once())
      ->method('set')
      ->with('field_slug', 'my-test-article');

    $hooks->presave($entity);
  }

}
```

### Kernel Testing

For integration tests that need Drupal's service container and database.

```php
<?php

namespace Drupal\Tests\mymodule\Kernel\Hook;

use Drupal\KernelTests\KernelTestBase;
use Drupal\node\Entity\Node;
use Drupal\node\Entity\NodeType;

class EntityHooksKernelTest extends KernelTestBase {

  protected static $modules = ['system', 'node', 'user', 'field', 'mymodule'];

  protected function setUp(): void {
    parent::setUp();
    $this->installEntitySchema('node');
    $this->installEntitySchema('user');

    NodeType::create(['type' => 'article', 'name' => 'Article'])->save();
  }

  public function testPresavePopulatesSlug(): void {
    $node = Node::create([
      'type' => 'article',
      'title' => 'My Test Article',
      'uid' => 0,
    ]);
    $node->save();

    // Verify that the hook set the slug field.
    $this->assertEquals('my-test-article', $node->get('field_slug')->value);
  }

}
```

### Verifying Hook Registration

To confirm your OOP hooks are registered correctly:

```bash
# Clear caches to force rediscovery.
ddev drush cr

# List all implementations of a hook (via Devel module or custom code).
ddev drush eval "print_r(array_keys(\Drupal::moduleHandler()->getImplementations('entity_presave')));"
```
