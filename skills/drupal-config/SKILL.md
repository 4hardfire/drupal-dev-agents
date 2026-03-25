---
name: drupal-config
description: Use when working with Drupal configuration system, config schema, config entities, simple configuration, configuration overrides, or config import/export.
version: 1.0.0
---

# Drupal Configuration System

## Overview

Drupal's configuration system manages all site settings that are not content.
Configuration is stored as YAML in the database (active storage) and can be
exported to the filesystem (sync storage) for version control and deployment.

There are two types of configuration:

| Aspect | Simple Configuration | Config Entity |
|---|---|---|
| Storage key | Single name (e.g., `my_module.settings`) | Prefixed with wildcard (e.g., `my_module.task_type.*`) |
| PHP class | None — accessed via `ConfigFactory` | Extends `ConfigEntityBase` |
| Multiple instances | No — one config object per name | Yes — many entities of the same type |
| Admin UI | Typically a `ConfigFormBase` settings form | Entity list builder + entity forms |
| Schema type | `config_object` | `config_entity` |
| Use when | Module settings, feature flags, API keys | Reusable bundles, styles, views, workflows |

**Examples of simple config:** `system.site`, `my_module.settings`, `system.performance`.
**Examples of config entities:** View, ImageStyle, Role, Vocabulary, NodeType.

---

## Simple Configuration — ConfigFactory

The `config.factory` service is the primary way to read and write simple
configuration.

### Reading Configuration (Immutable)

```php
// In a service or controller with dependency injection.
$config = $this->configFactory->get('my_module.settings');
$value = $config->get('items_per_page');
$all = $config->getRawData();

// Static access (avoid when possible).
$config = \Drupal::config('my_module.settings');
$value = $config->get('items_per_page');
```

`get()` returns an `ImmutableConfig` object. You cannot call `set()` or
`save()` on it — this prevents accidental writes.

### Writing Configuration (Editable)

```php
// In a service or controller.
$config = $this->configFactory->getEditable('my_module.settings');
$config->set('items_per_page', 25);
$config->set('display_mode', 'grid');
$config->save();

// Clear a single key.
$config->clear('deprecated_key')->save();

// Delete an entire config object.
$config->delete();
```

### Nested Values

```php
// Set a nested value.
$config->set('api.endpoint', 'https://example.com/v2');
$config->set('api.timeout', 30);

// Read a nested value.
$endpoint = $config->get('api.endpoint');

// Read the entire nested mapping.
$api = $config->get('api');
// Returns: ['endpoint' => 'https://example.com/v2', 'timeout' => 30]
```

### Dependency Injection

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Service;

use Drupal\Core\Config\ConfigFactoryInterface;

class MyService {

  public function __construct(
    protected readonly ConfigFactoryInterface $configFactory,
  ) {}

  public function getItemsPerPage(): int {
    return (int) $this->configFactory
      ->get('my_module.settings')
      ->get('items_per_page');
  }

}
```

---

## Config Schema

Every configuration object **must** have a schema definition. Without it,
config validation, translation, and import/export will fail.

Schema files live in `config/schema/MODULE_NAME.schema.yml`.

### Simple Config Schema

```yaml
my_module.settings:
  type: config_object
  label: 'My Module settings'
  mapping:
    items_per_page:
      type: integer
      label: 'Items per page'
    display_mode:
      type: string
      label: 'Display mode'
    enable_feature:
      type: boolean
      label: 'Enable special feature'
    api:
      type: mapping
      label: 'API settings'
      mapping:
        endpoint:
          type: uri
          label: 'API endpoint'
        timeout:
          type: integer
          label: 'Request timeout'
```

### Core Schema Data Types

| Type | Description |
|---|---|
| `string` | Plain string |
| `label` | Translatable human-readable label |
| `text` | Translatable long text |
| `integer` | Integer value |
| `float` | Floating-point number |
| `boolean` | Boolean true/false |
| `uri` | URI string |
| `path` | Drupal internal path |
| `email` | Email address |
| `plural_label` | Translatable plural label |
| `date_format` | PHP date format string |
| `mapping` | Key-value structure (like an associative array) |
| `sequence` | Ordered list (like an indexed array) |
| `config_object` | Top-level simple configuration |
| `config_entity` | Top-level config entity |
| `config_dependencies` | Standard dependency structure |

> See `references/config-schema.md` for a deep dive on all schema types,
> sequences, dynamic references, and inheritance.

---

## Default Configuration — config/install

Files placed in `config/install/` are imported when the module is first
installed. The filename must match the config name exactly.

### File: `config/install/my_module.settings.yml`

```yaml
items_per_page: 10
display_mode: list
enable_feature: false
api:
  endpoint: 'https://api.example.com/v1'
  timeout: 30
```

### Config Dependencies

If your default config depends on another module, declare it:

```yaml
# config/install/my_module.settings.yml
items_per_page: 10
display_mode: list
enable_feature: false
```

Dependencies can be declared in the config YAML itself:

```yaml
dependencies:
  module:
    - node
  config:
    - system.site
```

Or enforced dependencies (the config is deleted when the dependency is removed):

```yaml
dependencies:
  enforced:
    module:
      - my_other_module
