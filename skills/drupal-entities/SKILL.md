---
name: drupal-entities
description: Use when creating custom Drupal entity types (content entities or configuration entities), defining base fields, entity handlers, entity forms, entity list builders, or working with entity queries.
version: 1.0.0
---

# Drupal Entities

## Overview

Drupal's Entity API is the backbone of its data model. Every piece of structured
data — nodes, users, taxonomy terms, custom records — is an entity. When you
build a custom module you will almost always define at least one entity type.

There are two kinds:

| Aspect | Content Entity | Config Entity |
|---|---|---|
| Storage | Database tables | YAML config files |
| Revisionable | Yes (optional) | No |
| Translatable | Yes (optional) | Yes (limited) |
| Fieldable | Yes (via Field UI) | No |
| Exportable | Only via migrate / default content | Yes (config export) |
| Use when | Storing user-generated or runtime data | Storing site-builder settings, reusable configuration objects |

**Examples of content entities:** Node, User, Comment, Media, custom records.
**Examples of config entities:** View, ImageStyle, Role, Vocabulary, custom type definitions.

---

## Content Entity — Complete Example

The example below defines a `Task` content entity in a module called
`task_tracker`.

### 1. Entity Type Definition

#### Drupal 10 — Annotation

```php
<?php

declare(strict_types=1);

namespace Drupal\task_tracker\Entity;

use Drupal\Core\Entity\ContentEntityBase;
use Drupal\Core\Entity\ContentEntityInterface;
use Drupal\Core\Entity\EntityChangedInterface;
use Drupal\Core\Entity\EntityChangedTrait;
use Drupal\Core\Entity\EntityTypeInterface;
use Drupal\Core\Field\BaseFieldDefinition;
use Drupal\user\EntityOwnerInterface;
use Drupal\user\EntityOwnerTrait;

/**
 * Defines the Task entity.
 *
 * @ContentEntityType(
 *   id = "task",
 *   label = @Translation("Task"),
 *   label_collection = @Translation("Tasks"),
 *   label_singular = @Translation("task"),
 *   label_plural = @Translation("tasks"),
 *   handlers = {
 *     "form" = {
 *       "add" = "Drupal\task_tracker\Form\TaskForm",
 *       "edit" = "Drupal\task_tracker\Form\TaskForm",
 *       "delete" = "Drupal\Core\Entity\ContentEntityDeleteForm",
 *     },
 *     "list_builder" = "Drupal\task_tracker\TaskListBuilder",
 *     "access" = "Drupal\task_tracker\TaskAccessControlHandler",
 *     "view_builder" = "Drupal\Core\Entity\EntityViewBuilder",
 *     "views_data" = "Drupal\views\EntityViewsData",
 *     "route_provider" = {
 *       "html" = "Drupal\Core\Entity\Routing\AdminHtmlRouteProvider",
 *     },
 *   },
 *   base_table = "task",
 *   admin_permission = "administer tasks",
 *   entity_keys = {
 *     "id" = "id",
 *     "uuid" = "uuid",
 *     "label" = "title",
 *     "owner" = "uid",
 *   },
 *   links = {
 *     "canonical" = "/task/{task}",
 *     "add-form" = "/task/add",
 *     "edit-form" = "/task/{task}/edit",
 *     "delete-form" = "/task/{task}/delete",
 *     "collection" = "/admin/content/tasks",
 *   },
 * )
 */
class Task extends ContentEntityBase implements ContentEntityInterface, EntityChangedInterface, EntityOwnerInterface {

  use EntityChangedTrait;
  use EntityOwnerTrait;

  // ... (base field definitions below)

}
```

#### Drupal 11+ — PHP Attribute

