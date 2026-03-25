# Database API -- Deep Dive

## Joins

The select query builder supports inner joins, left joins, and right joins.

### Inner Join

```php
$database = \Drupal::database();

$query = $database->select('my_module_log', 'l');
$query->innerJoin('users_field_data', 'u', 'l.uid = u.uid');
$query->fields('l', ['lid', 'type', 'message', 'created']);
$query->fields('u', ['name']);
$query->condition('l.type', 'error');
$query->orderBy('l.created', 'DESC');

$result = $query->execute();
foreach ($result as $record) {
  // $record->name is available from the joined table.
}
```

### Left Join

```php
$query = $database->select('node_field_data', 'n');
$query->leftJoin('my_module_tracking', 't', 'n.nid = t.entity_id AND t.entity_type = :type', [
  ':type' => 'node',
]);
$query->fields('n', ['nid', 'title']);
$query->addField('t', 'action', 'tracking_action');
$query->isNull('t.tid'); // Nodes with no tracking entries.

$result = $query->execute();
```

### Multiple Joins

```php
$query = $database->select('my_module_log', 'l');
$query->innerJoin('users_field_data', 'u', 'l.uid = u.uid');
$query->leftJoin('user__roles', 'r', 'u.uid = r.entity_id');
$query->fields('l', ['lid', 'message']);
$query->fields('u', ['name']);
$query->addField('r', 'roles_target_id', 'role');
$query->condition('l.severity', 3, '<=');

$result = $query->execute();
```

---

## Aggregate Queries

### groupBy and Aggregate Functions

```php
$query = $database->select('my_module_log', 'l');
$query->addField('l', 'type');
$query->addExpression('COUNT(l.lid)', 'entry_count');
$query->addExpression('MAX(l.created)', 'latest');
$query->addExpression('AVG(l.severity)', 'avg_severity');
$query->groupBy('l.type');
$query->orderBy('entry_count', 'DESC');

$result = $query->execute();
foreach ($result as $record) {
  // $record->type, $record->entry_count, $record->latest, $record->avg_severity
}
```

### HAVING Clause

```php
$query = $database->select('my_module_log', 'l');
$query->addField('l', 'uid');
$query->addExpression('COUNT(l.lid)', 'entry_count');
$query->groupBy('l.uid');
$query->havingCondition('entry_count', 100, '>');

$result = $query->execute();
```

### Aggregate with Join

```php
$query = $database->select('my_module_tracking', 't');
$query->innerJoin('users_field_data', 'u', 't.uid = u.uid');
$query->addField('u', 'name');
$query->addField('t', 'entity_type');
$query->addExpression('COUNT(t.tid)', 'interaction_count');
$query->groupBy('u.name');
$query->groupBy('t.entity_type');
$query->havingCondition('interaction_count', 10, '>=');
$query->orderBy('interaction_count', 'DESC');

$result = $query->execute();
```

---

## Subqueries

### Subquery in a Condition

```php
// Find users who have more than 50 log entries.
$subquery = $database->select('my_module_log', 'l');
$subquery->addField('l', 'uid');
$subquery->addExpression('COUNT(*)', 'cnt');
$subquery->groupBy('l.uid');
$subquery->havingCondition('cnt', 50, '>');

// Use the subquery as a condition.
$query = $database->select('users_field_data', 'u');
$query->fields('u', ['uid', 'name']);
$query->condition('u.uid', $subquery, 'IN');

$result = $query->execute();
```

### Subquery as an Expression

```php
$query = $database->select('users_field_data', 'u');
$query->fields('u', ['uid', 'name']);

// Add a correlated subquery as a field.
$subquery = $database->select('my_module_log', 'l');
$subquery->addExpression('COUNT(*)');
$subquery->where('l.uid = u.uid');
$query->addExpression('(' . (string) $subquery . ')', 'log_count', $subquery->getArguments());

$result = $query->execute();
```

---

## Transactions

Wrap multiple operations in a transaction to ensure atomicity. If an exception occurs, the transaction rolls back automatically when the transaction object goes out of scope.

