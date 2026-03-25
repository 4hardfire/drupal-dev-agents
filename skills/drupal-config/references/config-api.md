# Configuration API Deep Dive

## ConfigFactory Service

The primary way to interact with configuration in Drupal.

### Reading Configuration (Immutable)

```php
// In a service or controller with DI:
$config = $this->configFactory->get('mymodule.settings');
$value = $config->get('api_key');
$nested = $config->get('connection.host'); // Dot notation for nested

// All data as array:
$all = $config->getRawData();
```

### Writing Configuration (Editable)

```php
// Must explicitly request editable config:
$config = $this->configFactory->getEditable('mymodule.settings');
$config->set('api_key', 'new-value');
$config->set('connection.host', 'localhost');
$config->save();

// Clear a value:
$config->clear('deprecated_setting')->save();

// Delete entire config object:
$config->delete();
```

### ImmutableConfig vs Config

| Method | ImmutableConfig | Config (Editable) |
|--------|----------------|-------------------|
| `get()` | Yes | Yes |
| `getRawData()` | Yes | Yes |
| `set()` | No (throws) | Yes |
| `clear()` | No (throws) | Yes |
| `save()` | No (throws) | Yes |
| `delete()` | No (throws) | Yes |

Always use `get()` unless you need to write. This prevents accidental modifications.

## Config Events

```php
use Drupal\Core\Config\ConfigEvents;

// Available events:
ConfigEvents::SAVE        // After a config object is saved
ConfigEvents::DELETE      // After a config object is deleted
ConfigEvents::RENAME      // After a config object is renamed
ConfigEvents::IMPORT_VALIDATE  // Before config import validation
ConfigEvents::IMPORT      // During config import
ConfigEvents::COLLECTION_INFO  // When collecting config collection info
ConfigEvents::IMPORT_MISSING_CONTENT  // When imported config references missing content
```

### Subscribing to Config Events

```php
<?php

namespace Drupal\mymodule\EventSubscriber;

use Drupal\Core\Config\ConfigCrudEvent;
use Drupal\Core\Config\ConfigEvents;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

class ConfigEventSubscriber implements EventSubscriberInterface {

  public static function getSubscribedEvents(): array {
    return [
      ConfigEvents::SAVE => ['onConfigSave', 0],
      ConfigEvents::DELETE => ['onConfigDelete', 0],
    ];
  }

  public function onConfigSave(ConfigCrudEvent $event): void {
    $config = $event->getConfig();
    if ($config->getName() === 'system.site') {
      // React to site config changes.
      $new_name = $config->get('name');
    }
  }

  public function onConfigDelete(ConfigCrudEvent $event): void {
    // Handle config deletion.
  }

}
```

## Config Dependencies

### Enforced Dependencies

In `config/install/*.yml`:
```yaml
dependencies:
  enforced:
    module:
      - mymodule
```

This ensures the config is deleted when the module is uninstalled.

### Calculated Dependencies

Config entities can declare dependencies via `calculateDependencies()`:

```php
public function calculateDependencies() {
  parent::calculateDependencies();
  // Add dependency on a module.
  $this->addDependency('module', 'some_module');
  // Add dependency on a config entity.
  $this->addDependency('config', 'node.type.article');
  // Add dependency on content.
  $this->addDependency('content', 'node:article:some-uuid');
  return $this;
}
```

## Simple Config vs Config Entity Decision Guide

| Criteria | Simple Config | Config Entity |
|----------|--------------|---------------|
| Number of instances | One (singleton) | Multiple (CRUD) |
| Admin UI needed | Settings form only | List + add/edit/delete |
| Exportable | Yes | Yes |
| Has an ID/label | No | Yes |
| Fieldable | No | No (but can have properties) |
| Plugins can reference | No | Yes (e.g., image styles) |
| Example | Module settings | Image styles, views, roles |

**Use simple config** for: module settings, API keys, feature toggles, site-wide preferences.

**Use config entity** for: things users create multiple of — image styles, text formats, search pages, workflows.

## Config Translation

For translatable config, add `translatable: true` to schema:

```yaml
# config/schema/mymodule.schema.yml
mymodule.settings:
  type: config_object
  label: 'My module settings'
  mapping:
    welcome_message:
      type: label
      label: 'Welcome message'
      translatable: true
```

Types that are translatable by default: `label`, `text`, `plural_label`.

## Config Split

Common pattern for environment-specific config using the `config_split` module:

```yaml
# config/sync/config_split.config_split.dev.yml
id: dev
label: Development
status: true
folder: ../config/splits/dev
blacklist:
  - system.performance
  - system.logging
graylist: {  }
```

This allows different config values per environment (dev, staging, prod).

## Config Override System

### Settings.php Overrides (Highest Priority)

```php
// In settings.php or settings.local.php:
$config['system.performance']['css']['preprocess'] = FALSE;
$config['system.performance']['js']['preprocess'] = FALSE;
$config['system.site']['name'] = 'Local Dev Site';
```

### Module-Based Config Overrides

```php
<?php

namespace Drupal\mymodule;

use Drupal\Core\Cache\CacheableMetadata;
use Drupal\Core\Config\ConfigFactoryOverrideInterface;
use Drupal\Core\Config\StorageInterface;

class ConfigOverrides implements ConfigFactoryOverrideInterface {

  public function loadOverrides($names): array {
    $overrides = [];
    if (in_array('system.site', $names)) {
      $overrides['system.site']['name'] = 'Overridden Site Name';
    }
    return $overrides;
  }

  public function getCacheSuffix(): string {
    return 'mymodule_override';
  }

  public function getCacheableMetadata($name): CacheableMetadata {
    return new CacheableMetadata();
  }

  public function createConfigObject($name, $collection = StorageInterface::DEFAULT_COLLECTION) {
    return NULL;
  }

}
```

Register as a service with `config.factory.override` tag:
```yaml
services:
  mymodule.config_overrides:
    class: Drupal\mymodule\ConfigOverrides
    tags:
      - { name: config.factory.override, priority: 5 }
```

### Override Priority (lowest to highest)
1. Stored values (database/files)
2. Module overrides (`config.factory.override` tag, ordered by priority)
3. Settings.php `$config` array
4. Language overrides (config translation)

> **Note**: `getEditable()` ignores overrides and returns the stored value. Only `get()` applies overrides.

## Config Import/Export with Drush

```bash
# Export all config to sync directory
ddev drush config:export -y

# Import config from sync directory
ddev drush config:import -y

# Show differences between active and sync
ddev drush config:status

# Export a single config item
ddev drush config:get system.site

# Set a single config value
ddev drush config:set system.site name "New Name" -y

# Partial import (single file)
ddev drush config:import --partial --source=/path/to/dir -y
```
