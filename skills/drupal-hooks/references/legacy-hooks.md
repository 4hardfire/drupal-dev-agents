# Legacy Procedural Hooks Reference

A catalog of the most commonly used Drupal hooks with their function signatures
and descriptions. All hooks follow the naming pattern `{modulename}_{hookname}()`.

---

## Entity Hooks

These hooks fire during entity CRUD operations. Defined in `core/lib/Drupal/Core/Entity/entity.api.php`.

```php
/**
 * Implements hook_entity_presave().
 *
 * Fires before an entity is saved (both insert and update).
 */
function mymodule_entity_presave(\Drupal\Core\Entity\EntityInterface $entity): void {}

/**
 * Implements hook_entity_insert().
 *
 * Fires after a new entity is saved to the database for the first time.
 */
function mymodule_entity_insert(\Drupal\Core\Entity\EntityInterface $entity): void {}

/**
 * Implements hook_entity_update().
 *
 * Fires after an existing entity is saved with changes.
 */
function mymodule_entity_update(\Drupal\Core\Entity\EntityInterface $entity): void {}

/**
 * Implements hook_entity_delete().
 *
 * Fires after an entity has been deleted from the database.
 */
function mymodule_entity_delete(\Drupal\Core\Entity\EntityInterface $entity): void {}

/**
 * Implements hook_entity_load().
 *
 * Fires after entities are loaded from storage.
 */
function mymodule_entity_load(array $entities, string $entity_type_id): void {}

/**
 * Implements hook_entity_view().
 *
 * Fires when an entity's render array is being built.
 */
function mymodule_entity_view(array &$build, \Drupal\Core\Entity\EntityInterface $entity, \Drupal\Core\Entity\Display\EntityViewDisplayInterface $display, string $view_mode): void {}

/**
 * Implements hook_entity_view_alter().
 *
 * Alters the render array after all hook_entity_view() implementations.
 */
function mymodule_entity_view_alter(array &$build, \Drupal\Core\Entity\EntityInterface $entity, \Drupal\Core\Entity\Display\EntityViewDisplayInterface $display): void {}

/**
 * Implements hook_entity_access().
 *
 * Controls access to an entity operation.
 */
function mymodule_entity_access(\Drupal\Core\Entity\EntityInterface $entity, string $operation, \Drupal\Core\Session\AccountInterface $account): \Drupal\Core\Access\AccessResultInterface {
  return \Drupal\Core\Access\AccessResult::neutral();
}

/**
 * Implements hook_entity_create_access().
 *
 * Controls access to create an entity of a given type.
 */
function mymodule_entity_create_access(\Drupal\Core\Session\AccountInterface $account, array $context, ?string $entity_bundle): \Drupal\Core\Access\AccessResultInterface {
  return \Drupal\Core\Access\AccessResult::neutral();
}

/**
 * Implements hook_ENTITY_TYPE_presave() for node entities.
 *
 * Entity-type-specific variant. Replace ENTITY_TYPE with the entity type ID.
 */
function mymodule_node_presave(\Drupal\Core\Entity\EntityInterface $entity): void {}
```

---

## Form Hooks

Defined in `core/lib/Drupal/Core/Form/form.api.php`.

```php
/**
 * Implements hook_form_alter().
 *
 * Alter any form before rendering. Called for every form on the site.
 */
function mymodule_form_alter(array &$form, \Drupal\Core\Form\FormStateInterface $form_state, string $form_id): void {}

/**
 * Implements hook_form_FORM_ID_alter() for user_login_form.
 *
 * Alter a specific form by its form ID. More efficient than hook_form_alter()
 * when you only need to alter one form.
 */
function mymodule_form_user_login_form_alter(array &$form, \Drupal\Core\Form\FormStateInterface $form_state, string $form_id): void {}

/**
 * Implements hook_form_BASE_FORM_ID_alter() for node_form.
 *
 * Alter all forms that share a base form ID (e.g., all node forms).
 */
function mymodule_form_node_form_alter(array &$form, \Drupal\Core\Form\FormStateInterface $form_state, string $form_id): void {}
```

---

## Render and Theme Hooks

Defined in `core/lib/Drupal/Core/Render/theme.api.php`.

