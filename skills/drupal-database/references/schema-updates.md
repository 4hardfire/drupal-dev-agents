# Schema Definitions and Update Hooks -- Deep Dive

## Complex Schema Definitions

### Multi-Column Indexes and Unique Keys

```php
function my_module_schema(): array {
  $schema['my_module_tracking'] = [
    'description' => 'Tracks entity interactions.',
    'fields' => [
      'tid' => [
        'type' => 'serial',
        'unsigned' => TRUE,
        'not null' => TRUE,
      ],
      'entity_type' => [
        'type' => 'varchar_ascii',
        'length' => 128,
        'not null' => TRUE,
        'default' => '',
      ],
      'entity_id' => [
        'type' => 'int',
        'unsigned' => TRUE,
        'not null' => TRUE,
        'default' => 0,
      ],
      'langcode' => [
        'type' => 'varchar_ascii',
        'length' => 12,
        'not null' => TRUE,
        'default' => '',
      ],
      'uid' => [
        'type' => 'int',
        'unsigned' => TRUE,
        'not null' => TRUE,
        'default' => 0,
      ],
      'action' => [
        'type' => 'varchar_ascii',
        'length' => 32,
        'not null' => TRUE,
        'default' => '',
      ],
      'data' => [
        'type' => 'blob',
        'size' => 'big',
        'not null' => FALSE,
        'description' => 'Serialized additional data.',
      ],
      'weight' => [
        'type' => 'int',
        'not null' => TRUE,
        'default' => 0,
      ],
      'created' => [
        'type' => 'int',
        'not null' => TRUE,
        'default' => 0,
      ],
    ],
    'primary key' => ['tid'],
    'unique keys' => [
      'entity_user_action' => ['entity_type', 'entity_id', 'uid', 'action'],
    ],
    'indexes' => [
      'entity' => ['entity_type', 'entity_id'],
      'entity_lang' => ['entity_type', 'entity_id', 'langcode'],
      'uid' => ['uid'],
      'created' => ['created'],
      'weight_created' => ['weight', 'created'],
      // Partial index on varchar: specify prefix length.
      'action_type' => [['entity_type', 64], 'action'],
    ],
    'foreign keys' => [
      'tracking_user' => [
        'table' => 'users',
        'columns' => ['uid' => 'uid'],
      ],
    ],
  ];

  return $schema;
}
```

> **Note on foreign keys**: Drupal's Schema API accepts foreign key definitions for documentation purposes, but does not enforce them at the database level. They serve as metadata for developers.

### Varchar Prefix Indexes

When indexing long varchar columns, specify a prefix length to limit the index size:

```php
'indexes' => [
  // Index only the first 64 characters of entity_type.
  'entity_prefix' => [['entity_type', 64]],
],
```

---

## Schema Modification in Update Hooks

The `\Drupal\Core\Database\Schema` object provides methods for altering tables:

### Adding Columns

```php
/**
 * Add 'priority' and 'expires' columns to {my_module_tracking}.
 */
function my_module_update_10001(): string {
  $schema = \Drupal::database()->schema();

  $spec_priority = [
    'type' => 'int',
    'size' => 'small',
    'not null' => TRUE,
    'default' => 0,
    'description' => 'Priority level.',
  ];
  $schema->addField('my_module_tracking', 'priority', $spec_priority);

  $spec_expires = [
    'type' => 'int',
    'not null' => FALSE,
    'description' => 'Unix timestamp when the entry expires.',
  ];
  $schema->addField('my_module_tracking', 'expires', $spec_expires);

  return 'Added priority and expires columns.';
}
```

### Changing Column Definitions

```php
/**
 * Change 'action' column from varchar(32) to varchar(128).
 */
function my_module_update_10002(): string {
  $schema = \Drupal::database()->schema();

  $new_spec = [
    'type' => 'varchar_ascii',
    'length' => 128,
    'not null' => TRUE,
    'default' => '',
    'description' => 'The action that was performed.',
  ];
  // Third parameter is keys_new: updated indexes for this field.
  $schema->changeField('my_module_tracking', 'action', 'action', $new_spec);

  return 'Changed action column length to 128.';
}
```

