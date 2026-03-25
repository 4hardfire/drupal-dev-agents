# Config Schema — Deep Dive

## Overview

Config schema defines the structure and data types for every configuration
object in Drupal. It is used for:

- **Validation** — `drush config:import` and config forms validate against schema.
- **Translation** — Only schema-typed values marked as translatable can be translated.
- **Type casting** — Ensures values are cast correctly when read from YAML.

Schema files live in `config/schema/MODULE_NAME.schema.yml`.

---

## Top-Level Types

### config_object — Simple Configuration

```yaml
my_module.settings:
  type: config_object
  label: 'My Module settings'
  mapping:
    enabled:
      type: boolean
      label: 'Enabled'
    name:
      type: string
      label: 'Machine name'
```

### config_entity — Configuration Entity

```yaml
my_module.task_type.*:
  type: config_entity
  label: 'Task type'
  mapping:
    id:
      type: string
      label: 'Machine name'
    label:
      type: label
      label: 'Label'
    description:
      type: text
      label: 'Description'
    uuid:
      type: string
      label: 'UUID'
```

The `*` wildcard matches any config entity instance (e.g.,
`my_module.task_type.bug`, `my_module.task_type.feature`).

---

## Scalar Types

### string

Plain, non-translatable string.

```yaml
api_key:
  type: string
  label: 'API key'
```

### label

Translatable human-readable short text. Used for names and titles.

```yaml
site_name:
  type: label
  label: 'Site name'
```

### text

Translatable long text. Used for descriptions and body content.

```yaml
description:
  type: text
  label: 'Description'
```

### plural_label

Translatable label with plural form support.

```yaml
label_plural:
  type: plural_label
  label: 'Plural label'
```

### integer

```yaml
max_items:
  type: integer
  label: 'Maximum items'
```

### float

```yaml
ratio:
  type: float
  label: 'Aspect ratio'
```

### boolean

```yaml
enabled:
  type: boolean
  label: 'Enabled'
```

### uri

Any valid URI string.

```yaml
endpoint:
  type: uri
  label: 'API endpoint'
```

### path

A Drupal internal path (e.g., `/admin/config`).

```yaml
redirect_path:
  type: path
  label: 'Redirect path'
```

### email

```yaml
admin_email:
  type: email
  label: 'Admin email'
```

### date_format

A PHP date format pattern string.

```yaml
format:
  type: date_format
  label: 'Date format'
```

---

## Compound Types

### mapping

An associative array with named keys. Each key has its own type definition.

```yaml
api:
  type: mapping
  label: 'API settings'
  mapping:
    endpoint:
      type: uri
      label: 'Endpoint URL'
    timeout:
      type: integer
      label: 'Timeout in seconds'
    auth:
      type: mapping
      label: 'Authentication'
      mapping:
        username:
          type: string
          label: 'Username'
        token:
          type: string
          label: 'API token'
```

### sequence

An ordered list where all items share the same type.

```yaml
# Simple sequence of strings.
allowed_domains:
  type: sequence
  label: 'Allowed domains'
  sequence:
    type: string
    label: 'Domain'

# Sequence of mappings.
endpoints:
  type: sequence
  label: 'API endpoints'
  sequence:
    type: mapping
    label: 'Endpoint'
    mapping:
      url:
        type: uri
        label: 'URL'
      weight:
        type: integer
        label: 'Weight'
      enabled:
        type: boolean
        label: 'Enabled'
```

---

## Schema Inheritance

You can define reusable base types and extend them. Drupal core defines many
base types in `core/config/schema/core.data_types.schema.yml`.

### Defining a Reusable Type

```yaml
my_module.api_connection:
  type: mapping
  label: 'API connection'
  mapping:
    endpoint:
      type: uri
      label: 'Endpoint'
    timeout:
      type: integer
      label: 'Timeout'
    api_key:
      type: string
      label: 'API key'
```

### Using It in Another Schema

```yaml
my_module.settings:
  type: config_object
  label: 'My Module settings'
  mapping:
    primary_api:
      type: my_module.api_connection
      label: 'Primary API connection'
    fallback_api:
      type: my_module.api_connection
      label: 'Fallback API connection'
```

---

## Dynamic Type References

Dynamic type references use `[%key]` or `[%parent.%key]` to resolve the schema
type based on the actual config data. This is how Drupal handles plugin-based
configuration.

### Core Example: Field Storage

```yaml
# From core.data_types.schema.yml
field.storage.config.field_type:
  type: string

field.storage.*.*:
  type: config_entity
  mapping:
    field_name:
      type: string
    entity_type:
      type: string
    type:
      type: string
    settings:
      type: field.storage_settings.[%parent.type]
```