```php
<?php

declare(strict_types=1);

namespace Drupal\task_tracker\Entity;

use Drupal\Core\Entity\Attribute\ContentEntityType;
use Drupal\Core\Entity\ContentEntityBase;
use Drupal\Core\Entity\ContentEntityInterface;
use Drupal\Core\Entity\EntityChangedInterface;
use Drupal\Core\Entity\EntityChangedTrait;
use Drupal\Core\Entity\EntityTypeInterface;
use Drupal\Core\Entity\Routing\AdminHtmlRouteProvider;
use Drupal\Core\Field\BaseFieldDefinition;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use Drupal\task_tracker\Form\TaskForm;
use Drupal\task_tracker\TaskAccessControlHandler;
use Drupal\task_tracker\TaskListBuilder;
use Drupal\user\EntityOwnerInterface;
use Drupal\user\EntityOwnerTrait;
use Drupal\views\EntityViewsData;

#[ContentEntityType(
  id: 'task',
  label: new TranslatableMarkup('Task'),
  label_collection: new TranslatableMarkup('Tasks'),
  label_singular: new TranslatableMarkup('task'),
  label_plural: new TranslatableMarkup('tasks'),
  handlers: [
    'form' => [
      'add' => TaskForm::class,
      'edit' => TaskForm::class,
      'delete' => 'Drupal\Core\Entity\ContentEntityDeleteForm',
    ],
    'list_builder' => TaskListBuilder::class,
    'access' => TaskAccessControlHandler::class,
    'view_builder' => 'Drupal\Core\Entity\EntityViewBuilder',
    'views_data' => EntityViewsData::class,
    'route_provider' => [
      'html' => AdminHtmlRouteProvider::class,
    ],
  ],
  base_table: 'task',
  admin_permission: 'administer tasks',
  entity_keys: [
    'id' => 'id',
    'uuid' => 'uuid',
    'label' => 'title',
    'owner' => 'uid',
  ],
  links: [
    'canonical' => '/task/{task}',
    'add-form' => '/task/add',
    'edit-form' => '/task/{task}/edit',
    'delete-form' => '/task/{task}/delete',
    'collection' => '/admin/content/tasks',
  ],
)]
class Task extends ContentEntityBase implements ContentEntityInterface, EntityChangedInterface, EntityOwnerInterface {

  use EntityChangedTrait;
  use EntityOwnerTrait;

  // ... (base field definitions below)

}
```

### 2. Base Field Definitions

Add this method to the `Task` class:

```php
  /**
   * {@inheritdoc}
   */
  public static function baseFieldDefinitions(EntityTypeInterface $entity_type): array {
    $fields = parent::baseFieldDefinitions($entity_type);

    // Owner (user reference) — provided by EntityOwnerTrait.
    $fields += static::ownerBaseFieldDefinitions($entity_type);

    // Title — string field.
    $fields['title'] = BaseFieldDefinition::create('string')
      ->setLabel(new TranslatableMarkup('Title'))
      ->setRequired(TRUE)
      ->setSetting('max_length', 255)
      ->setDisplayOptions('form', [
        'type' => 'string_textfield',
        'weight' => 0,
      ])
      ->setDisplayConfigurable('form', TRUE)
      ->setDisplayOptions('view', [
        'label' => 'hidden',
        'type' => 'string',
        'weight' => 0,
      ])
      ->setDisplayConfigurable('view', TRUE);

    // Status — boolean field.
    $fields['status'] = BaseFieldDefinition::create('boolean')
      ->setLabel(new TranslatableMarkup('Completed'))
      ->setDefaultValue(FALSE)
      ->setDisplayOptions('form', [
        'type' => 'boolean_checkbox',
        'weight' => 5,
      ])
      ->setDisplayConfigurable('form', TRUE)
      ->setDisplayConfigurable('view', TRUE);

    // Assignee — entity reference to user.
    $fields['assignee'] = BaseFieldDefinition::create('entity_reference')
      ->setLabel(new TranslatableMarkup('Assignee'))
      ->setSetting('target_type', 'user')
      ->setDisplayOptions('form', [
        'type' => 'entity_reference_autocomplete',
        'weight' => 10,
      ])
      ->setDisplayConfigurable('form', TRUE)
      ->setDisplayConfigurable('view', TRUE);

    // Created timestamp.
    $fields['created'] = BaseFieldDefinition::create('created')
      ->setLabel(new TranslatableMarkup('Created'))
      ->setDescription(new TranslatableMarkup('The time the task was created.'));

    // Changed timestamp.
    $fields['changed'] = BaseFieldDefinition::create('changed')
      ->setLabel(new TranslatableMarkup('Changed'))
      ->setDescription(new TranslatableMarkup('The time the task was last updated.'));

    return $fields;
  }
```