```php
/**
 * Implements hook_theme().
 *
 * Register theme implementations (templates or theme functions).
 */
function mymodule_theme(array $existing, string $type, string $theme, string $path): array {
  return [
    'mymodule_item' => [
      'variables' => ['title' => NULL, 'description' => NULL],
      'template' => 'mymodule-item',
    ],
  ];
}

/**
 * Implements hook_theme_suggestions_HOOK() for node templates.
 *
 * Add custom theme suggestions.
 */
function mymodule_theme_suggestions_node(array $variables): array {
  $suggestions = [];
  $node = $variables['elements']['#node'];
  $suggestions[] = 'node__' . $node->bundle() . '__custom';
  return $suggestions;
}

/**
 * Implements hook_theme_suggestions_alter().
 *
 * Alter theme suggestions for all hooks.
 */
function mymodule_theme_suggestions_alter(array &$suggestions, array $variables, string $hook): void {}

/**
 * Implements hook_preprocess_HOOK() for node templates.
 *
 * Preprocess variables before a template is rendered.
 */
function mymodule_preprocess_node(array &$variables): void {
  $node = $variables['node'];
  $variables['custom_variable'] = $node->get('field_custom')->value;
}

/**
 * Implements hook_page_attachments().
 *
 * Add assets (CSS/JS libraries) to all pages.
 */
function mymodule_page_attachments(array &$attachments): void {
  $attachments['#attached']['library'][] = 'mymodule/global-styles';
}

/**
 * Implements hook_page_attachments_alter().
 *
 * Alter page attachments added by other modules.
 */
function mymodule_page_attachments_alter(array &$attachments): void {}
```

---

## Node Hooks

Entity-type-specific hooks for nodes.

```php
/**
 * Implements hook_node_access().
 */
function mymodule_node_access(\Drupal\node\NodeInterface $node, string $operation, \Drupal\Core\Session\AccountInterface $account): \Drupal\Core\Access\AccessResultInterface {
  return \Drupal\Core\Access\AccessResult::neutral();
}

/**
 * Implements hook_node_presave().
 */
function mymodule_node_presave(\Drupal\node\NodeInterface $node): void {}

/**
 * Implements hook_node_insert().
 */
function mymodule_node_insert(\Drupal\node\NodeInterface $node): void {}

/**
 * Implements hook_node_update().
 */
function mymodule_node_update(\Drupal\node\NodeInterface $node): void {}

/**
 * Implements hook_node_delete().
 */
function mymodule_node_delete(\Drupal\node\NodeInterface $node): void {}

/**
 * Implements hook_node_view().
 */
function mymodule_node_view(array &$build, \Drupal\node\NodeInterface $node, \Drupal\Core\Entity\Display\EntityViewDisplayInterface $display, string $view_mode): void {}
```

---

## User Hooks

```php
/**
 * Implements hook_user_login().
 *
 * Fires when a user logs in.
 */
function mymodule_user_login(\Drupal\Core\Session\AccountInterface $account): void {}

/**
 * Implements hook_user_logout().
 *
 * Fires when a user logs out.
 */
function mymodule_user_logout(\Drupal\Core\Session\AccountInterface $account): void {}

/**
 * Implements hook_user_presave().
 */
function mymodule_user_presave(\Drupal\user\UserInterface $account): void {}

/**
 * Implements hook_user_insert().
 */
function mymodule_user_insert(\Drupal\user\UserInterface $account): void {}

/**
 * Implements hook_user_cancel().
 *
 * Fires when a user account is being cancelled.
 */
function mymodule_user_cancel(array $edit, \Drupal\Core\Session\AccountInterface $account, string $method): void {}
```

---

## Cron Hooks

```php
/**
 * Implements hook_cron().
 *
 * Called each time cron runs. Keep operations lightweight or use queues.
 */
function mymodule_cron(): void {}
```

---

## Install and Update Hooks

Defined in `mymodule.install`, not `mymodule.module`.

```php
/**
 * Implements hook_install().
 *
 * Runs when the module is first installed.
 */
function mymodule_install(): void {}

/**
 * Implements hook_uninstall().
 *
 * Runs when the module is uninstalled. Clean up all module data.
 */
function mymodule_uninstall(): void {}

/**
 * Implements hook_schema().
 *
 * Define database tables for the module.
 */
function mymodule_schema(): array {
  $schema['mymodule_data'] = [
    'description' => 'Stores custom data.',
    'fields' => [
      'id' => ['type' => 'serial', 'unsigned' => TRUE, 'not null' => TRUE],
      'name' => ['type' => 'varchar', 'length' => 255, 'not null' => TRUE],
      'created' => ['type' => 'int', 'not null' => TRUE, 'default' => 0],
    ],
    'primary key' => ['id'],
  ];
  return $schema;
}

/**
 * Implements hook_update_N().
 *
 * Update hook for schema or data migrations. N is a sequential number.
 * For Drupal 10+, numbering starts at {module_major_version}01.
 */
function mymodule_update_10001(): void {
  // Perform a database update or data migration.
}

/**
 * Implements hook_requirements().
 *
 * Check installation requirements and report status.
 */
function mymodule_requirements(string $phase): array {
  $requirements = [];
  if ($phase === 'runtime') {
    $requirements['mymodule_status'] = [
      'title' => t('My Module'),
      'value' => t('OK'),
      'severity' => REQUIREMENT_OK,
    ];
  }
  return $requirements;
}
```

