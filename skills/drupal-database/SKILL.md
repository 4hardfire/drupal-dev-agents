---
name: drupal-database
description: Use when working with Drupal database operations, schema definitions, update hooks (hook_update_N), post_update hooks, or the Database API (select, insert, update, delete queries).
version: 1.0.0
---

# Drupal Database API

## Overview

Drupal's Database API is a PDO-based abstraction layer that provides a structured, database-agnostic way to interact with the database. It supports MySQL/MariaDB, PostgreSQL, and SQLite through a unified interface. The API offers both a dynamic query builder (recommended) and a static query method for complex SQL.

### Core Concepts

- **Connection**: Obtained via `\Drupal::database()` or by injecting the `database` service. Returns a `\Drupal\Core\Database\Connection` object.
- **Dynamic Queries**: Built using object-oriented methods (`select()`, `insert()`, `update()`, `delete()`, `merge()`). Database-agnostic and the preferred approach.
- **Static Queries**: Raw SQL strings passed to `query()`. Use only when the query builder cannot express the query.
- **Schema API**: Defines database table structures in `hook_schema()` and modifies them in update hooks.
- **Transactions**: Supported via `$connection->startTransaction()`.

### When to Use Database API vs Entity API

| Use Database API | Use Entity API |
|------------------|----------------|
| Custom tables not backed by entities | Loading, saving, deleting entities |
| Aggregation and reporting queries | CRUD on nodes, users, taxonomy terms |
| Performance-critical bulk operations | Anything that needs hooks, access control, validation |
| Legacy tables or external data sources | Field data on fieldable entities |
| Logging, statistics, session tables | Content and configuration entities |

**Rule of thumb**: If the data is an entity or entity field, use the Entity API. If you manage your own custom table, use the Database API.

---

## Schema Definition

Define custom tables in `hook_schema()` inside your module's `.install` file. Drupal creates these tables when the module is installed and removes them on uninstall.

### Common Column Types

| Type | Description | Required Properties |
|------|-------------|---------------------|
| `serial` | Auto-incrementing integer | `not null`, `unsigned` |
| `int` | Integer | `size` (tiny/small/medium/normal/big) |
| `varchar` | Variable-length string | `length` |
| `varchar_ascii` | ASCII-only variable-length string | `length` |
| `text` | Unlimited-length text | `size` (tiny/small/medium/normal/big) |
| `blob` | Binary data | `size` (tiny/small/medium/normal/big) |
| `float` | Floating-point number | `size` (tiny/small/medium/normal/big) |
| `numeric` | Exact decimal number | `precision`, `scale` |

### Full Schema Example

```php
<?php

/**
 * @file
 * Install, update, and uninstall functions for my_module.
 */

/**
 * Implements hook_schema().
 */
function my_module_schema(): array {
  $schema['my_module_log'] = [
    'description' => 'Stores log entries for my module.',
    'fields' => [
      'lid' => [
        'type' => 'serial',
        'unsigned' => TRUE,
        'not null' => TRUE,
        'description' => 'Primary key: log entry ID.',
      ],
      'uid' => [
        'type' => 'int',
        'unsigned' => TRUE,
        'not null' => TRUE,
        'default' => 0,
        'description' => 'The {users}.uid of the user who triggered the event.',
      ],
      'type' => [
        'type' => 'varchar_ascii',
        'length' => 64,
        'not null' => TRUE,
        'default' => '',
        'description' => 'Type of log entry.',
      ],
      'message' => [
        'type' => 'text',
        'size' => 'big',
        'not null' => TRUE,
        'description' => 'The log message.',
      ],
      'severity' => [
        'type' => 'int',
        'size' => 'tiny',
        'unsigned' => TRUE,
        'not null' => TRUE,
        'default' => 0,
        'description' => 'Severity level (RFC 5424).',
      ],
      'amount' => [
        'type' => 'numeric',
        'precision' => 10,
        'scale' => 2,
        'not null' => FALSE,
        'description' => 'An optional numeric amount.',
      ],
      'created' => [
        'type' => 'int',
        'not null' => TRUE,
        'default' => 0,
        'description' => 'Unix timestamp when the entry was created.',
      ],
    ],
    'primary key' => ['lid'],
    'indexes' => [
      'type' => ['type'],
      'uid' => ['uid'],
      'created' => ['created'],
      'type_severity' => ['type', 'severity'],
    ],
    'unique keys' => [],
    'foreign keys' => [
      'log_author' => [
        'table' => 'users',
        'columns' => ['uid' => 'uid'],
      ],
    ],
  ];

  return $schema;
}
```