### 3. Entity Form

```php
<?php

declare(strict_types=1);

namespace Drupal\task_tracker\Form;

use Drupal\Core\Entity\ContentEntityForm;
use Drupal\Core\Form\FormStateInterface;

/**
 * Form handler for the Task add/edit forms.
 */
class TaskForm extends ContentEntityForm {

  /**
   * {@inheritdoc}
   */
  public function save(array $form, FormStateInterface $form_state): int {
    $result = parent::save($form, $form_state);

    $message_args = ['%label' => $this->entity->toLink()->toString()];
    $message = match ($result) {
      SAVED_NEW => $this->t('Created new task %label.', $message_args),
      default => $this->t('Updated task %label.', $message_args),
    };
    $this->messenger()->addStatus($message);

    $form_state->setRedirectUrl($this->entity->toUrl('collection'));

    return $result;
  }

}
```

### 4. List Builder

```php
<?php

declare(strict_types=1);

namespace Drupal\task_tracker;

use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Entity\EntityListBuilder;

/**
 * Provides a list builder for the Task entity.
 */
class TaskListBuilder extends EntityListBuilder {

  /**
   * {@inheritdoc}
   */
  public function buildHeader(): array {
    $header['id'] = $this->t('ID');
    $header['title'] = $this->t('Title');
    $header['status'] = $this->t('Completed');
    $header['uid'] = $this->t('Author');
    return $header + parent::buildHeader();
  }

  /**
   * {@inheritdoc}
   */
  public function buildRow(EntityInterface $entity): array {
    /** @var \Drupal\task_tracker\Entity\Task $entity */
    $row['id'] = $entity->id();
    $row['title'] = $entity->toLink();
    $row['status'] = $entity->get('status')->value ? $this->t('Yes') : $this->t('No');
    $row['uid']['data'] = [
      '#theme' => 'username',
      '#account' => $entity->getOwner(),
    ];
    return $row + parent::buildRow($entity);
  }

}
```

### 5. Access Control Handler

```php
<?php

declare(strict_types=1);

namespace Drupal\task_tracker;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Entity\EntityAccessControlHandler;
use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Session\AccountInterface;

/**
 * Defines the access control handler for the Task entity.
 */
class TaskAccessControlHandler extends EntityAccessControlHandler {

  /**
   * {@inheritdoc}
   */
  protected function checkAccess(EntityInterface $entity, $operation, AccountInterface $account): AccessResult {
    return match ($operation) {
      'view' => AccessResult::allowedIfHasPermission($account, 'view task'),
      'update' => AccessResult::allowedIfHasPermission($account, 'edit task'),
      'delete' => AccessResult::allowedIfHasPermission($account, 'delete task'),
      default => AccessResult::neutral(),
    };
  }

  /**
   * {@inheritdoc}
   */
  protected function checkCreateAccess(AccountInterface $account, array $context, $entity_bundle = NULL): AccessResult {
    return AccessResult::allowedIfHasPermission($account, 'create task');
  }

}
```

### 6. Permissions File

`task_tracker.permissions.yml`:

```yaml
administer tasks:
  title: 'Administer tasks'
  restrict access: true
create task:
  title: 'Create task'
view task:
  title: 'View task'
edit task:
  title: 'Edit task'
delete task:
  title: 'Delete task'
```

---

## Config Entity — Complete Example

A config entity stores settings that are exported with `drush config:export`.
The example below defines a `TaskType` config entity (a bundle provider for
`Task`).

### 1. Entity Type Definition (D11+ Attribute)

