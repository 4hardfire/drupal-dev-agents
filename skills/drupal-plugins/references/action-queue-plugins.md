# Action and QueueWorker Plugins

## Action Plugins

Action plugins perform operations on entities, typically used in Views bulk operations and the core Action system.

### Drupal 10 (Annotation)

```php
<?php

namespace Drupal\my_module\Plugin\Action;

use Drupal\Core\Action\ActionBase;
use Drupal\Core\Session\AccountInterface;

/**
 * Marks content as featured.
 *
 * @Action(
 *   id = "my_module_mark_featured",
 *   label = @Translation("Mark as featured"),
 *   type = "node",
 * )
 */
class MarkFeaturedAction extends ActionBase {

  /**
   * {@inheritdoc}
   */
  public function execute($entity = NULL): void {
    /** @var \Drupal\node\NodeInterface $entity */
    if ($entity->hasField('field_featured')) {
      $entity->set('field_featured', TRUE);
      $entity->save();
    }
  }

  /**
   * {@inheritdoc}
   */
  public function access($object, ?AccountInterface $account = NULL, $return_as_object = FALSE) {
    /** @var \Drupal\node\NodeInterface $object */
    $result = $object->access('update', $account, TRUE);
    return $return_as_object ? $result : $result->isAllowed();
  }

}
```

### Drupal 11+ (Attribute)

```php
<?php

namespace Drupal\my_module\Plugin\Action;

use Drupal\Core\Action\ActionBase;
use Drupal\Core\Action\Attribute\Action;
use Drupal\Core\Session\AccountInterface;
use Drupal\Core\StringTranslation\TranslatableMarkup;

#[Action(
  id: 'my_module_mark_featured',
  label: new TranslatableMarkup('Mark as featured'),
  type: 'node',
)]
class MarkFeaturedAction extends ActionBase {

  // Methods are identical to the annotation version.

}
```

### Configurable Action

For actions that require user input (e.g., choosing a value to set), extend `ConfigurableActionBase`:

```php
<?php

namespace Drupal\my_module\Plugin\Action;

use Drupal\Core\Action\Attribute\Action;
use Drupal\Core\Action\ConfigurableActionBase;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\Session\AccountInterface;
use Drupal\Core\StringTranslation\TranslatableMarkup;

#[Action(
  id: 'my_module_set_priority',
  label: new TranslatableMarkup('Set priority level'),
  type: 'node',
)]
class SetPriorityAction extends ConfigurableActionBase {

  /**
   * {@inheritdoc}
   */
  public function defaultConfiguration(): array {
    return ['priority' => 'normal'];
  }

  /**
   * {@inheritdoc}
   */
  public function buildConfigurationForm(array $form, FormStateInterface $form_state): array {
    $form['priority'] = [
      '#type' => 'select',
      '#title' => $this->t('Priority'),
      '#options' => [
        'low' => $this->t('Low'),
        'normal' => $this->t('Normal'),
        'high' => $this->t('High'),
      ],
      '#default_value' => $this->configuration['priority'],
    ];
    return $form;
  }

  /**
   * {@inheritdoc}
   */
  public function submitConfigurationForm(array &$form, FormStateInterface $form_state): void {
    $this->configuration['priority'] = $form_state->getValue('priority');
  }

  /**
   * {@inheritdoc}
   */
  public function execute($entity = NULL): void {
    /** @var \Drupal\node\NodeInterface $entity */
    if ($entity->hasField('field_priority')) {
      $entity->set('field_priority', $this->configuration['priority']);
      $entity->save();
    }
  }

  /**
   * {@inheritdoc}
   */
  public function access($object, ?AccountInterface $account = NULL, $return_as_object = FALSE) {
    $result = $object->access('update', $account, TRUE);
    return $return_as_object ? $result : $result->isAllowed();
  }

}
```

---

## QueueWorker Plugins

QueueWorker plugins process items from a named queue. They are typically triggered by cron but can also be processed manually.

