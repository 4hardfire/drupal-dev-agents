# Content Entity — Deep Dive

This reference covers advanced content entity patterns: all common field types,
revision support, translation support, bundle support, custom access handlers,
and custom storage handlers.

---

## Full Content Entity with All Common Field Types

```php
<?php

declare(strict_types=1);

namespace Drupal\example\Entity;

use Drupal\Core\Entity\ContentEntityBase;
use Drupal\Core\Entity\ContentEntityInterface;
use Drupal\Core\Entity\EntityChangedInterface;
use Drupal\Core\Entity\EntityChangedTrait;
use Drupal\Core\Entity\EntityPublishedInterface;
use Drupal\Core\Entity\EntityPublishedTrait;
use Drupal\Core\Entity\EntityTypeInterface;
use Drupal\Core\Entity\RevisionableContentEntityBase;
use Drupal\Core\Entity\RevisionableInterface;
use Drupal\Core\Field\BaseFieldDefinition;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use Drupal\user\EntityOwnerInterface;
use Drupal\user\EntityOwnerTrait;

/**
 * Defines the Project entity with full field coverage.
 *
 * This example extends RevisionableContentEntityBase to get
 * built-in revision support.
 */
class Project extends RevisionableContentEntityBase implements
  ContentEntityInterface,
  EntityChangedInterface,
  EntityOwnerInterface,
  EntityPublishedInterface,
  RevisionableInterface {

  use EntityChangedTrait;
  use EntityOwnerTrait;
  use EntityPublishedTrait;

  /**
   * {@inheritdoc}
   */
  public static function baseFieldDefinitions(EntityTypeInterface $entity_type): array {
    // Pulls in id, uuid, revision_id, langcode from parent.
    $fields = parent::baseFieldDefinitions($entity_type);

    // Owner field from EntityOwnerTrait.
    $fields += static::ownerBaseFieldDefinitions($entity_type);

    // Published field from EntityPublishedTrait.
    $fields += static::publishedBaseFieldDefinitions($entity_type);

    // ---------------------------------------------------------------
    // String field
    // ---------------------------------------------------------------
    $fields['title'] = BaseFieldDefinition::create('string')
      ->setLabel(new TranslatableMarkup('Title'))
      ->setRequired(TRUE)
      ->setRevisionable(TRUE)
      ->setTranslatable(TRUE)
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

    // ---------------------------------------------------------------
    // String (long / plain text, no format)
    // ---------------------------------------------------------------
    $fields['summary'] = BaseFieldDefinition::create('string_long')
      ->setLabel(new TranslatableMarkup('Summary'))
      ->setRevisionable(TRUE)
      ->setTranslatable(TRUE)
      ->setDisplayOptions('form', [
        'type' => 'string_textarea',
        'weight' => 5,
        'settings' => [
          'rows' => 4,
        ],
      ])
      ->setDisplayConfigurable('form', TRUE)
      ->setDisplayConfigurable('view', TRUE);

    // ---------------------------------------------------------------
    // Text (long, with format / filtered markup)
    // ---------------------------------------------------------------
    $fields['description'] = BaseFieldDefinition::create('text_long')
      ->setLabel(new TranslatableMarkup('Description'))
      ->setRevisionable(TRUE)
      ->setTranslatable(TRUE)
      ->setDisplayOptions('form', [
        'type' => 'text_textarea',
        'weight' => 10,
      ])
      ->setDisplayConfigurable('form', TRUE)
      ->setDisplayOptions('view', [
        'label' => 'above',
        'type' => 'text_default',
        'weight' => 10,
      ])
      ->setDisplayConfigurable('view', TRUE);

    // ---------------------------------------------------------------
    // Boolean
    // ---------------------------------------------------------------
    $fields['is_featured'] = BaseFieldDefinition::create('boolean')
      ->setLabel(new TranslatableMarkup('Featured'))
      ->setRevisionable(TRUE)
      ->setDefaultValue(FALSE)
      ->setDisplayOptions('form', [
        'type' => 'boolean_checkbox',
        'weight' => 15,
      ])
      ->setDisplayConfigurable('form', TRUE)
      ->setDisplayConfigurable('view', TRUE);

    // ---------------------------------------------------------------
    // Integer
    // ---------------------------------------------------------------
    $fields['priority'] = BaseFieldDefinition::create('integer')
      ->setLabel(new TranslatableMarkup('Priority'))
      ->setRevisionable(TRUE)
      ->setSetting('min', 1)
      ->setSetting('max', 10)
      ->setDefaultValue(5)
      ->setDisplayOptions('form', [
        'type' => 'number',
        'weight' => 20,
      ])
      ->setDisplayConfigurable('form', TRUE)
      ->setDisplayConfigurable('view', TRUE);

    // ---------------------------------------------------------------
    // Decimal
    // ---------------------------------------------------------------
    $fields['budget'] = BaseFieldDefinition::create('decimal')
      ->setLabel(new TranslatableMarkup('Budget'))
      ->setRevisionable(TRUE)
      ->setSetting('precision', 10)
      ->setSetting('scale', 2)
      ->setDisplayOptions('form', [
        'type' => 'number',
        'weight' => 25,
      ])
      ->setDisplayConfigurable('form', TRUE)
      ->setDisplayConfigurable('view', TRUE);

    // ---------------------------------------------------------------
    // Float
    // ---------------------------------------------------------------
    $fields['completion_rate'] = BaseFieldDefinition::create('float')
      ->setLabel(new TranslatableMarkup('Completion rate'))
      ->setRevisionable(TRUE)
      ->setDisplayOptions('form', [
        'type' => 'number',
        'weight' => 27,
      ])
      ->setDisplayConfigurable('form', TRUE)
      ->setDisplayConfigurable('view', TRUE);

    // ---------------------------------------------------------------
    // Email
    // ---------------------------------------------------------------
    $fields['contact_email'] = BaseFieldDefinition::create('email')
      ->setLabel(new TranslatableMarkup('Contact email'))
      ->setRevisionable(TRUE)
      ->setDisplayOptions('form', [
        'type' => 'email_default',
        'weight' => 30,
      ])
      ->setDisplayConfigurable('form', TRUE)
      ->setDisplayConfigurable('view', TRUE);

    // ---------------------------------------------------------------
    // Link / URI
    // ---------------------------------------------------------------
    $fields['website'] = BaseFieldDefinition::create('link')
      ->setLabel(new TranslatableMarkup('Website'))
      ->setRevisionable(TRUE)
      ->setSetting('link_type', 16) // LinkItemInterface::LINK_EXTERNAL
      ->setDisplayOptions('form', [
        'type' => 'link_default',
        'weight' => 35,
      ])
      ->setDisplayConfigurable('form', TRUE)
      ->setDisplayConfigurable('view', TRUE);

    // ---------------------------------------------------------------
    // Entity reference
    // ---------------------------------------------------------------
    $fields['manager'] = BaseFieldDefinition::create('entity_reference')
      ->setLabel(new TranslatableMarkup('Manager'))
      ->setRevisionable(TRUE)
      ->setSetting('target_type', 'user')
      ->setSetting('handler', 'default')
      ->setDisplayOptions('form', [
        'type' => 'entity_reference_autocomplete',
        'weight' => 40,
      ])
      ->setDisplayConfigurable('form', TRUE)
      ->setDisplayOptions('view', [
        'label' => 'inline',
        'type' => 'entity_reference_label',
        'weight' => 40,
      ])
      ->setDisplayConfigurable('view', TRUE);

    // ---------------------------------------------------------------
    // Entity reference — with handler settings
    // ---------------------------------------------------------------
    $fields['tags'] = BaseFieldDefinition::create('entity_reference')
      ->setLabel(new TranslatableMarkup('Tags'))
      ->setRevisionable(TRUE)
      ->setSetting('target_type', 'taxonomy_term')
      ->setSetting('handler', 'default:taxonomy_term')
      ->setSetting('handler_settings', [
        'target_bundles' => ['tags' => 'tags'],
      ])
      ->setCardinality(BaseFieldDefinition::CARDINALITY_UNLIMITED)
      ->setDisplayOptions('form', [
        'type' => 'entity_reference_autocomplete',
        'weight' => 45,
      ])
      ->setDisplayConfigurable('form', TRUE)
      ->setDisplayConfigurable('view', TRUE);

    // ---------------------------------------------------------------
    // Timestamp
    // ---------------------------------------------------------------
    $fields['deadline'] = BaseFieldDefinition::create('timestamp')
      ->setLabel(new TranslatableMarkup('Deadline'))
      ->setRevisionable(TRUE)
      ->setDisplayOptions('form', [
        'type' => 'datetime_timestamp',
        'weight' => 50,
      ])
      ->setDisplayConfigurable('form', TRUE)
      ->setDisplayOptions('view', [
        'label' => 'inline',
        'type' => 'timestamp',
        'weight' => 50,
      ])
      ->setDisplayConfigurable('view', TRUE);

    // ---------------------------------------------------------------
    // List (string) — allowed values
    // ---------------------------------------------------------------
    $fields['status_label'] = BaseFieldDefinition::create('list_string')
      ->setLabel(new TranslatableMarkup('Status'))
      ->setRevisionable(TRUE)
      ->setSetting('allowed_values', [
        'open' => 'Open',
        'in_progress' => 'In Progress',
        'closed' => 'Closed',
      ])
      ->setDefaultValue('open')
      ->setDisplayOptions('form', [
        'type' => 'options_select',
        'weight' => 55,
      ])
      ->setDisplayConfigurable('form', TRUE)
      ->setDisplayConfigurable('view', TRUE);

    // ---------------------------------------------------------------
    // Created / Changed timestamps
    // ---------------------------------------------------------------
    $fields['created'] = BaseFieldDefinition::create('created')
      ->setLabel(new TranslatableMarkup('Created'))
      ->setDescription(new TranslatableMarkup('The time the entity was created.'));

    $fields['changed'] = BaseFieldDefinition::create('changed')
      ->setLabel(new TranslatableMarkup('Changed'))
      ->setDescription(new TranslatableMarkup('The time the entity was last edited.'));

    return $fields;
  }

}
```

