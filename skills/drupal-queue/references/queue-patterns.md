# Queue Patterns and Advanced Usage

## Reliable vs Unreliable Queues

Drupal defines two queue interfaces that differ in delivery guarantees:

### Reliable Queues (DatabaseQueue)

The default queue implementation uses the `queue` database table. It guarantees:

- **Persistence**: Items survive page requests, cron failures, and server restarts.
- **Atomic claiming**: `claimItem()` uses database locking to prevent two workers from processing the same item.
- **Lease expiration**: Claimed items that are not deleted within the lease time are automatically released for retry.

```php
// The 'queue' service returns reliable DatabaseQueue instances by default.
$queue = \Drupal::service('queue')->get('my_module_import');
$queue->createQueue();
$queue->createItem(['file' => '/tmp/data.csv', 'offset' => 0]);
```

The `queue` database table schema:

| Column | Type | Purpose |
|--------|------|---------|
| `item_id` | serial | Unique item identifier |
| `name` | varchar | Queue name |
| `data` | longblob | Serialized item data |
| `expire` | int | Lease expiration timestamp (0 = unclaimed) |
| `created` | int | Timestamp when item was created |

### Unreliable Queues

For high-throughput scenarios where occasional item loss is acceptable, use the `queue.database.unreliable` service or an in-memory queue:

```php
// Unreliable database queue — faster, fewer guarantees.
$queue = \Drupal::service('queue.database.unreliable')->get('my_module_stats');
```

Unreliable queues may skip locking or use less durable storage. They are appropriate for:
- Statistics collection
- Cache warming tasks
- Non-critical logging

---

## Custom Queue Backends (Redis)

You can replace the default `DatabaseQueue` with a Redis-backed queue for better performance. The `redis` contrib module provides this.

### Configuration in settings.php

```php
// Use Redis for all queues.
$settings['queue_default'] = 'queue.redis_reliable';

// Use Redis for a specific queue only.
$settings['queue_service_my_module_data_sync'] = 'queue.redis_reliable';

// Keep a specific queue on the database (e.g., for reliability).
$settings['queue_service_my_module_critical_jobs'] = 'queue.database';
```

### How Queue Service Resolution Works

When `QueueFactory::get('queue_name')` is called, Drupal checks:

1. `$settings['queue_service_{queue_name}']` — per-queue override.
2. `$settings['queue_default']` — global default.
3. Falls back to `queue.database` (the reliable `DatabaseQueue`).

---

## Exception Handling in QueueWorkers

Drupal provides several exception types to control queue processing flow in `processItem()`.

### Standard Exception

If `processItem()` throws any uncaught exception, the item remains in the queue and its lease expires. It will be retried on the next cron run.

```php
public function processItem($data): void {
  $response = $this->httpClient->get($data['url']);
  if ($response->getStatusCode() !== 200) {
    // Item stays in queue, will be retried after lease expires.
    throw new \RuntimeException('API returned ' . $response->getStatusCode());
  }
  // Process response...
}
```

### RequeueException

Use `Drupal\Core\Queue\RequeueException` to explicitly put the item back in the queue and continue processing other items. Unlike a standard exception, this does not count as an error.

```php
use Drupal\Core\Queue\RequeueException;

public function processItem($data): void {
  if ($this->rateLimiter->isThrottled()) {
    // Requeue this item; move on to the next one.
    throw new RequeueException('Rate limit reached, requeueing item.');
  }
  $this->processData($data);
}
```

### DelayedRequeueException

Available in Drupal 10.1+. Requeues the item with a specified delay before it becomes available again.

```php
use Drupal\Core\Queue\DelayedRequeueException;

public function processItem($data): void {
  if ($this->externalService->isTemporarilyUnavailable()) {
    // Requeue with a 60-second delay.
    throw new DelayedRequeueException(60, 'Service unavailable, retrying in 60s.');
  }
  $this->processData($data);
}
```

> **Note**: `DelayedRequeueException` depends on the queue backend supporting delays. `DatabaseQueue` supports it by updating the `expire` column. Not all backends support this feature.