### Drupal 10 (Annotation)

```php
<?php

namespace Drupal\my_module\Plugin\QueueWorker;

use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Logger\LoggerChannelInterface;
use Drupal\Core\Plugin\ContainerFactoryPluginInterface;
use Drupal\Core\Queue\QueueWorkerBase;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Processes email notification queue items.
 *
 * @QueueWorker(
 *   id = "my_module_email_notifications",
 *   title = @Translation("Email notification processor"),
 *   cron = {"time" = 30}
 * )
 */
class EmailNotificationWorker extends QueueWorkerBase implements ContainerFactoryPluginInterface {

  public function __construct(
    array $configuration,
    $plugin_id,
    $plugin_definition,
    protected EntityTypeManagerInterface $entityTypeManager,
    protected LoggerChannelInterface $logger,
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
      $container->get('logger.channel.my_module'),
    );
  }

  /**
   * {@inheritdoc}
   */
  public function processItem($data): void {
    // $data is whatever was added to the queue.
    if (empty($data['user_id']) || empty($data['message'])) {
      $this->logger->error('Invalid queue item data.');
      return;
    }

    $user = $this->entityTypeManager->getStorage('user')->load($data['user_id']);
    if (!$user) {
      $this->logger->warning('User @uid not found.', ['@uid' => $data['user_id']]);
      return;
    }

    // Send the email notification.
    // If processing fails, throw an exception to requeue the item.
    // throw new \Exception('Temporary failure, requeue.');
    $this->logger->info('Processed notification for user @uid.', ['@uid' => $data['user_id']]);
  }

}
```

### Drupal 11+ (Attribute)

```php
<?php

namespace Drupal\my_module\Plugin\QueueWorker;

use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Logger\LoggerChannelInterface;
use Drupal\Core\Plugin\ContainerFactoryPluginInterface;
use Drupal\Core\Queue\Attribute\QueueWorker;
use Drupal\Core\Queue\QueueWorkerBase;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use Symfony\Component\DependencyInjection\ContainerInterface;

#[QueueWorker(
  id: 'my_module_email_notifications',
  title: new TranslatableMarkup('Email notification processor'),
  cron: ['time' => 30],
)]
class EmailNotificationWorker extends QueueWorkerBase implements ContainerFactoryPluginInterface {

  // Methods are identical to the annotation version.

}
```

### Adding Items to a Queue

Queue items are added from other parts of your code, often from hooks, form submissions, or services:

```php
/** @var \Drupal\Core\Queue\QueueFactory $queue_factory */
$queue_factory = \Drupal::service('queue');
$queue = $queue_factory->get('my_module_email_notifications');

$queue->createItem([
  'user_id' => $user->id(),
  'message' => 'Your content has been published.',
  'timestamp' => \Drupal::time()->getRequestTime(),
]);
```

### Processing Queues Manually

```php
$queue_manager = \Drupal::service('plugin.manager.queue_worker');
$queue_factory = \Drupal::service('queue');

$queue_worker = $queue_manager->createInstance('my_module_email_notifications');
$queue = $queue_factory->get('my_module_email_notifications');

while ($item = $queue->claimItem()) {
  try {
    $queue_worker->processItem($item->data);
    $queue->deleteItem($item);
  }
  catch (\Exception $e) {
    $queue->releaseItem($item);
  }
}
```

---

## Key Points

- The `cron.time` setting (in seconds) limits how long cron spends processing the queue per run.
- If `processItem()` throws an exception, the item is released back to the queue for retry.
- If `processItem()` throws `\Drupal\Core\Queue\RequeueException`, the item is immediately requeued without counting as a failure.
- If `processItem()` throws `\Drupal\Core\Queue\SuspendQueueException`, cron stops processing this queue for the current run.
- Action plugins with `type` set will only appear for that entity type in Views bulk operations.
- Actions without a `type` are generic and can be used anywhere.
