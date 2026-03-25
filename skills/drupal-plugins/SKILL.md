---
name: drupal-plugins
description: Use when creating Drupal plugins (blocks, field types, field widgets, field formatters, actions, queue workers, conditions, views plugins, constraints) or working with the Drupal plugin system, annotations, or Symfony attributes for plugins.
version: 1.0.0
---

# Drupal Plugin System

## Overview

Drupal's plugin system is a set of reusable, swappable components that follow a common pattern. Plugins are small pieces of functionality that are interchangeable -- for example, image effects, blocks, field types, and field formatters are all plugins.

### Core Concepts

- **Plugin Type**: A category of plugins that share a purpose and interface (e.g., Block, FieldType, Action).
- **Plugin Discovery**: The mechanism by which Drupal finds plugins. Most common types use **annotated class discovery** (Drupal 10) or **attribute-based discovery** (Drupal 11+).
- **Plugin Factory**: Creates plugin instances. The default factory is `DefaultFactory`.
- **Plugin Manager**: A service that handles discovery, instantiation, and caching for a specific plugin type. Each plugin type has its own manager service (e.g., `plugin.manager.block`).
- **Plugin Definition**: Metadata about a plugin (its ID, label, category, etc.) provided through annotations or attributes.
- **Plugin Derivative**: A mechanism to dynamically generate multiple plugin definitions from a single class (e.g., one block per menu, one block per view).

### Discovery Methods

| Method | Description | Usage |
|--------|-------------|-------|
| Annotated Class Discovery | Finds plugins via `@PluginType` doc-block annotations | Drupal 10 (still works in 11) |
| Attribute Discovery | Finds plugins via PHP 8.1+ `#[PluginType]` attributes | Drupal 11+ (recommended) |
| YAML Discovery | Reads plugin definitions from `.yml` files | menu links, local tasks |
| Hook Discovery | Uses hooks to find plugins | Rare, legacy |

### Plugin Managers

Every plugin type is managed by a service that extends `DefaultPluginManager`. To get a plugin instance:

```php
// Inject the manager as a service dependency.
$block_manager = \Drupal::service('plugin.manager.block');

// Get a plugin instance.
$plugin = $block_manager->createInstance('my_module_example_block', $configuration);

// Get all definitions.
$definitions = $block_manager->getDefinitions();
```

---

## Block Plugin

Blocks are one of the most commonly created plugin types. They appear in the block layout UI and can be placed in regions.

### Drupal 10 (Annotation Syntax)

```php
<?php

namespace Drupal\my_module\Plugin\Block;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Block\BlockBase;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\Session\AccountInterface;

/**
 * Provides a 'Welcome' block.
 *
 * @Block(
 *   id = "my_module_welcome_block",
 *   admin_label = @Translation("Welcome block"),
 *   category = @Translation("Custom"),
 * )
 */
class WelcomeBlock extends BlockBase {

  /**
   * {@inheritdoc}
   */
  public function defaultConfiguration(): array {
    return [
      'welcome_text' => 'Hello, welcome to our site!',
    ];
  }

  /**
   * {@inheritdoc}
   */
  public function blockForm($form, FormStateInterface $form_state): array {
    $form['welcome_text'] = [
      '#type' => 'textarea',
      '#title' => $this->t('Welcome text'),
      '#default_value' => $this->configuration['welcome_text'],
      '#required' => TRUE,
    ];
    return $form;
  }

  /**
   * {@inheritdoc}
   */
  public function blockValidate($form, FormStateInterface $form_state): void {
    if (strlen($form_state->getValue('welcome_text')) < 5) {
      $form_state->setErrorByName('welcome_text', $this->t('Welcome text must be at least 5 characters.'));
    }
  }

  /**
   * {@inheritdoc}
   */
  public function blockSubmit($form, FormStateInterface $form_state): void {
    $this->configuration['welcome_text'] = $form_state->getValue('welcome_text');
  }

  /**
   * {@inheritdoc}
   */
  public function build(): array {
    return [
      '#markup' => $this->configuration['welcome_text'],
      '#cache' => [
        'contexts' => ['user.roles'],
        'tags' => ['config:my_module.settings'],
        'max-age' => 3600,
      ],
    ];
  }

  /**
   * {@inheritdoc}
   */
  protected function blockAccess(AccountInterface $account): AccessResult {
    return AccessResult::allowedIfHasPermission($account, 'access content');
  }

}
```

### Drupal 11+ (Attribute Syntax)