---

## Token Hooks

```php
/**
 * Implements hook_token_info().
 *
 * Define available tokens.
 */
function mymodule_token_info(): array {
  $types['mymodule'] = [
    'name' => t('My Module'),
    'description' => t('Custom tokens for My Module.'),
  ];
  $tokens['mymodule']['example'] = [
    'name' => t('Example'),
    'description' => t('An example token value.'),
  ];
  return ['types' => $types, 'tokens' => $tokens];
}

/**
 * Implements hook_tokens().
 *
 * Provide replacement values for tokens.
 */
function mymodule_tokens(string $type, array $tokens, array $data, array $options, \Drupal\Core\Render\BubbleableMetadata $bubbleable_metadata): array {
  $replacements = [];
  if ($type === 'mymodule') {
    foreach ($tokens as $name => $original) {
      if ($name === 'example') {
        $replacements[$original] = 'replacement value';
      }
    }
  }
  return $replacements;
}
```

---

## Menu and Routing Hooks

```php
/**
 * Implements hook_menu_links_discovered_alter().
 *
 * Alter menu links defined in *.links.menu.yml files.
 */
function mymodule_menu_links_discovered_alter(array &$links): void {}

/**
 * Implements hook_local_tasks_alter().
 *
 * Alter local task (tab) definitions.
 */
function mymodule_local_tasks_alter(array &$local_tasks): void {}

/**
 * Implements hook_menu_local_actions_alter().
 *
 * Alter local action definitions.
 */
function mymodule_menu_local_actions_alter(array &$local_actions): void {}
```

---

## Block Hooks

```php
/**
 * Implements hook_block_view_alter().
 *
 * Alter the render array of all blocks.
 */
function mymodule_block_view_alter(array &$build, \Drupal\Core\Block\BlockPluginInterface $block): void {}

/**
 * Implements hook_block_view_BASE_BLOCK_ID_alter() for system_branding_block.
 *
 * Alter a specific block's render array.
 */
function mymodule_block_view_system_branding_block_alter(array &$build, \Drupal\Core\Block\BlockPluginInterface $block): void {}

/**
 * Implements hook_block_access().
 */
function mymodule_block_access(\Drupal\block\Entity\Block $block, string $operation, \Drupal\Core\Session\AccountInterface $account): \Drupal\Core\Access\AccessResultInterface {
  return \Drupal\Core\Access\AccessResult::neutral();
}
```

---

## Views Hooks

```php
/**
 * Implements hook_views_data().
 *
 * Describe custom database tables to Views.
 */
function mymodule_views_data(): array {
  $data = [];
  $data['mymodule_data'] = [
    'table' => [
      'group' => t('My Module'),
      'base' => [
        'field' => 'id',
        'title' => t('My Module Data'),
      ],
    ],
    'id' => [
      'title' => t('ID'),
      'field' => ['id' => 'numeric'],
      'sort' => ['id' => 'standard'],
      'filter' => ['id' => 'numeric'],
    ],
  ];
  return $data;
}

/**
 * Implements hook_views_data_alter().
 *
 * Alter Views data defined by other modules.
 */
function mymodule_views_data_alter(array &$data): void {}

/**
 * Implements hook_views_pre_view().
 */
function mymodule_views_pre_view(\Drupal\views\ViewExecutable $view, string $display_id, array &$args): void {}

/**
 * Implements hook_views_pre_render().
 */
function mymodule_views_pre_render(\Drupal\views\ViewExecutable $view): void {}

/**
 * Implements hook_views_query_alter().
 */
function mymodule_views_query_alter(\Drupal\views\ViewExecutable $view, \Drupal\views\Plugin\views\query\QueryPluginBase $query): void {}
```