```php
$database = \Drupal::database();
$transaction = $database->startTransaction();

try {
  // Insert a parent record.
  $lid = $database->insert('my_module_log')
    ->fields([
      'uid' => 1,
      'type' => 'batch',
      'message' => 'Batch operation started.',
      'severity' => 6,
      'created' => \Drupal::time()->getRequestTime(),
    ])
    ->execute();

  // Insert related records.
  foreach ($items as $item) {
    $database->insert('my_module_tracking')
      ->fields([
        'entity_type' => $item['entity_type'],
        'entity_id' => $item['entity_id'],
        'uid' => 1,
        'action' => 'batch_process',
        'created' => \Drupal::time()->getRequestTime(),
      ])
      ->execute();
  }

  // Update the parent record.
  $database->update('my_module_log')
    ->fields(['message' => 'Batch operation completed.'])
    ->condition('lid', $lid)
    ->execute();
}
catch (\Exception $e) {
  $transaction->rollBack();
  \Drupal::logger('my_module')->error('Batch operation failed: @message', [
    '@message' => $e->getMessage(),
  ]);
  throw $e;
}
```

### Transaction Rules

- The transaction is committed when the `$transaction` object is destroyed (goes out of scope).
- Call `$transaction->rollBack()` explicitly in catch blocks.
- Transactions can be nested; Drupal uses savepoints for inner transactions.
- Do not call `unset($transaction)` to commit -- let it go out of scope naturally or assign `NULL` to the variable.

---

## Static Queries

Use `query()` for raw SQL when the dynamic query builder cannot express the query. Always use placeholders for user-supplied values.

```php
$database = \Drupal::database();

// Simple static query.
$result = $database->query(
  'SELECT l.lid, l.message, u.name
   FROM {my_module_log} l
   INNER JOIN {users_field_data} u ON l.uid = u.uid
   WHERE l.type = :type AND l.created > :since
   ORDER BY l.created DESC',
  [
    ':type' => 'error',
    ':since' => strtotime('-7 days'),
  ]
);

foreach ($result as $record) {
  // Access fields as object properties.
}
```

### Static Query Rules

- Always wrap table names in curly braces: `{table_name}`. Drupal adds the table prefix automatically.
- Always use named placeholders (`:name`) for values. Never concatenate variables.
- `query()` returns a `StatementInterface` object, iterable like dynamic query results.
- Use static queries for database-specific functions or complex SQL not supported by the query builder (e.g., `UNION`, window functions, `CASE` expressions).

### UNION Example

```php
$sql = '(SELECT lid AS id, message AS label, created FROM {my_module_log} WHERE type = :type1)
        UNION ALL
        (SELECT tid AS id, action AS label, created FROM {my_module_tracking} WHERE entity_type = :etype)
        ORDER BY created DESC
        LIMIT 20';

$result = $database->query($sql, [
  ':type1' => 'error',
  ':etype' => 'node',
]);
```

---

## Tagged Queries

Tags allow other modules to alter queries via `hook_query_TAG_alter()`.

```php
// Add a tag to a select query.
$query = $database->select('my_module_log', 'l')
  ->fields('l')
  ->condition('type', 'error');
$query->addTag('my_module_log_listing');
$query->addTag('node_access'); // Common tag for access control.

$result = $query->execute();
```

Other modules can then alter the query:

```php
/**
 * Implements hook_query_my_module_log_listing_alter().
 */
function other_module_query_my_module_log_listing_alter(
  \Drupal\Core\Database\Query\AlterableInterface $query
): void {
  // Add an extra condition.
  $query->condition('l.severity', 4, '<=');
}
```

### Common Built-in Tags

- `node_access` -- Adds node access restrictions to the query.
- `entity_query` -- Used internally by entity queries.
- `translatable` -- Marks queries that filter on translatable fields.
- `pager` -- Indicates the query powers a pager.

---

## Connection Targets

Drupal supports multiple database connections (e.g., default and replica).

### Using the Replica Connection