```php
<?php

namespace Drupal\my_module\Plugin\Block;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Block\Attribute\Block;
use Drupal\Core\Block\BlockBase;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\Session\AccountInterface;
use Drupal\Core\StringTranslation\TranslatableMarkup;

#[Block(
  id: 'my_module_welcome_block',
  admin_label: new TranslatableMarkup('Welcome block'),
  category: new TranslatableMarkup('Custom'),
)]
class WelcomeBlock extends BlockBase {

  // Method implementations are identical to the annotation example above.
  // See the annotation example for defaultConfiguration(), blockForm(),
  // blockValidate(), blockSubmit(), build(), and blockAccess().

}
```

> **Key difference**: In D11+ attributes, translatable strings use `new TranslatableMarkup('...')` instead of `@Translation("...")`.

---

## Field Type Plugin

Field types define how data is stored in the database for a field.

### Drupal 10 (Annotation Syntax)

```php
<?php

namespace Drupal\my_module\Plugin\Field\FieldType;

use Drupal\Core\Field\FieldItemBase;
use Drupal\Core\Field\FieldStorageDefinitionInterface;
use Drupal\Core\TypedData\DataDefinition;

/**
 * Plugin implementation of the 'rgb_color' field type.
 *
 * @FieldType(
 *   id = "rgb_color",
 *   label = @Translation("RGB Color"),
 *   description = @Translation("Stores an RGB color value."),
 *   default_widget = "rgb_color_default",
 *   default_formatter = "rgb_color_swatch",
 * )
 */
class RgbColorItem extends FieldItemBase {

  /**
   * {@inheritdoc}
   */
  public static function schema(FieldStorageDefinitionInterface $field_definition): array {
    return [
      'columns' => [
        'value' => [
          'type' => 'varchar',
          'length' => 7,
          'not null' => FALSE,
        ],
      ],
    ];
  }

  /**
   * {@inheritdoc}
   */
  public static function propertyDefinitions(FieldStorageDefinitionInterface $field_definition): array {
    $properties = [];
    $properties['value'] = DataDefinition::create('string')
      ->setLabel(t('Hex color'))
      ->setRequired(TRUE);
    return $properties;
  }

  /**
   * {@inheritdoc}
   */
  public function isEmpty(): bool {
    $value = $this->get('value')->getValue();
    return $value === NULL || $value === '';
  }

}
```

### Drupal 11+ (Attribute Syntax)

```php
<?php

namespace Drupal\my_module\Plugin\Field\FieldType;

use Drupal\Core\Field\Attribute\FieldType;
use Drupal\Core\Field\FieldItemBase;
use Drupal\Core\Field\FieldStorageDefinitionInterface;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use Drupal\Core\TypedData\DataDefinition;

#[FieldType(
  id: 'rgb_color',
  label: new TranslatableMarkup('RGB Color'),
  description: new TranslatableMarkup('Stores an RGB color value.'),
  default_widget: 'rgb_color_default',
  default_formatter: 'rgb_color_swatch',
)]
class RgbColorItem extends FieldItemBase {

  // Method implementations are identical to the annotation example above.

}
```

---

## Other Plugin Types (Quick Reference)

### Action Plugin

Used for bulk operations on entities (e.g., "Publish selected content").

- **Base class**: `Drupal\Core\Action\ActionBase` or `Drupal\Core\Action\ConfigurableActionBase`
- **Manager service**: `plugin.manager.action`
- **Annotation**: `@Action`
- **Attribute**: `#[Action]`
- **File location**: `src/Plugin/Action/`
- Full examples in `references/action-queue-plugins.md`

### QueueWorker Plugin

Processes items from a queue, often triggered by cron.

- **Base class**: `Drupal\Core\Queue\QueueWorkerBase`
- **Manager service**: `plugin.manager.queue_worker`
- **Annotation**: `@QueueWorker`
- **Attribute**: `#[QueueWorker]`
- **File location**: `src/Plugin/QueueWorker/`
- Full examples in `references/action-queue-plugins.md`

### Condition Plugin

Evaluates a condition (e.g., "current user has role X", "node is of type Y"). Used in block visibility, context system.

- **Base class**: `Drupal\Core\Condition\ConditionPluginBase`
- **Manager service**: `plugin.manager.condition`
- **Annotation**: `@Condition`
- **Attribute**: `#[Condition]`
- **File location**: `src/Plugin/Condition/`
- Full examples in `references/misc-plugins.md`

### Constraint (Validation) Plugin

Validates entity fields or typed data.