```php
<?php

declare(strict_types=1);

namespace Drupal\task_tracker\Entity;

use Drupal\Core\Config\Entity\ConfigEntityBundleBase;
use Drupal\Core\Entity\Attribute\ConfigEntityType;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use Drupal\task_tracker\Form\TaskTypeForm;
use Drupal\task_tracker\TaskTypeListBuilder;

#[ConfigEntityType(
  id: 'task_type',
  label: new TranslatableMarkup('Task type'),
  label_collection: new TranslatableMarkup('Task types'),
  handlers: [
    'form' => [
      'add' => TaskTypeForm::class,
      'edit' => TaskTypeForm::class,
      'delete' => 'Drupal\Core\Entity\EntityDeleteForm',
    ],
    'list_builder' => TaskTypeListBuilder::class,
    'route_provider' => [
      'html' => 'Drupal\Core\Entity\Routing\AdminHtmlRouteProvider',
    ],
  ],
  config_prefix: 'task_type',
  admin_permission: 'administer tasks',
  bundle_of: 'task',
  entity_keys: [
    'id' => 'id',
    'label' => 'label',
    'uuid' => 'uuid',
  ],
  links: [
    'add-form' => '/admin/structure/task-types/add',
    'edit-form' => '/admin/structure/task-types/{task_type}/edit',
    'delete-form' => '/admin/structure/task-types/{task_type}/delete',
    'collection' => '/admin/structure/task-types',
  ],
  config_export: [
    'id',
    'label',
    'description',
  ],
)]
class TaskType extends ConfigEntityBundleBase {

  /**
   * The machine name.
   */
  protected string $id;

  /**
   * The human-readable name.
   */
  protected string $label;

  /**
   * A brief description.
   */
  protected string $description = '';

  /**
   * Gets the description.
   */
  public function getDescription(): string {
    return $this->description;
  }

}
```

### 2. Config Schema

`config/schema/task_tracker.schema.yml`:

```yaml
task_tracker.task_type.*:
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

### 3. Config Entity Form

```php
<?php

declare(strict_types=1);

namespace Drupal\task_tracker\Form;

use Drupal\Core\Entity\EntityForm;
use Drupal\Core\Form\FormStateInterface;

/**
 * Form handler for the TaskType add/edit forms.
 */
class TaskTypeForm extends EntityForm {

  /**
   * {@inheritdoc}
   */
  public function form(array $form, FormStateInterface $form_state): array {
    $form = parent::form($form, $form_state);

    /** @var \Drupal\task_tracker\Entity\TaskType $entity */
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
        'exists' => '\Drupal\task_tracker\Entity\TaskType::load',
      ],
      '#disabled' => !$entity->isNew(),
    ];

    $form['description'] = [
      '#type' => 'textarea',
      '#title' => $this->t('Description'),
      '#default_value' => $entity->getDescription(),
    ];

    return $form;
  }

  /**
   * {@inheritdoc}
   */
  public function save(array $form, FormStateInterface $form_state): int {
    $result = parent::save($form, $form_state);

    $message_args = ['%label' => $this->entity->label()];
    $message = match ($result) {
      SAVED_NEW => $this->t('Created task type %label.', $message_args),
      default => $this->t('Updated task type %label.', $message_args),
    };
    $this->messenger()->addStatus($message);

    $form_state->setRedirectUrl($this->entity->toUrl('collection'));

    return $result;
  }

}
```

### 4. Config Entity List Builder

```php
<?php

declare(strict_types=1);

namespace Drupal\task_tracker;

use Drupal\Core\Config\Entity\ConfigEntityListBuilder;
use Drupal\Core\Entity\EntityInterface;

/**
 * Provides a list builder for the TaskType config entity.
 */
class TaskTypeListBuilder extends ConfigEntityListBuilder {

  /**
   * {@inheritdoc}
   */
  public function buildHeader(): array {
    $header['label'] = $this->t('Label');
    $header['id'] = $this->t('Machine name');
    $header['description'] = $this->t('Description');
    return $header + parent::buildHeader();
  }

  /**
   * {@inheritdoc}
   */
  public function buildRow(EntityInterface $entity): array {
    /** @var \Drupal\task_tracker\Entity\TaskType $entity */
    $row['label'] = $entity->label();
    $row['id'] = $entity->id();
    $row['description'] = $entity->getDescription();
    return $row + parent::buildRow($entity);
  }

}
```

---

## Entity Queries

### Using entityQuery()

```php
// Simple query.
$ids = \Drupal::entityQuery('task')
  ->condition('status', TRUE)
  ->condition('assignee', $current_user_id)
  ->sort('created', 'DESC')
  ->range(0, 10)
  ->accessCheck(TRUE)
  ->execute();

// Load entities from IDs.
$tasks = \Drupal::entityTypeManager()
  ->getStorage('task')
  ->loadMultiple($ids);
