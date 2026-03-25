---
name: drupal-queue
description: Use when working with Drupal queues, QueueWorker plugins, queue processing, batch operations, or cron-based queue running.
version: 1.0.0
---

# Drupal Queue API

## Overview

Drupal's Queue API provides a mechanism for deferring work to be processed later, either during cron runs or via explicit processing. Queues decouple the creation of work items from their execution, enabling background processing of expensive tasks like sending emails, syncing data with external APIs, or processing large entity sets.

### Core Concepts

- **Queue**: A FIFO data structure that holds items for later processing. Each queue is identified by a unique name.
- **QueueFactory**: The service (`queue`) used to create or retrieve queue instances.
- **QueueWorker Plugin**: A plugin that defines how items from a specific queue are processed. Typically triggered by cron.
- **Reliable vs Unreliable Queues**: Reliable queues (e.g., `DatabaseQueue`) guarantee item delivery — items persist across requests and survive failures. Unreliable queues (e.g., in-memory) may lose items on failure.
- **Cron-Based Processing**: By default, Drupal's cron runs all QueueWorker plugins, processing items for a configurable time window.

### Queue Lifecycle

1. A queue is created via `QueueFactory::get()`.
2. Items are added with `createItem($data)`.
3. During cron (or manual processing), items are claimed with `claimItem()`.
4. The worker processes the item via `processItem($data)`.
5. On success, the item is removed with `deleteItem($item)`.
6. On failure, the item is released back with `releaseItem($item)` for retry.

---

## Creating Queue Items

Use the `queue` service (an instance of `QueueFactory`) to get a queue and add items.

```php
<?php

namespace Drupal\my_module\Service;

use Drupal\Core\Queue\QueueFactory;

class DataSyncService {

  public function __construct(
    protected QueueFactory $queueFactory,
  ) {}

  /**
   * Enqueues entities for background synchronization.
   */
  public function enqueueEntities(array $entity_ids): void {
    $queue = $this->queueFactory->get('my_module_data_sync');

    // Ensure the underlying storage exists (idempotent for DatabaseQueue).
    $queue->createQueue();

    foreach ($entity_ids as $entity_id) {
      $queue->createItem([
        'entity_id' => $entity_id,
        'operation' => 'sync',
        'created' => \Drupal::time()->getRequestTime(),
      ]);
    }
  }

}
```

Register the service in `my_module.services.yml`:

```yaml
services:
  my_module.data_sync:
    class: Drupal\my_module\Service\DataSyncService
    arguments: ['@queue']
```

---

## QueueWorker Plugin

A `QueueWorker` plugin defines how to process items from a named queue. Drupal's cron system automatically discovers and runs these plugins.

### Drupal 10 (Annotation Syntax)

```php
<?php

namespace Drupal\my_module\Plugin\QueueWorker;

use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Plugin\ContainerFactoryPluginInterface;
use Drupal\Core\Queue\QueueWorkerBase;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Processes data sync queue items.
 *
 * @QueueWorker(
 *   id = "my_module_data_sync",
 *   title = @Translation("Data sync worker"),
 *   cron = {"time" = 30}
 * )
 */
class DataSyncWorker extends QueueWorkerBase implements ContainerFactoryPluginInterface {

  public function __construct(
    array $configuration,
    $plugin_id,
    $plugin_definition,
    protected EntityTypeManagerInterface $entityTypeManager,
  ) {
    parent::__construct($configuration, $plugin_id, $plugin_definition);
  }

  /**
   * {@inheritdoc}
   */
  public static function create(ContainerInterface $container, array $configuration, $plugin_id, $plugin_definition): static {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition,
      $container->get('entity_type.manager'),
    );
  }

  /**
   * {@inheritdoc}
   */
  public function processItem($data): void {
    $entity = $this->entityTypeManager->getStorage('node')->load($data['entity_id']);
    if (!$entity) {
      // Entity no longer exists; nothing to do. Item will be deleted.
      return;
    }

    // Perform the sync operation.
    // If this method throws an exception, the item remains in the queue
    // and will be retried on the next cron run.
    $this->syncEntity($entity, $data['operation']);
  }

  protected function syncEntity($entity, string $operation): void {
    // Implementation here.
  }

}
```

### Drupal 11+ (Attribute Syntax)

```php
<?php

namespace Drupal\my_module\Plugin\QueueWorker;

use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Plugin\ContainerFactoryPluginInterface;
use Drupal\Core\Queue\Attribute\QueueWorker;
use Drupal\Core\Queue\QueueWorkerBase;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use Symfony\Component\DependencyInjection\ContainerInterface;

#[QueueWorker(
  id: 'my_module_data_sync',
  title: new TranslatableMarkup('Data sync worker'),
  cron: ['time' => 30],
)]
class DataSyncWorker extends QueueWorkerBase implements ContainerFactoryPluginInterface {

  public function __construct(
    array $configuration,
    $plugin_id,
    $plugin_definition,
    protected EntityTypeManagerInterface $entityTypeManager,
  ) {
    parent::__construct($configuration, $plugin_id, $plugin_definition);
  }

  /**
   * {@inheritdoc}
   */
  public static function create(ContainerInterface $container, array $configuration, $plugin_id, $plugin_definition): static {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition,
      $container->get('entity_type.manager'),
    );
  }

  /**
   * {@inheritdoc}
   */
  public function processItem($data): void {
    $entity = $this->entityTypeManager->getStorage('node')->load($data['entity_id']);
    if (!$entity) {
      return;
    }
    $this->syncEntity($entity, $data['operation']);
  }

  protected function syncEntity($entity, string $operation): void {
    // Implementation here.
  }

}
```