- **Base class**: `Drupal\Core\Validation\Plugin\Validation\Constraint\` (varies)
- **Manager service**: `validation.constraint`
- **Annotation**: `@Constraint`
- **File location**: `src/Plugin/Validation/Constraint/`
- Full examples in `references/misc-plugins.md`

### Views Plugins

Views has its own plugin system for fields, filters, sort handlers, arguments, relationships, and display modes.

- **File location**: `src/Plugin/views/field/`, `src/Plugin/views/filter/`, etc.
- Must be declared in `my_module.views.inc` via `hook_views_data()`
- Full examples in `references/misc-plugins.md`

---

## Creating a Custom Plugin Type

To define your own plugin type, you need:

1. **An annotation or attribute class** defining the metadata.
2. **A plugin interface** specifying what plugins must implement.
3. **A plugin manager** service that discovers and manages your plugins.

### Step 1: Define the Attribute (Drupal 11+)

```php
<?php

namespace Drupal\my_module\Attribute;

use Drupal\Component\Plugin\Attribute\Plugin;
use Drupal\Core\StringTranslation\TranslatableMarkup;

#[\Attribute(\Attribute::TARGET_CLASS)]
class Sandwich extends Plugin {

  public function __construct(
    public readonly string $id,
    public readonly TranslatableMarkup $label,
    public readonly ?TranslatableMarkup $description = NULL,
  ) {
    parent::__construct($id);
  }

}
```

### Step 2: Define the Plugin Interface

```php
<?php

namespace Drupal\my_module\Plugin;

use Drupal\Component\Plugin\PluginInspectionInterface;

interface SandwichInterface extends PluginInspectionInterface {

  public function calories(): int;

  public function ingredients(): array;

}
```

### Step 3: Create the Plugin Manager

```php
<?php

namespace Drupal\my_module\Plugin;

use Drupal\Core\Cache\CacheBackendInterface;
use Drupal\Core\Extension\ModuleHandlerInterface;
use Drupal\Core\Plugin\DefaultPluginManager;
use Drupal\my_module\Attribute\Sandwich;

class SandwichPluginManager extends DefaultPluginManager {

  public function __construct(
    \Traversable $namespaces,
    CacheBackendInterface $cache_backend,
    ModuleHandlerInterface $module_handler,
  ) {
    parent::__construct(
      'Plugin/Sandwich',
      $namespaces,
      $module_handler,
      SandwichInterface::class,
      Sandwich::class,
    );
    $this->setCacheBackend($cache_backend, 'sandwich_plugins');
    $this->alterInfo('sandwich_info');
  }

}
```

### Step 4: Register the Manager as a Service

In `my_module.services.yml`:

```yaml
services:
  plugin.manager.sandwich:
    class: Drupal\my_module\Plugin\SandwichPluginManager
    parent: default_plugin_manager
```

---

## Scaffolding with Drush

Use `ddev drush generate` (Drush 12+) to scaffold plugin code quickly:

```bash
# Block plugin
ddev drush gen plugin:block

# Field type
ddev drush gen plugin:field:type

# Field widget
ddev drush gen plugin:field:widget

# Field formatter
ddev drush gen plugin:field:formatter

# Action plugin
ddev drush gen plugin:action

# Queue worker
ddev drush gen plugin:queue-worker

# Condition plugin
ddev drush gen plugin:condition

# Views field plugin
ddev drush gen plugin:views:field

# Custom plugin type manager
ddev drush gen plugin-manager
```

Each generator will prompt for module name, plugin ID, label, and class name, then generate properly structured files.

After generating, always:
1. Clear the cache: `ddev drush cr`
2. For blocks, place the block via Structure > Block layout or config.
3. For field types, create a field of that type on an entity bundle.

---

## Key Principles

- **Always clear cache** after adding or modifying a plugin class (`ddev drush cr`).
- **Plugin IDs must be unique** across all modules. Prefix with your module name.
- **Use dependency injection** -- override `create()` from `ContainerFactoryPluginInterface` to inject services into plugins.
- **Cache metadata** should be added to any render array returned by a plugin's `build()` method.
- **Annotations still work** in Drupal 11 but attributes are the recommended approach. New projects should use attributes.
- **Plugin derivatives** let you generate multiple plugin instances from a single class (see `references/block-plugins.md`).

---

## Related Skills

- **drupal-services** -- How to define and inject services into plugins via `ContainerFactoryPluginInterface`.
- **drupal-forms** -- Form API details for plugin configuration forms.
- **drupal-cache** -- Cache contexts, tags, and max-age for plugin render arrays.
- **drupal-queue** -- Queue system details for QueueWorker plugins.
- **drush-generate** -- Full reference for Drush code generation commands.