### Removing Columns

```php
/**
 * Remove the deprecated 'weight' column.
 */
function my_module_update_10003(): string {
  $schema = \Drupal::database()->schema();

  // Drop indexes that include this column first.
  if ($schema->indexExists('my_module_tracking', 'weight_created')) {
    $schema->dropIndex('my_module_tracking', 'weight_created');
  }

  $schema->dropField('my_module_tracking', 'weight');

  return 'Removed weight column and its index.';
}
```

### Adding and Dropping Indexes

```php
/**
 * Add composite index on priority and created.
 */
function my_module_update_10004(): string {
  $schema = \Drupal::database()->schema();

  // addIndex requires the field spec for the indexed columns.
  $fields_spec = [
    'fields' => [
      'priority' => [
        'type' => 'int',
        'size' => 'small',
        'not null' => TRUE,
        'default' => 0,
      ],
      'created' => [
        'type' => 'int',
        'not null' => TRUE,
        'default' => 0,
      ],
    ],
  ];
  $schema->addIndex('my_module_tracking', 'priority_created', ['priority', 'created'], $fields_spec);

  return 'Added priority_created index.';
}
```

### Adding and Dropping Unique Keys

```php
/**
 * Add a unique key on entity_type + entity_id + langcode.
 */
function my_module_update_10005(): string {
  $schema = \Drupal::database()->schema();

  $schema->addUniqueKey('my_module_tracking', 'entity_lang_unique', [
    'entity_type', 'entity_id', 'langcode',
  ]);

  return 'Added unique key on entity_type, entity_id, langcode.';
}

/**
 * Drop the old unique key.
 */
function my_module_update_10006(): string {
  $schema = \Drupal::database()->schema();

  $schema->dropUniqueKey('my_module_tracking', 'entity_user_action');

  return 'Dropped entity_user_action unique key.';
}
```

### Creating and Dropping Entire Tables

```php
/**
 * Create the {my_module_archive} table.
 */
function my_module_update_10007(): string {
  $schema = \Drupal::database()->schema();

  if (!$schema->tableExists('my_module_archive')) {
    $table_spec = [
      'description' => 'Archive of old tracking entries.',
      'fields' => [
        'aid' => [
          'type' => 'serial',
          'unsigned' => TRUE,
          'not null' => TRUE,
        ],
        'original_tid' => [
          'type' => 'int',
          'unsigned' => TRUE,
          'not null' => TRUE,
        ],
        'data' => [
          'type' => 'blob',
          'size' => 'big',
          'not null' => FALSE,
        ],
        'archived' => [
          'type' => 'int',
          'not null' => TRUE,
          'default' => 0,
        ],
      ],
      'primary key' => ['aid'],
      'indexes' => [
        'original_tid' => ['original_tid'],
        'archived' => ['archived'],
      ],
    ];
    $schema->createTable('my_module_archive', $table_spec);
  }

  return 'Created my_module_archive table.';
}

/**
 * Drop the deprecated {my_module_temp} table.
 */
function my_module_update_10008(): string {
  $schema = \Drupal::database()->schema();

  if ($schema->tableExists('my_module_temp')) {
    $schema->dropTable('my_module_temp');
  }

  return 'Dropped my_module_temp table.';
}
```

---

## Install and Uninstall Hooks

### hook_install()

Runs once when the module is first installed, after `hook_schema()` tables are created.

```php
/**
 * Implements hook_install().
 */
function my_module_install(): void {
  // Set default configuration values.
  \Drupal::configFactory()->getEditable('my_module.settings')
    ->set('retention_days', 90)
    ->set('enabled', TRUE)
    ->save();

  // Insert initial data.
  \Drupal::database()->insert('my_module_log')
    ->fields([
      'uid' => 1,
      'type' => 'system',
      'message' => 'Module installed.',
      'severity' => 6,
      'created' => \Drupal::time()->getRequestTime(),
    ])
    ->execute();
}
```