```

---

## Optional Configuration — config/optional

Files in `config/optional/` are imported **only** when all their dependencies
are met at install time. If a dependency is missing, the config is silently
skipped.

Use `config/optional/` for:

- Views that depend on optional modules (e.g., a view requiring `media` module).
- Config entities that reference other config entities which may not exist.
- Third-party integration settings.

```
my_module/
  config/
    install/
      my_module.settings.yml           # Always installed.
    optional/
      views.view.my_module_listing.yml  # Only if Views module is enabled.
      core.entity_view_display.node.article.my_module_teaser.yml  # Only if article bundle exists.
```

Optional config files must include proper `dependencies` so Drupal knows what
conditions must be met:

```yaml
# config/optional/views.view.my_module_listing.yml
dependencies:
  module:
    - my_module
    - node
    - views
```

---

## Configuration Overrides

Overrides let you change configuration values without modifying the stored
config. They apply at runtime and do not affect config export.

### settings.php Overrides

The most common approach for environment-specific values:

```php
// In settings.php or settings.local.php.
$config['system.site']['name'] = 'My Site (Local Dev)';
$config['system.performance']['css']['preprocess'] = FALSE;
$config['system.performance']['js']['preprocess'] = FALSE;
$config['my_module.settings']['api']['endpoint'] = 'https://staging-api.example.com/v1';
```

### Config Override Module

For programmatic overrides, implement `ConfigFactoryOverrideInterface`:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module;

use Drupal\Core\Cache\CacheableMetadata;
use Drupal\Core\Config\ConfigFactoryOverrideInterface;
use Drupal\Core\Config\StorageInterface;

class MyConfigOverride implements ConfigFactoryOverrideInterface {

  /**
   * {@inheritdoc}
   */
  public function loadOverrides($names): array {
    $overrides = [];
    if (in_array('system.site', $names)) {
      $overrides['system.site']['name'] = 'Overridden Site Name';
    }
    return $overrides;
  }

  /**
   * {@inheritdoc}
   */
  public function getCacheSuffix(): string {
    return 'my_module_override';
  }

  /**
   * {@inheritdoc}
   */
  public function getCacheableMetadata($name): CacheableMetadata {
    return new CacheableMetadata();
  }

  /**
   * {@inheritdoc}
   */
  public function createConfigObject($name, $collection = StorageInterface::DEFAULT_COLLECTION) {
    return NULL;
  }

}
```

Register it as a tagged service:

```yaml
# my_module.services.yml
services:
  my_module.config_override:
    class: Drupal\my_module\MyConfigOverride
    tags:
      - { name: config.factory.override }
```

**Important:** Overridden values are returned by `$config->get()` but are not
saved to storage. To get the original stored value, use
`$config->getOriginal('key', FALSE)`.

---

## Config Import / Export

### Drush Commands

```bash
# Export active config to the sync directory (config/sync by default).
ddev drush config:export
# Short alias:
ddev drush cex

# Import config from the sync directory into active storage.
ddev drush config:import
# Short alias:
ddev drush cim

# Show differences between active and sync storage.
ddev drush config:status
# Short alias:
ddev drush cst

# Export a single config item to stdout.
ddev drush config:get system.site

# Set a single config value.
ddev drush config:set system.site name "New Site Name"

# Partial import — import only specific files.
ddev drush config:import --partial --source=/path/to/config/files
```

### Sync Directory

The sync directory is configured in `settings.php`:

```php
$settings['config_sync_directory'] = '../config/sync';
```

### Typical Workflow

1. Make changes on your local site (admin UI, drush, code).
2. `ddev drush cex` to export changes to files.
3. Commit the config YAML files to version control.
4. On deploy, `ddev drush cim` to import config on the target environment.
5. `ddev drush cr` to rebuild caches.

---

## Scaffolding with Drush

```bash
# Generate a module services file.
ddev drush gen yml:module:services

# Generate a config form (creates form class + config schema + default config + route).
ddev drush gen form:config

# Generate a config entity type (creates entity class + schema + forms + list builder).
ddev drush gen entity:config
```

---

## Key Points to Remember

1. **Always provide a config schema.** Without it, config validation fails,
   config translation breaks, and `drush config:import` may reject your config.

2. **Use `getEditable()` only when writing.** For read-only access, use `get()`
   which returns an `ImmutableConfig` object. This prevents accidental
   modifications.

3. **Default config is only imported once.** Changing files in `config/install/`
   after the module is installed has no effect. Use `hook_update_N()` to modify
   config on existing sites.

4. **Config overrides do not persist.** They are applied at runtime and are not
   exported. Use `$config->getOriginal()` to read the stored value.

5. **Config entity vs simple config:** If you need multiple instances of the
   same structure (e.g., multiple image styles, multiple task types), use a
   config entity. If you need a single settings object, use simple config.

6. **Enforced dependencies** cause config to be deleted when the dependency
   module is uninstalled. Use them for config that is meaningless without its
   provider.

---

## Related Skills

- **drupal-entities** — Config entities, content entities, entity handlers.
- **drupal-forms** — ConfigFormBase for settings forms, form validation.
- **drupal-services** — Service definitions, dependency injection.
- **drupal-hooks** — hook_update_N() for config updates on existing sites.
- **drush-generate** — Full reference for all Drush code generators.
