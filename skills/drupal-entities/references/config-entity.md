# Config Entity — Deep Dive

This reference covers advanced config entity patterns: full schema definition,
exportable configuration, entity operations, dependency management, and
third-party settings.

---

## Full Config Entity Example

```php
<?php

declare(strict_types=1);

namespace Drupal\example\Entity;

use Drupal\Core\Config\Entity\ConfigEntityBase;
use Drupal\Core\Entity\Attribute\ConfigEntityType;
use Drupal\Core\Entity\EntityWithPluginCollectionInterface;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use Drupal\example\Form\NotificationRuleForm;
use Drupal\example\NotificationRuleListBuilder;

#[ConfigEntityType(
  id: 'notification_rule',
  label: new TranslatableMarkup('Notification rule'),
  label_collection: new TranslatableMarkup('Notification rules'),
  label_singular: new TranslatableMarkup('notification rule'),
  label_plural: new TranslatableMarkup('notification rules'),
  handlers: [
    'form' => [
      'add' => NotificationRuleForm::class,
      'edit' => NotificationRuleForm::class,
      'delete' => 'Drupal\Core\Entity\EntityDeleteForm',
    ],
    'list_builder' => NotificationRuleListBuilder::class,
    'route_provider' => [
      'html' => 'Drupal\Core\Entity\Routing\AdminHtmlRouteProvider',
    ],
  ],
  config_prefix: 'notification_rule',
  admin_permission: 'administer notification rules',
  entity_keys: [
    'id' => 'id',
    'label' => 'label',
    'uuid' => 'uuid',
    'status' => 'status',
    'weight' => 'weight',
  ],
  links: [
    'add-form' => '/admin/config/system/notification-rules/add',
    'edit-form' => '/admin/config/system/notification-rules/{notification_rule}/edit',
    'delete-form' => '/admin/config/system/notification-rules/{notification_rule}/delete',
    'enable' => '/admin/config/system/notification-rules/{notification_rule}/enable',
    'disable' => '/admin/config/system/notification-rules/{notification_rule}/disable',
    'collection' => '/admin/config/system/notification-rules',
  ],
  config_export: [
    'id',
    'label',
    'description',
    'status',
    'weight',
    'event',
    'conditions',
    'channels',
  ],
)]
class NotificationRule extends ConfigEntityBase {

  /**
   * The machine name.
   */
  protected string $id;

  /**
   * The human-readable label.
   */
  protected string $label;

  /**
   * A description of this rule.
   */
  protected string $description = '';

  /**
   * Whether this rule is enabled.
   */
  protected bool $status = TRUE;

  /**
   * Sort weight.
   */
  protected int $weight = 0;

  /**
   * The triggering event name.
   */
  protected string $event = '';

  /**
   * An array of condition configuration.
   */
  protected array $conditions = [];

  /**
   * An array of notification channel IDs.
   */
  protected array $channels = [];

  /**
   * Gets the description.
   */
  public function getDescription(): string {
    return $this->description;
  }

  /**
   * Gets the event.
   */
  public function getEvent(): string {
    return $this->event;
  }

  /**
   * Gets the conditions.
   */
  public function getConditions(): array {
    return $this->conditions;
  }

  /**
   * Gets the channels.
   */
  public function getChannels(): array {
    return $this->channels;
  }

  /**
   * Sets the conditions.
   */
  public function setConditions(array $conditions): static {
    $this->conditions = $conditions;
    return $this;
  }

}
```

---

## Full Config Schema

`config/schema/example.schema.yml`:

```yaml
example.notification_rule.*:
  type: config_entity
  label: 'Notification rule'
  mapping:
    id:
      type: string
      label: 'Machine name'
    label:
      type: label
      label: 'Label'
    uuid:
      type: string
      label: 'UUID'
    description:
      type: text
      label: 'Description'
    status:
      type: boolean
      label: 'Enabled'
    weight:
      type: integer
      label: 'Weight'
    event:
      type: string
      label: 'Event name'
    conditions:
      type: sequence
      label: 'Conditions'
      sequence:
        type: mapping
        label: 'Condition'
        mapping:
          plugin:
            type: string
            label: 'Plugin ID'
          settings:
            type: mapping
            label: 'Settings'
    channels:
      type: sequence
      label: 'Channels'
      sequence:
        type: string
        label: 'Channel ID'
```

### Schema Type Reference