---

## Revision Support

To make a content entity revisionable:

1. Extend `RevisionableContentEntityBase` instead of `ContentEntityBase`.
2. Add revision entity keys and a revision table.
3. Mark fields as `->setRevisionable(TRUE)`.

### Entity Type Keys and Tables

```php
// In the annotation or attribute:
  base_table: 'project',
  revision_table: 'project_revision',
  entity_keys: [
    'id' => 'id',
    'uuid' => 'uuid',
    'revision' => 'revision_id',
    'label' => 'title',
    'owner' => 'uid',
    'published' => 'status',
  ],
  revision_metadata_keys: [
    'revision_user' => 'revision_uid',
    'revision_created' => 'revision_timestamp',
    'revision_log_message' => 'revision_log',
  ],
```

### Links for Revisions

```php
  links: [
    'canonical' => '/project/{project}',
    'add-form' => '/project/add',
    'edit-form' => '/project/{project}/edit',
    'delete-form' => '/project/{project}/delete',
    'collection' => '/admin/content/projects',
    'revision' => '/project/{project}/revision/{project_revision}/view',
    'revision_revert' => '/project/{project}/revision/{project_revision}/revert',
    'revision_delete' => '/project/{project}/revision/{project_revision}/delete',
    'version-history' => '/project/{project}/revisions',
  ],
```