When Drupal encounters a field storage config for a `text` field, it resolves
`field.storage_settings.[%parent.type]` to `field.storage_settings.text` and
looks up that schema definition.

### Using Dynamic References in Custom Modules

```yaml
# Define a per-plugin-type schema pattern.
my_module.processor_settings.*:
  type: mapping
  label: 'Processor settings'

# Specific plugin implementations provide their own schema.
my_module.processor_settings.html_filter:
  type: mapping
  label: 'HTML filter processor'
  mapping:
    allowed_tags:
      type: string
      label: 'Allowed HTML tags'
    strip_comments:
      type: boolean
      label: 'Strip comments'

# Reference it dynamically in the parent config.
my_module.settings:
  type: config_object
  label: 'My Module settings'
  mapping:
    processor:
      type: string
      label: 'Active processor'
    processor_settings:
      type: my_module.processor_settings.[%parent.processor]
      label: 'Processor settings'
```

---

## third_party_settings Schema

Modules can attach settings to config entities they do not own via
`third_party_settings`. The schema follows a standard pattern:

```yaml
# Schema for third-party settings on node types.
node.type.*.third_party.my_module:
  type: mapping
  label: 'My Module settings'
  mapping:
    custom_flag:
      type: boolean
      label: 'Custom flag'
    custom_label:
      type: label
      label: 'Custom label'
```

In code, these are accessed via the config entity:

```php
$node_type = NodeType::load('article');
$flag = $node_type->getThirdPartySetting('my_module', 'custom_flag', FALSE);
$node_type->setThirdPartySetting('my_module', 'custom_flag', TRUE);
$node_type->save();
```

---

## Core Schema Examples

### system.site

```yaml
# From core/config/schema/core.data_types.schema.yml
system.site:
  type: config_object
  label: 'Site information'
  mapping:
    uuid:
      type: string
      label: 'Site UUID'
    name:
      type: label
      label: 'Site name'
    mail:
      type: email
      label: 'Email'
    slogan:
      type: label
      label: 'Slogan'
    page:
      type: mapping
      label: 'Pages'
      mapping:
        403:
          type: path
          label: '403 page'
        404:
          type: path
          label: '404 page'
        front:
          type: path
          label: 'Front page'
```

### views.view.*

Views demonstrate complex schema with sequences, nested mappings, and dynamic
references:

```yaml
views.view.*:
  type: config_entity
  label: 'View'
  mapping:
    id:
      type: string
    label:
      type: label
    description:
      type: text
    tag:
      type: string
    base_table:
      type: string
    display:
      type: sequence
      sequence:
        type: views.display.[%key]
```

### block.block.*

```yaml
block.block.*:
  type: config_entity
  label: 'Block'
  mapping:
    id:
      type: string
    theme:
      type: string
    region:
      type: string
    weight:
      type: integer
    plugin:
      type: string
    settings:
      type: block.settings.[%parent.plugin]
```

---

## Validating Schema

### Drush

```bash
# Check if all active config has valid schema.
ddev drush config:inspect

# Validate a specific config name.
ddev drush config:inspect my_module.settings
```

### PHPUnit

Drupal provides `SchemaCheckTestTrait` for testing config schema:

```php
use Drupal\Tests\SchemaCheckTestTrait;
use Drupal\Tests\BrowserTestBase;

class MyModuleConfigTest extends BrowserTestBase {

  use SchemaCheckTestTrait;

  protected static $modules = ['my_module'];

  public function testConfigSchema(): void {
    $config = $this->config('my_module.settings');
    $this->assertConfigSchema(
      \Drupal::service('config.typed'),
      'my_module.settings',
      $config->get()
    );
  }

}
```

---

## Common Pitfalls

1. **Missing schema causes test failures.** Drupal's test framework validates
   all config against schema by default. Any missing schema will cause
   `ConfigSchemaChecker` errors.

2. **Sequence vs mapping confusion.** Use `sequence` for ordered lists of the
   same type. Use `mapping` for named keys with different types.

3. **Translatable types matter.** Use `label` and `text` for user-facing
   strings. Use `string` for machine values that should not be translated.

4. **Wildcard patterns must match config names.** The schema key
   `my_module.task_type.*` matches `my_module.task_type.bug` but not
   `my_module.task_type.bug.settings`.

5. **Schema changes need cache rebuild.** After modifying schema files, run
   `ddev drush cr` to pick up changes.