### SuspendQueueException

Use `Drupal\Core\Queue\SuspendQueueException` to stop processing the entire queue for the current cron run. This is useful when an external dependency is down and retrying other items would also fail.

```php
use Drupal\Core\Queue\SuspendQueueException;

public function processItem($data): void {
  try {
    $this->apiClient->connect();
  }
  catch (ConnectionException $e) {
    // Stop processing this entire queue until the next cron run.
    throw new SuspendQueueException(
      'Cannot connect to external API. Suspending queue.',
      0,
      $e
    );
  }
  $this->apiClient->sync($data);
}
```

### Exception Summary

| Exception | Effect | Use When |
|-----------|--------|----------|
| `\Exception` (any) | Item stays in queue, lease expires, cron continues | Transient per-item failure |
| `RequeueException` | Item re-released immediately, next item processed | Item cannot be processed now but others can |
| `DelayedRequeueException` | Item re-released after delay | Temporary per-item issue, needs cooldown |
| `SuspendQueueException` | Entire queue stops for this cron run | External dependency is down |

---

## Drush Queue Commands

Drush provides built-in commands for inspecting and running queues.

```bash
# List all queues and their item counts.
ddev drush queue:list

# Run all items in a specific queue.
ddev drush queue:run my_module_data_sync

# Run with a time limit (in seconds).
ddev drush queue:run my_module_data_sync --time-limit=60

# Run with a specific number of items.
ddev drush queue:run my_module_data_sync --items-limit=100

# Delete all items from a queue.
ddev drush queue:delete my_module_data_sync
```

### Useful Aliases

```bash
# Short forms.
ddev drush q:list
ddev drush q:run my_module_data_sync
```

---

## Batch Operations with Entity Processing

When processing large numbers of entities, combine batch operations with careful memory management.

### Progressive Batch with Sandbox

Use the `$context['sandbox']` for tracking progress across batch iterations:

```php
/**
 * Batch operation that processes entities progressively.
 */
function my_module_batch_process_entities(string $entity_type, string $bundle, array &$context): void {
  $storage = \Drupal::entityTypeManager()->getStorage($entity_type);

  // Initialize sandbox on first call.
  if (!isset($context['sandbox']['progress'])) {
    $context['sandbox']['progress'] = 0;
    $context['sandbox']['max'] = (int) $storage->getQuery()
      ->accessCheck(FALSE)
      ->condition('type', $bundle)
      ->count()
      ->execute();
    $context['sandbox']['last_id'] = 0;
  }

  // Process 50 entities at a time.
  $limit = 50;
  $ids = $storage->getQuery()
    ->accessCheck(FALSE)
    ->condition('type', $bundle)
    ->condition('nid', $context['sandbox']['last_id'], '>')
    ->sort('nid', 'ASC')
    ->range(0, $limit)
    ->execute();

  $entities = $storage->loadMultiple($ids);
  foreach ($entities as $entity) {
    // Perform the operation.
    $entity->set('field_processed', TRUE);
    $entity->save();

    $context['sandbox']['progress']++;
    $context['sandbox']['last_id'] = $entity->id();
    $context['results']['processed'][] = $entity->id();
  }

  // Update progress (0 to 1).
  if ($context['sandbox']['max'] > 0) {
    $context['finished'] = $context['sandbox']['progress'] / $context['sandbox']['max'];
  }
  else {
    $context['finished'] = 1;
  }

  $context['message'] = t('Processed @progress of @max entities.', [
    '@progress' => $context['sandbox']['progress'],
    '@max' => $context['sandbox']['max'],
  ]);
}
```

> **Key**: When `$context['finished']` is less than 1, the batch system calls the same operation function again. This avoids loading all entities into memory at once.

---

## Memory Management in Long-Running Queues

When processing large queues (thousands of items), memory leaks can crash the process. Apply these strategies:

### 1. Reset Static Entity Cache

Drupal caches loaded entities in memory. Reset this after processing each item or batch of items:

```php
public function processItem($data): void {
  $storage = $this->entityTypeManager->getStorage('node');
  $entity = $storage->load($data['entity_id']);

  if ($entity) {
    $this->doWork($entity);
  }

  // Clear the static entity cache to free memory.
  $storage->resetCache([$data['entity_id']]);
}
```

### 2. Process in Chunks with Cache Resets

For manual queue processing (not cron), reset caches periodically:

```php
$queue = \Drupal::queue('my_module_bulk_import');
$worker = \Drupal::service('plugin.manager.queue_worker')
  ->createInstance('my_module_bulk_import');

$count = 0;
while ($item = $queue->claimItem()) {
  try {
    $worker->processItem($item->data);
    $queue->deleteItem($item);
    $count++;
  }
  catch (\Exception $e) {
    $queue->releaseItem($item);
  }

  // Reset entity caches every 100 items.
  if ($count % 100 === 0) {
    \Drupal::entityTypeManager()->getStorage('node')->resetCache();
    // Optionally trigger garbage collection.
    gc_collect_cycles();
  }
}
```

### 3. Monitor Memory Usage

Add memory guards to long-running processes:

```php
$memory_limit = 256 * 1024 * 1024; // 256 MB threshold.

while ($item = $queue->claimItem()) {
  if (memory_get_usage(TRUE) > $memory_limit) {
    $queue->releaseItem($item);
    \Drupal::logger('my_module')->warning('Memory limit approaching, stopping queue processing.');
    break;
  }

  try {
    $worker->processItem($item->data);
    $queue->deleteItem($item);
  }
  catch (\Exception $e) {
    $queue->releaseItem($item);
  }
}
```

---

## Queue Monitoring and Debugging

### Checking Queue Status Programmatically

```php
$queue = \Drupal::queue('my_module_data_sync');
$count = $queue->numberOfItems();

\Drupal::logger('my_module')->info('Queue has @count items pending.', [
  '@count' => $count,
]);
```

### Database Inspection

For `DatabaseQueue`, you can inspect items directly:

```php
// Count items in a specific queue.
$count = \Drupal::database()->select('queue', 'q')
  ->condition('name', 'my_module_data_sync')
  ->countQuery()
  ->execute()
  ->fetchField();

// Find stale claimed items (lease expired but not deleted).
$stale = \Drupal::database()->select('queue', 'q')
  ->fields('q')
  ->condition('name', 'my_module_data_sync')
  ->condition('expire', 0, '>')
  ->condition('expire', \Drupal::time()->getRequestTime(), '<')
  ->execute()
  ->fetchAll();
```

### Implementing hook_cron for Custom Queue Logic

If you need more control than a QueueWorker provides, process queues in `hook_cron`:

```php
/**
 * Implements hook_cron().
 */
function my_module_cron(): void {
  $queue = \Drupal::queue('my_module_custom_process');
  $limit = 100;
  $processed = 0;

  while ($processed < $limit && ($item = $queue->claimItem())) {
    try {
      _my_module_process_queue_item($item->data);
      $queue->deleteItem($item);
      $processed++;
    }
    catch (\Exception $e) {
      $queue->releaseItem($item);
      \Drupal::logger('my_module')->error('Failed to process item @id: @message', [
        '@id' => $item->item_id,
        '@message' => $e->getMessage(),
      ]);
    }
  }
}
```

---

## Cron Queue Configuration

### Overriding Cron Time per Queue

You can override the cron processing time for any QueueWorker via `settings.php`:

```php
// Override cron time for a specific queue (in seconds).
$settings['queue_workers']['my_module_data_sync'] = [
  'cron' => ['time' => 120],
];
```

### Disabling Cron Processing for a Queue

Set the cron time to 0 in the annotation/attribute or settings to prevent cron from running the worker. This is useful for queues processed by dedicated Drush commands:

```php
// In the QueueWorker attribute:
#[QueueWorker(
  id: 'my_module_manual_only',
  title: new TranslatableMarkup('Manual processing only'),
  cron: ['time' => 0],
)]
```

Then process via a custom Drush command or controller instead.