---

## Translation Support

To make a content entity translatable:

1. Add `translatable: TRUE` to the entity type definition.
2. Add a `data_table` and optionally a `revision_data_table`.
3. Include `langcode` in entity keys.
4. Mark translatable fields with `->setTranslatable(TRUE)`.

```php
// In the annotation or attribute:
  translatable: TRUE,
  data_table: 'project_field_data',
  revision_data_table: 'project_field_revision',
  entity_keys: [
    'id' => 'id',
    'uuid' => 'uuid',
    'revision' => 'revision_id',
    'langcode' => 'langcode',
    'label' => 'title',
    'owner' => 'uid',
    'published' => 'status',
  ],
```

### Working with Translations in Code

```php
// Check if a translation exists.
if ($entity->hasTranslation('fr')) {
  $french = $entity->getTranslation('fr');
  $title = $french->get('title')->value;
}

// Add a translation.
$translation = $entity->addTranslation('de', [
  'title' => 'Projekt Titel',
]);
$translation->save();
```

---

## Bundle Support

Bundles let you create subtypes of an entity (like content types for nodes).
The bundle is provided by a config entity.

### Content Entity Side

```php
// In the annotation or attribute, add:
  bundle_label: new TranslatableMarkup('Project type'),
  bundle_entity_type: 'project_type',
  entity_keys: [
    'id' => 'id',
    'uuid' => 'uuid',
    'bundle' => 'type',
    'label' => 'title',
    // ...
  ],
  links: [
    // ...
    'add-page' => '/project/add',
    'add-form' => '/project/add/{project_type}',
  ],
```