```

### Injected Service (recommended)

```php
<?php

declare(strict_types=1);

namespace Drupal\task_tracker\Service;

use Drupal\Core\Entity\EntityTypeManagerInterface;

class TaskRepository {

  public function __construct(
    protected readonly EntityTypeManagerInterface $entityTypeManager,
  ) {}

  /**
   * Loads open tasks assigned to a user.
   *
   * @return \Drupal\task_tracker\Entity\Task[]
   */
  public function findOpenTasksByUser(int $uid): array {
    $ids = $this->entityTypeManager
      ->getStorage('task')
      ->getQuery()
      ->condition('status', FALSE)
      ->condition('assignee', $uid)
      ->accessCheck(TRUE)
      ->execute();

    return $this->entityTypeManager
      ->getStorage('task')
      ->loadMultiple($ids);
  }

}
```

### Common Query Conditions

```php
// Equals.
->condition('field_name', $value)

// Not equals.
->condition('field_name', $value, '<>')

// IN a list.
->condition('field_name', [1, 2, 3], 'IN')

// NULL check.
->notExists('field_name')
->exists('field_name')

// Date range.
->condition('created', $timestamp, '>=')
->condition('created', $end_timestamp, '<')

// Entity reference (sub-property).
->condition('field_tags.target_id', $tid)

// OR conditions.
$or = $query->orConditionGroup();
$or->condition('status', TRUE);
$or->condition('uid', $current_user_id);
$query->condition($or);

// Count.
$count = \Drupal::entityQuery('task')
  ->condition('status', TRUE)
  ->accessCheck(TRUE)
  ->count()
  ->execute();
```

---

## Scaffolding with Drush

Drush's code generator can scaffold entire entity types for you. Always run
these inside your DDEV environment.

### Content Entity

```bash
ddev drush gen entity:content
```

The interactive wizard will prompt for module name, entity class, base table,
revision support, translation support, and more. This generates:

- Entity class with annotation and `baseFieldDefinitions()`
- Entity form, delete form
- List builder
- Access control handler
- Permissions YAML
- Route configuration (via `AdminHtmlRouteProvider`)
- Install / update hooks for the database schema

### Config Entity

```bash
ddev drush gen entity:config
```

Generates:

- Config entity class with annotation
- Config schema YAML
- Entity form and list builder
- Route provider links

### Useful Follow-up Commands

```bash
# After creating or modifying entity definitions:
ddev drush entity:updates        # Apply pending entity schema updates
ddev drush cache:rebuild         # Rebuild caches so new entity types are discovered

# Export config for config entities:
ddev drush config:export         # Export active config to sync directory
```

---

## Key Points to Remember

1. **Always call `->accessCheck(TRUE)` (or `FALSE`)** on entity queries.
   Drupal 10+ throws a deprecation notice, and Drupal 11 throws an exception,
   if you omit it.

2. **Use `EntityTypeManagerInterface`** via dependency injection rather than
   the static `\Drupal::entityTypeManager()` helper whenever possible.

3. **Content entities need an install schema.** When you enable the module,
   Drupal will create the base table automatically from `baseFieldDefinitions()`.
   If you add fields later, you need `hook_update_N()` or
   `drush entity:updates`.

4. **Config entities need a schema file.** Without
   `config/schema/module.schema.yml`, config export/import will fail validation.

5. **The `links` array** controls route generation when using
   `AdminHtmlRouteProvider`. Ensure the path tokens match the entity type ID
   (e.g., `{task}` for entity type `task`).

6. **Annotations (D10) vs Attributes (D11+):** Annotations still work in
   Drupal 11, but attributes are the forward-compatible approach. New code
   targeting Drupal 11+ should use attributes.

---

## Related Skills

- **drupal-forms** — Building custom forms beyond entity forms.
- **drupal-routing** — Manual route definitions and controllers.
- **drupal-services** — Dependency injection and service definitions.
- **drupal-database** — Direct database queries when entity queries are not sufficient.
- **drupal-plugins** — Plugin types, which share annotation/attribute patterns with entities.
- **drupal-config** — Configuration management, import/export.
- **drush-generate** — Full reference for all Drush code generators.