| Schema type | PHP type | Use for |
|---|---|---|
| `string` | string | Machine names, IDs |
| `label` | string | Human-readable translatable labels |
| `text` | string | Longer translatable text |
| `boolean` | bool | Flags |
| `integer` | int | Whole numbers |
| `float` | float | Decimal numbers |
| `uri` | string | URIs |
| `email` | string | Email addresses |
| `mapping` | array | Associative arrays with known keys |
| `sequence` | array | Indexed arrays / lists |
| `config_entity` | — | Base type for config entities (includes UUID, langcode) |

---

## Exportable Config — Default Configuration

Ship default config entities with your module by placing YAML files in:

```
modules/example/config/install/example.notification_rule.daily_digest.yml
```

```yaml
id: daily_digest
label: 'Daily digest'
description: 'Sends a daily summary of activity.'
status: true
weight: 0
event: cron
conditions: []
channels:
  - email
langcode: en
```

When the module is installed, this config entity is imported automatically.

### Optional Configuration

Place config that should only be imported when a dependency is present in:

```
modules/example/config/optional/example.notification_rule.slack_alert.yml
```

Add a `dependencies` key to declare the requirement:

```yaml
id: slack_alert
label: 'Slack alert'
description: 'Sends alerts to Slack.'
status: true
weight: 10
event: entity_insert
conditions: []
channels:
  - slack
langcode: en
dependencies:
  module:
    - slack_integration
```

---

## Config Entity Operations

The default list builder provides edit/delete operations. Add custom operations
(enable/disable, duplicate) by overriding the list builder:

```php
<?php

declare(strict_types=1);

namespace Drupal\example;

use Drupal\Core\Config\Entity\ConfigEntityListBuilder;
use Drupal\Core\Entity\EntityInterface;

/**
 * List builder for notification rules with custom operations.
 */
class NotificationRuleListBuilder extends ConfigEntityListBuilder {

  /**
   * {@inheritdoc}
   */
  public function buildHeader(): array {
    $header['label'] = $this->t('Label');
    $header['event'] = $this->t('Event');
    $header['status'] = $this->t('Status');
    $header['weight'] = $this->t('Weight');
    return $header + parent::buildHeader();
  }

  /**
   * {@inheritdoc}
   */
  public function buildRow(EntityInterface $entity): array {
    /** @var \Drupal\example\Entity\NotificationRule $entity */
    $row['label'] = $entity->label();
    $row['event'] = $entity->getEvent();
    $row['status'] = $entity->status() ? $this->t('Enabled') : $this->t('Disabled');
    $row['weight'] = $entity->get('weight');
    return $row + parent::buildRow($entity);
  }

  /**
   * {@inheritdoc}
   */
  public function getDefaultOperations(EntityInterface $entity): array {
    $operations = parent::getDefaultOperations($entity);

    // Enable / Disable toggle.
    if ($entity->status()) {
      $operations['disable'] = [
        'title' => $this->t('Disable'),
        'url' => $entity->toUrl('disable'),
        'weight' => 50,
      ];
    }
    else {
      $operations['enable'] = [
        'title' => $this->t('Enable'),
        'url' => $entity->toUrl('enable'),
        'weight' => 50,
      ];
    }

    return $operations;
  }

}
```

---

## Dependency Management

Config entities can declare dependencies on modules, config, or content. Drupal
uses these to prevent orphaned config during module uninstall.

### Automatic Dependencies

Drupal calculates dependencies automatically from entity reference fields and
plugin references. You can also declare them explicitly by overriding
`calculateDependencies()`:

```php
/**
 * {@inheritdoc}
 */
public function calculateDependencies(): static {
  parent::calculateDependencies();

  // Depend on the module that provides the event.
  if ($this->event === 'commerce_order_placed') {
    $this->addDependency('module', 'commerce_order');
  }

  return $this;
}
```

### Reacting to Dependency Removal

Override `onDependencyRemoval()` to gracefully handle the removal of a
dependency (e.g., remove a channel if its provider module is uninstalled):

```php
/**
 * {@inheritdoc}
 */
public function onDependencyRemoval(array $dependencies): bool {
  $changed = FALSE;

  // If the slack_integration module is being removed, remove the
  // slack channel from our configuration.
  if (in_array('slack_integration', $dependencies['module'] ?? [])) {
    $this->channels = array_filter(
      $this->channels,
      fn(string $channel): bool => $channel !== 'slack',
    );
    $changed = TRUE;
  }

  return $changed;
}
```