### hook_uninstall()

Runs when the module is uninstalled. Tables defined in `hook_schema()` are dropped automatically; use this for additional cleanup.

```php
/**
 * Implements hook_uninstall().
 */
function my_module_uninstall(): void {
  // Delete module configuration.
  \Drupal::configFactory()->getEditable('my_module.settings')->delete();

  // Clean up any state values.
  \Drupal::state()->delete('my_module.last_run');

  // Drop tables not defined in hook_schema() (if any).
  $schema = \Drupal::database()->schema();
  if ($schema->tableExists('my_module_archive')) {
    $schema->dropTable('my_module_archive');
  }
}
```

---

## hook_requirements()

Check system requirements at install time and on the status report page.

```php
/**
 * Implements hook_requirements().
 */
function my_module_requirements(string $phase): array {
  $requirements = [];

  if ($phase === 'install') {
    // Check prerequisites before allowing installation.
    if (!class_exists('SomeExternalLibrary')) {
      $requirements['my_module_library'] = [
        'title' => t('My Module Library'),
        'description' => t('The SomeExternalLibrary class is required.'),
        'severity' => REQUIREMENT_ERROR,
      ];
    }
  }

  if ($phase === 'runtime') {
    $count = \Drupal::database()->select('my_module_log', 'l')
      ->countQuery()
      ->execute()
      ->fetchField();

    $requirements['my_module_log_count'] = [
      'title' => t('My Module Log Entries'),
      'value' => t('@count entries', ['@count' => $count]),
      'severity' => $count > 100000 ? REQUIREMENT_WARNING : REQUIREMENT_OK,
      'description' => $count > 100000
        ? t('Consider archiving old log entries to improve performance.')
        : NULL,
    ];
  }

  return $requirements;
}
```

---

## Batch Operations in Update Hooks

When an update hook must process thousands of rows, use the `$sandbox` parameter to process in batches. Drupal calls the update hook repeatedly until `$sandbox['#finished']` reaches 1.

### Pattern for Batch Updates

```php
/**
 * Migrate data from old format to new format.
 */
function my_module_update_10009(array &$sandbox): string {
  $database = \Drupal::database();

  // Initialize on first pass.
  if (!isset($sandbox['progress'])) {
    $sandbox['progress'] = 0;
    $sandbox['last_id'] = 0;
    $sandbox['max'] = $database->select('my_module_tracking', 't')
      ->countQuery()
      ->execute()
      ->fetchField();

    if ($sandbox['max'] == 0) {
      $sandbox['#finished'] = 1;
      return 'No entries to process.';
    }
  }

  $batch_size = 200;

  // Use last_id cursor for efficient paging (avoids OFFSET performance issues).
  $records = $database->select('my_module_tracking', 't')
    ->fields('t', ['tid', 'data'])
    ->condition('tid', $sandbox['last_id'], '>')
    ->orderBy('tid')
    ->range(0, $batch_size)
    ->execute()
    ->fetchAll();

  foreach ($records as $record) {
    // Perform the migration.
    $new_data = _my_module_transform_data($record->data);
    $database->update('my_module_tracking')
      ->fields(['data' => $new_data])
      ->condition('tid', $record->tid)
      ->execute();

    $sandbox['progress']++;
    $sandbox['last_id'] = $record->tid;
  }

  $sandbox['#finished'] = $sandbox['progress'] / $sandbox['max'];

  return 'Processed ' . $sandbox['progress'] . ' of ' . $sandbox['max'] . ' entries.';
}
```

### Key Batch Update Rules

- Always set `$sandbox['#finished']` as a float between 0 and 1.
- Use a cursor (like `last_id`) instead of `OFFSET` for better performance on large tables.
- Keep batch sizes reasonable (100-500 rows) to avoid timeouts.
- The return value of the final call is shown as the update message.
- If no rows need processing, set `$sandbox['#finished'] = 1` immediately.