```php
// Read from replica for expensive queries.
$database = \Drupal\Core\Database\Database::getConnection('replica');

$result = $database->select('my_module_log', 'l')
  ->fields('l')
  ->condition('created', strtotime('-24 hours'), '>')
  ->execute();

// Always write to the default connection.
$default = \Drupal\Core\Database\Database::getConnection('default');
$default->insert('my_module_log')
  ->fields([...])
  ->execute();
```

### Database Connection Configuration

Connections are defined in `settings.php`:

```php
$databases['default']['default'] = [
  'driver' => 'mysql',
  'database' => 'drupal',
  'username' => 'drupal',
  'password' => 'drupal',
  'host' => 'db',
  'port' => '3306',
  'prefix' => '',
];

$databases['default']['replica'] = [
  'driver' => 'mysql',
  'database' => 'drupal',
  'username' => 'readonly',
  'password' => 'readonly',
  'host' => 'db-replica',
  'port' => '3306',
  'prefix' => '',
];
```

---

## Condition Groups (OR, AND)

### OR Conditions

```php
$query = $database->select('my_module_log', 'l')
  ->fields('l');

$or_group = $query->orConditionGroup()
  ->condition('type', 'error')
  ->condition('severity', 2, '<=');
$query->condition($or_group);

$result = $query->execute();
```

### Nested Condition Groups

```php
$query = $database->select('my_module_log', 'l')
  ->fields('l');

// WHERE (type = 'error' OR severity <= 2) AND uid = 1
$or_group = $query->orConditionGroup()
  ->condition('type', 'error')
  ->condition('severity', 2, '<=');
$query->condition($or_group);
$query->condition('uid', 1);

$result = $query->execute();
```

---

## Fetching Results

The execute method returns a statement object with multiple fetch methods:

```php
$query = $database->select('my_module_log', 'l')
  ->fields('l', ['lid', 'type', 'message']);

$result = $query->execute();

// Iterate as objects (default).
foreach ($result as $record) {
  // $record->lid, $record->type, $record->message
}

// Fetch a single field value.
$count = $database->select('my_module_log', 'l')
  ->countQuery()
  ->execute()
  ->fetchField();

// Fetch a single row as an object.
$record = $database->select('my_module_log', 'l')
  ->fields('l')
  ->condition('lid', 42)
  ->execute()
  ->fetchObject();

// Fetch a single row as an associative array.
$row = $database->select('my_module_log', 'l')
  ->fields('l')
  ->condition('lid', 42)
  ->execute()
  ->fetchAssoc();

// Fetch all rows as an array of objects.
$records = $database->select('my_module_log', 'l')
  ->fields('l')
  ->execute()
  ->fetchAll();

// Fetch as key-value pairs (first column = key, second = value).
$map = $database->select('my_module_log', 'l')
  ->fields('l', ['lid', 'message'])
  ->execute()
  ->fetchAllKeyed();

// Fetch a single column as a flat array.
$lids = $database->select('my_module_log', 'l')
  ->fields('l', ['lid'])
  ->execute()
  ->fetchCol();
```

---

## Condition Operators

The `condition()` method supports these operators:

| Operator | Example | Description |
|----------|---------|-------------|
| `=` | `condition('type', 'error')` | Equals (default) |
| `<>` | `condition('type', 'debug', '<>')` | Not equals |
| `>` | `condition('severity', 3, '>')` | Greater than |
| `>=` | `condition('severity', 3, '>=')` | Greater than or equal |
| `<` | `condition('created', $time, '<')` | Less than |
| `<=` | `condition('created', $time, '<=')` | Less than or equal |
| `IN` | `condition('type', ['error', 'warning'], 'IN')` | In array |
| `NOT IN` | `condition('type', ['debug'], 'NOT IN')` | Not in array |
| `BETWEEN` | `condition('created', [$start, $end], 'BETWEEN')` | Between two values |
| `LIKE` | `condition('message', '%fail%', 'LIKE')` | Pattern matching |
| `IS NULL` | `isNull('field')` | Null check |
| `IS NOT NULL` | `isNotNull('field')` | Not null check |