---

## Third-Party Settings

Any config entity that implements `ThirdPartySettingsInterface` (which
`ConfigEntityBase` does) can store arbitrary settings keyed by module name.
This allows other modules to attach data to your config entity without
modifying its schema.

### Storing Third-Party Settings

```php
// In a form alter or custom code:
$entity->setThirdPartySetting('my_module', 'custom_flag', TRUE);
$entity->save();
```

### Reading Third-Party Settings

```php
$flag = $entity->getThirdPartySetting('my_module', 'custom_flag', FALSE);

// Get all settings for a module.
$all = $entity->getThirdPartySettings('my_module');

// Get all modules that have stored settings.
$providers = $entity->getThirdPartyProviders();
```

### Schema for Third-Party Settings

Add to your schema file:

```yaml
example.notification_rule.*.third_party.my_module:
  type: mapping
  label: 'My Module settings'
  mapping:
    custom_flag:
      type: boolean
      label: 'Custom flag'
```

---

## Config Entity Form — Full Example with Validation

```php
<?php

declare(strict_types=1);

namespace Drupal\example\Form;

use Drupal\Core\Entity\EntityForm;
use Drupal\Core\Form\FormStateInterface;

/**
 * Form handler for NotificationRule add/edit.
 */
class NotificationRuleForm extends EntityForm {

  /**
   * {@inheritdoc}
   */
  public function form(array $form, FormStateInterface $form_state): array {
    $form = parent::form($form, $form_state);

    /** @var \Drupal\example\Entity\NotificationRule $entity */
    $entity = $this->entity;

    $form['label'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Label'),
      '#maxlength' => 255,
      '#default_value' => $entity->label(),
      '#required' => TRUE,
    ];

    $form['id'] = [
      '#type' => 'machine_name',
      '#default_value' => $entity->id(),
      '#machine_name' => [
        'exists' => '\Drupal\example\Entity\NotificationRule::load',
      ],
      '#disabled' => !$entity->isNew(),
    ];

    $form['description'] = [
      '#type' => 'textarea',
      '#title' => $this->t('Description'),
      '#default_value' => $entity->getDescription(),
    ];

    $form['event'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Event'),
      '#default_value' => $entity->getEvent(),
      '#required' => TRUE,
      '#description' => $this->t('The event machine name that triggers this rule.'),
    ];

    $form['status'] = [
      '#type' => 'checkbox',
      '#title' => $this->t('Enabled'),
      '#default_value' => $entity->status(),
    ];

    return $form;
  }

  /**
   * {@inheritdoc}
   */
  public function validateForm(array &$form, FormStateInterface $form_state): void {
    parent::validateForm($form, $form_state);

    $event = $form_state->getValue('event');
    if ($event && !preg_match('/^[a-z_]+$/', $event)) {
      $form_state->setErrorByName('event', $this->t('The event name must contain only lowercase letters and underscores.'));
    }
  }

  /**
   * {@inheritdoc}
   */
  public function save(array $form, FormStateInterface $form_state): int {
    $result = parent::save($form, $form_state);

    $message_args = ['%label' => $this->entity->label()];
    $message = match ($result) {
      SAVED_NEW => $this->t('Created notification rule %label.', $message_args),
      default => $this->t('Updated notification rule %label.', $message_args),
    };
    $this->messenger()->addStatus($message);

    $form_state->setRedirectUrl($this->entity->toUrl('collection'));

    return $result;
  }

}
```

---

## Key Differences from Content Entities

| Feature | Content Entity | Config Entity |
|---|---|---|
| Base class | `ContentEntityBase` | `ConfigEntityBase` |
| Fields | `BaseFieldDefinition` in `baseFieldDefinitions()` | PHP class properties |
| Storage | Database tables (`SqlContentEntityStorage`) | Config YAML (`ConfigEntityStorage`) |
| Schema | Auto-generated from field definitions | Manual `config/schema/*.schema.yml` |
| Field UI | Supports configurable fields via Field UI | Not supported |
| Revisions | Supported | Not supported |
| Exportable | Not natively (use Migrate or Default Content) | Yes, via config export/import |
| Query | `\Drupal::entityQuery()` with SQL backend | `\Drupal::entityQuery()` with config backend |
| Performance | Cached, but backed by DB queries | Loaded from cached config — very fast |
| CRUD forms | `ContentEntityForm` (auto-generates from fields) | `EntityForm` (manual form elements) |