### Bundle Field

```php
// The bundle field is automatically created by the entity system.
// You do NOT need to define it in baseFieldDefinitions(), but you
// can customize its display:

$fields['type'] = BaseFieldDefinition::create('entity_reference')
  ->setLabel(new TranslatableMarkup('Type'))
  ->setSetting('target_type', 'project_type')
  ->setReadOnly(TRUE);
```

### Per-Bundle Fields

Implement `bundleFieldDefinitions()` to add fields that only appear on
specific bundles:

```php
/**
 * {@inheritdoc}
 */
public static function bundleFieldDefinitions(
  EntityTypeInterface $entity_type,
  $bundle,
  array $base_field_definitions,
): array {
  $fields = [];

  if ($bundle === 'research') {
    $fields['lab_code'] = BaseFieldDefinition::create('string')
      ->setLabel(new TranslatableMarkup('Lab code'))
      ->setRequired(TRUE)
      ->setTargetBundle($bundle)
      ->setDisplayOptions('form', [
        'type' => 'string_textfield',
        'weight' => 5,
      ])
      ->setDisplayConfigurable('form', TRUE)
      ->setDisplayConfigurable('view', TRUE);
  }

  return $fields;
}
```

---

## Custom Access Handler — Advanced

```php
<?php

declare(strict_types=1);

namespace Drupal\example;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Access\AccessResultInterface;
use Drupal\Core\Entity\EntityAccessControlHandler;
use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Session\AccountInterface;

/**
 * Access handler with ownership-aware logic.
 */
class ProjectAccessControlHandler extends EntityAccessControlHandler {

  /**
   * {@inheritdoc}
   */
  protected function checkAccess(EntityInterface $entity, $operation, AccountInterface $account): AccessResultInterface {
    /** @var \Drupal\example\Entity\Project $entity */

    // Admin permission bypasses all checks.
    $admin_permission = $this->entityType->getAdminPermission();
    if ($admin_permission && $account->hasPermission($admin_permission)) {
      return AccessResult::allowed()->cachePerPermissions();
    }

    $is_owner = ($account->id() && $account->id() === $entity->getOwnerId());

    return match ($operation) {
      'view' => $this->checkViewAccess($entity, $account, $is_owner),
      'update' => AccessResult::allowedIf($is_owner)
        ->orIf(AccessResult::allowedIfHasPermission($account, 'edit any project'))
        ->addCacheableDependency($entity)
        ->cachePerUser(),
      'delete' => AccessResult::allowedIfHasPermission($account, 'delete any project')
        ->orIf(
          AccessResult::allowedIf($is_owner)
            ->andIf(AccessResult::allowedIfHasPermission($account, 'delete own project'))
        )
        ->addCacheableDependency($entity)
        ->cachePerUser(),
      default => AccessResult::neutral(),
    };
  }

  /**
   * Checks view access.
   */
  protected function checkViewAccess(EntityInterface $entity, AccountInterface $account, bool $is_owner): AccessResultInterface {
    // Published entities: any viewer.
    if ($entity->isPublished()) {
      return AccessResult::allowedIfHasPermission($account, 'view project')
        ->addCacheableDependency($entity);
    }

    // Unpublished: only owner or admin.
    return AccessResult::allowedIf($is_owner)
      ->orIf(AccessResult::allowedIfHasPermission($account, 'view unpublished project'))
      ->addCacheableDependency($entity)
      ->cachePerUser();
  }

  /**
   * {@inheritdoc}
   */
  protected function checkCreateAccess(AccountInterface $account, array $context, $entity_bundle = NULL): AccessResultInterface {
    return AccessResult::allowedIfHasPermission($account, 'create project')
      ->cachePerPermissions();
  }

}
```