---

## Update Hooks

Update hooks run once during `drush updb` (database updates) and allow you to make schema changes or migrate data.

### hook_update_N() Pattern

The `N` in `hook_update_N()` follows the convention: **XYZZ**, where X is the Drupal core major version compatibility, Y is for your module's major version, and ZZ is a sequential counter starting at 01. For contributed and custom modules targeting Drupal 10+, start at **10001**.

```php
/**
 * Add the 'status' column to {my_module_log}.
 */
function my_module_update_10001(): string {
  $schema = \Drupal::database()->schema();

  $spec = [
    'type' => 'int',
    'size' => 'tiny',
    'unsigned' => TRUE,
    'not null' => TRUE,
    'default' => 1,
    'description' => 'Boolean indicating whether the entry is active.',
  ];
  $schema->addField('my_module_log', 'status', $spec);

  return 'Added status column to my_module_log table.';
}

/**
 * Add an index on the 'status' column.
 */
function my_module_update_10002(): string {
  $schema = \Drupal::database()->schema();
  $schema->addIndex('my_module_log', 'status', ['status'], [
    'fields' => [
      'status' => [
        'type' => 'int',
        'size' => 'tiny',
        'unsigned' => TRUE,
        'not null' => TRUE,
        'default' => 1,
      ],
    ],
  ]);

  return 'Added index on status column.';
}
```

### Using Batch in Update Hooks

For updates that process large amounts of data, use the `$sandbox` parameter to run in batches:

```php
/**
 * Populate the 'status' column for existing entries.
 */
function my_module_update_10003(array &$sandbox): string {
  $database = \Drupal::database();

  if (!isset($sandbox['progress'])) {
    $sandbox['progress'] = 0;
    $sandbox['max'] = $database->select('my_module_log', 'l')
      ->countQuery()
      ->execute()
      ->fetchField();
  }

  // Process 100 rows per batch.
  $lids = $database->select('my_module_log', 'l')
    ->fields('l', ['lid'])
    ->range($sandbox['progress'], 100)
    ->orderBy('lid')
    ->execute()
    ->fetchCol();

  foreach ($lids as $lid) {
    $database->update('my_module_log')
      ->fields(['status' => 1])
      ->condition('lid', $lid)
      ->execute();
    $sandbox['progress']++;
  }

  $sandbox['#finished'] = empty($sandbox['max']) ? 1 : ($sandbox['progress'] / $sandbox['max']);

  return 'Updated status for ' . $sandbox['progress'] . ' log entries.';
}
```

---

## Post-Update Hooks

`hook_post_update_NAME()` runs after all `hook_update_N()` hooks. Use it for data migrations and entity resaves that depend on schema being up to date. The NAME part is a descriptive string (not a number).

Post-update hooks live in `my_module.post_update.php`.

```php
<?php

/**
 * @file
 * Post-update functions for my_module.
 */

/**
 * Resave all log entries to populate computed fields.
 */
function my_module_post_update_resave_log_entries(array &$sandbox): string {
  $database = \Drupal::database();

  if (!isset($sandbox['progress'])) {
    $sandbox['progress'] = 0;
    $sandbox['max'] = $database->select('my_module_log', 'l')
      ->countQuery()
      ->execute()
      ->fetchField();
  }

  $entries = $database->select('my_module_log', 'l')
    ->fields('l')
    ->range($sandbox['progress'], 50)
    ->orderBy('lid')
    ->execute()
    ->fetchAll();

  foreach ($entries as $entry) {
    // Perform data migration logic here.
    $sandbox['progress']++;
  }

  $sandbox['#finished'] = empty($sandbox['max']) ? 1 : ($sandbox['progress'] / $sandbox['max']);

  return 'Processed ' . $sandbox['progress'] . ' of ' . $sandbox['max'] . ' entries.';
}
```