> **cron property**: The `cron` key sets how many seconds the worker runs per cron invocation. `{"time" = 30}` (annotation) or `['time' => 30]` (attribute) means 30 seconds maximum. Set `{"time" = 0}` to skip cron processing entirely (manual-only queue).

---

## Processing Queues Manually

You can process queue items outside of cron — for example, in a Drush command or a controller.

```php
$queue = \Drupal::queue('my_module_data_sync');
$worker = \Drupal::service('plugin.manager.queue_worker')
  ->createInstance('my_module_data_sync');

while ($item = $queue->claimItem()) {
  try {
    $worker->processItem($item->data);
    $queue->deleteItem($item);
  }
  catch (\Exception $e) {
    // Release the item so it can be retried later.
    $queue->releaseItem($item);
    \Drupal::logger('my_module')->error('Queue processing failed: @message', [
      '@message' => $e->getMessage(),
    ]);
  }
}
```

### Key Methods on QueueInterface

| Method | Description |
|--------|-------------|
| `createItem($data)` | Add an item to the queue. Returns a unique item ID or FALSE. |
| `numberOfItems()` | Returns the number of items in the queue. |
| `claimItem($lease_time = 30)` | Claims an item for processing. Returns an object with `->data` and `->item_id`, or FALSE. |
| `deleteItem($item)` | Removes a successfully processed item. |
| `releaseItem($item)` | Releases a claimed item back to the queue for retry. |
| `createQueue()` | Creates the underlying storage (idempotent). |
| `deleteQueue()` | Deletes the queue and all its items. |

---

## Batch API

The Batch API processes large operations across multiple HTTP requests, showing a progress bar to the user. It is not the same as queues, but the two are often used together.

### Basic Batch Operation

```php
/**
 * Starts a batch operation to update nodes.
 */
function my_module_batch_update(array $nids): void {
  $operations = [];
  foreach (array_chunk($nids, 50) as $chunk) {
    $operations[] = ['my_module_batch_process', [$chunk]];
  }

  $batch = [
    'title' => t('Processing nodes...'),
    'operations' => $operations,
    'finished' => 'my_module_batch_finished',
    'init_message' => t('Starting node processing.'),
    'progress_message' => t('Processed @current out of @total batches.'),
    'error_message' => t('An error occurred during processing.'),
  ];

  batch_set($batch);
}

/**
 * Batch operation callback.
 */
function my_module_batch_process(array $nids, array &$context): void {
  if (!isset($context['results']['processed'])) {
    $context['results']['processed'] = 0;
  }

  $storage = \Drupal::entityTypeManager()->getStorage('node');
  foreach ($nids as $nid) {
    $node = $storage->load($nid);
    if ($node) {
      $node->set('field_status', 'processed');
      $node->save();
      $context['results']['processed']++;
    }
  }

  $context['message'] = t('Processed @count nodes.', [
    '@count' => $context['results']['processed'],
  ]);
}

/**
 * Batch finished callback.
 */
function my_module_batch_finished(bool $success, array $results, array $operations): void {
  if ($success) {
    \Drupal::messenger()->addStatus(t('Successfully processed @count nodes.', [
      '@count' => $results['processed'] ?? 0,
    ]));
  }
  else {
    \Drupal::messenger()->addError(t('An error occurred during batch processing.'));
  }
}
```

---

## Combining Batch + Queue

A common pattern: use the Batch API to populate a queue (with a progress bar for the user), then let cron-based QueueWorkers process the items in the background.

```php
/**
 * Batch callback that adds items to a queue.
 */
function my_module_batch_enqueue(array $entity_ids, array &$context): void {
  $queue = \Drupal::queue('my_module_data_sync');

  foreach ($entity_ids as $id) {
    $queue->createItem(['entity_id' => $id, 'operation' => 'sync']);
  }

  $context['results']['enqueued'] = ($context['results']['enqueued'] ?? 0) + count($entity_ids);
  $context['message'] = t('Enqueued @count items for processing.', [
    '@count' => $context['results']['enqueued'],
  ]);
}
```

This approach gives the user immediate feedback (the batch progress bar) while the heavy processing happens asynchronously via cron.

---

## Scaffolding with Drush

Use Drush generators to scaffold QueueWorker plugins:

```bash
# Generate a QueueWorker plugin
ddev drush gen plugin:queue-worker

# The generator prompts for:
#   - Module name
#   - Plugin ID (queue name)
#   - Plugin label
#   - Class name
#   - Cron processing time
```

After generating:
1. Clear cache: `ddev drush cr`
2. Add items to the queue (via service, form submit, etc.).
3. Run cron to process: `ddev drush cron` or `ddev drush queue:run <queue_name>`

---

## Key Principles

- **Always clear cache** after adding or modifying a QueueWorker plugin (`ddev drush cr`).
- **Use dependency injection** in QueueWorker plugins via `ContainerFactoryPluginInterface`.
- **Handle exceptions gracefully** in `processItem()` — unhandled exceptions leave the item in the queue for retry. Use `RequeueException` to explicitly requeue.
- **Set appropriate cron times** — the `cron.time` property limits how long processing runs per cron invocation. Tune based on item processing speed.
- **Chunk large data sets** — when populating queues, batch the data to avoid memory issues.
- **Prefix queue names** with your module name to avoid collisions (e.g., `my_module_data_sync`).

---

## Related Skills

- **drupal-plugins** -- Plugin system fundamentals, annotations vs attributes, plugin managers.
- **drupal-services** -- Dependency injection and service definitions for queue services.
- **drupal-hooks** -- Hook system for cron hooks (`hook_cron`) and alter hooks.
- **drupal-database** -- Database API for custom queue backends or direct queries.
- **drush-generate** -- Full reference for Drush code generation commands.