---

## Custom Storage Handler

Override the default SQL storage when you need to add custom load or save
logic.

```php
<?php

declare(strict_types=1);

namespace Drupal\example;

use Drupal\Core\Entity\Sql\SqlContentEntityStorage;
use Drupal\Core\Language\LanguageInterface;
use Drupal\Core\Session\AccountInterface;

/**
 * Custom storage handler for the Project entity.
 */
class ProjectStorage extends SqlContentEntityStorage {

  /**
   * Loads projects by their owner.
   *
   * @param \Drupal\Core\Session\AccountInterface $account
   *   The user account.
   *
   * @return \Drupal\example\Entity\Project[]
   *   An array of project entities.
   */
  public function loadByUser(AccountInterface $account): array {
    $ids = $this->getQuery()
      ->condition('uid', $account->id())
      ->accessCheck(TRUE)
      ->execute();

    return $this->loadMultiple($ids);
  }

  /**
   * Loads the most recent projects.
   *
   * @param int $limit
   *   Number of projects to load.
   *
   * @return \Drupal\example\Entity\Project[]
   *   An array of project entities.
   */
  public function loadRecent(int $limit = 10): array {
    $ids = $this->getQuery()
      ->condition('status', 1)
      ->sort('created', 'DESC')
      ->range(0, $limit)
      ->accessCheck(TRUE)
      ->execute();

    return $this->loadMultiple($ids);
  }

}
```

Register the storage handler in the entity type definition:

```php
  handlers: [
    'storage' => ProjectStorage::class,
    // ... other handlers
  ],
```

---

## Entity Operations

Add custom operation links to the list builder by implementing
`getDefaultOperations()`:

```php
/**
 * {@inheritdoc}
 */
public function getDefaultOperations(EntityInterface $entity): array {
  $operations = parent::getDefaultOperations($entity);

  if ($entity->access('update')) {
    $operations['clone'] = [
      'title' => $this->t('Clone'),
      'url' => $entity->toUrl('clone-form'),
      'weight' => 50,
    ];
  }

  return $operations;
}
```

Add the corresponding link template to the entity type definition:

```php
  links: [
    // ... existing links
    'clone-form' => '/project/{project}/clone',
  ],
```

---

## Entity Lifecycle Hooks

Override methods on the entity class to react to lifecycle events:

```php
/**
 * {@inheritdoc}
 */
public static function preCreate(EntityStorageInterface $storage, array &$values): void {
  parent::preCreate($storage, $values);
  // Set default owner to current user.
  if (empty($values['uid'])) {
    $values['uid'] = \Drupal::currentUser()->id();
  }
}

/**
 * {@inheritdoc}
 */
public function preSave(EntityStorageInterface $storage): void {
  parent::preSave($storage);
  // Automatically set the changed timestamp.
  // (EntityChangedTrait handles this, shown for illustration.)
}

/**
 * {@inheritdoc}
 */
public static function postDelete(EntityStorageInterface $storage, array $entities): void {
  parent::postDelete($storage, $entities);
  // Clean up related data.
  \Drupal::logger('example')->notice('Deleted @count projects.', [
    '@count' => count($entities),
  ]);
}
```