---

## Basic Queries

### Select

```php
$database = \Drupal::database();

// Simple select with conditions.
$result = $database->select('my_module_log', 'l')
  ->fields('l', ['lid', 'type', 'message', 'created'])
  ->condition('type', 'error')
  ->condition('severity', 3, '>=')
  ->orderBy('created', 'DESC')
  ->range(0, 10)
  ->execute();

foreach ($result as $record) {
  // $record is a \stdClass object.
  $message = $record->message;
}

// Count query.
$count = $database->select('my_module_log', 'l')
  ->condition('type', 'error')
  ->countQuery()
  ->execute()
  ->fetchField();
```

### Insert

```php
$database = \Drupal::database();

// Single insert.
$lid = $database->insert('my_module_log')
  ->fields([
    'uid' => 1,
    'type' => 'notice',
    'message' => 'Something happened.',
    'severity' => 5,
    'created' => \Drupal::time()->getRequestTime(),
  ])
  ->execute();
// $lid is the new auto-increment ID.
```

### Update

```php
$database = \Drupal::database();

$rows_affected = $database->update('my_module_log')
  ->fields([
    'severity' => 1,
  ])
  ->condition('type', 'debug')
  ->condition('created', strtotime('-30 days'), '<')
  ->execute();
```

### Delete

```php
$database = \Drupal::database();

$rows_deleted = $database->delete('my_module_log')
  ->condition('created', strtotime('-90 days'), '<')
  ->execute();
```

### Merge (Upsert)

```php
$database = \Drupal::database();

// Insert if the key doesn't exist; update if it does.
$database->merge('my_module_log')
  ->keys([
    'lid' => 42,
  ])
  ->fields([
    'uid' => 1,
    'type' => 'notice',
    'message' => 'Updated or inserted.',
    'severity' => 5,
    'created' => \Drupal::time()->getRequestTime(),
  ])
  ->execute();
```

---

## Scaffolding with Drush

Use `ddev drush generate` to scaffold install files and hooks:

```bash
# Generate a .install file with hook_schema()
ddev drush gen hook

# Generate update hooks
ddev drush gen hook

# Generate a complete module (includes .install)
ddev drush gen module
```

When prompted, select the appropriate hook (`hook_schema`, `hook_update_N`, `hook_install`, `hook_uninstall`). The generator creates properly structured files with the correct function signatures.

After generating, always run:
```bash
ddev drush cr   # Clear cache
ddev drush updb # Run pending update hooks
```

---

## Key Principles

- **Always use placeholders** -- Never concatenate user input into queries. The Database API handles escaping automatically when using `condition()` and `fields()`.
- **Prefer dynamic queries** over static `query()` calls for portability across database backends.
- **Update hooks are irreversible** -- They run once and are tracked by number. Never change an update hook after it has been released.
- **Post-update hooks run after update hooks** -- Use them for data migrations that depend on schema changes made in `hook_update_N()`.
- **Clear cache after schema changes** -- Always run `ddev drush cr` after modifying schema definitions.
- **Use transactions** for multi-step operations that must succeed or fail together.

---

## Related Skills

- **drupal-entities** -- Entity API for CRUD operations on content and configuration entities.
- **drupal-services** -- How to inject the `database` service instead of using `\Drupal::database()`.
- **drupal-plugins** -- Plugin system that may need custom tables for storage.
- **drupal-config** -- Configuration API for storing settings (not raw data).
- **drupal-queue** -- Queue system that uses database tables by default.
