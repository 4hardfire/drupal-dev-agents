# Block Plugins — Deep Dive

## Overview

Block plugins are one of the most commonly created Drupal plugins. They display content in theme regions and are managed through the Block Layout admin UI or placed programmatically.

- **Base class**: `Drupal\Core\Block\BlockBase`
- **Manager service**: `plugin.manager.block`
- **File location**: `src/Plugin/Block/`
- **Config entity**: Each placed block is a `block` config entity referencing the plugin.

---

## Block with Configuration Form

A block that stores configuration and provides an admin form for editing it.

### Drupal 10 (Annotation)

```php
<?php

namespace Drupal\my_module\Plugin\Block;

use Drupal\Core\Block\BlockBase;
use Drupal\Core\Form\FormStateInterface;

/**
 * Provides a configurable promotional banner block.
 *
 * @Block(
 *   id = "my_module_promo_banner",
 *   admin_label = @Translation("Promotional banner"),
 *   category = @Translation("Custom"),
 * )
 */
class PromoBannerBlock extends BlockBase {

  /**
   * {@inheritdoc}
   */
  public function defaultConfiguration(): array {
    return [
      'heading' => '',
      'body' => '',
      'link_url' => '',
      'link_text' => 'Learn more',
    ];
  }

  /**
   * {@inheritdoc}
   */
  public function blockForm($form, FormStateInterface $form_state): array {
    $form['heading'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Heading'),
      '#default_value' => $this->configuration['heading'],
      '#required' => TRUE,
    ];
    $form['body'] = [
      '#type' => 'text_format',
      '#title' => $this->t('Body'),
      '#default_value' => $this->configuration['body'],
      '#format' => 'basic_html',
    ];
    $form['link_url'] = [
      '#type' => 'url',
      '#title' => $this->t('Link URL'),
      '#default_value' => $this->configuration['link_url'],
    ];
    $form['link_text'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Link text'),
      '#default_value' => $this->configuration['link_text'],
    ];
    return $form;
  }

  /**
   * {@inheritdoc}
   */
  public function blockSubmit($form, FormStateInterface $form_state): void {
    $this->configuration['heading'] = $form_state->getValue('heading');
    $this->configuration['body'] = $form_state->getValue('body')['value'] ?? $form_state->getValue('body');
    $this->configuration['link_url'] = $form_state->getValue('link_url');
    $this->configuration['link_text'] = $form_state->getValue('link_text');
  }

  /**
   * {@inheritdoc}
   */
  public function build(): array {
    return [
      '#theme' => 'my_module_promo_banner',
      '#heading' => $this->configuration['heading'],
      '#body' => $this->configuration['body'],
      '#link_url' => $this->configuration['link_url'],
      '#link_text' => $this->configuration['link_text'],
    ];
  }

}
```

### Drupal 11+ (Attribute)

```php
<?php

namespace Drupal\my_module\Plugin\Block;

use Drupal\Core\Block\Attribute\Block;
use Drupal\Core\Block\BlockBase;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\StringTranslation\TranslatableMarkup;

#[Block(
  id: 'my_module_promo_banner',
  admin_label: new TranslatableMarkup('Promotional banner'),
  category: new TranslatableMarkup('Custom'),
)]
class PromoBannerBlock extends BlockBase {

  // All methods are identical to the annotation version above.

}
```

---

## Block Access Control

Override `blockAccess()` to restrict who can see the block. This is checked before `build()` is called.

```php
use Drupal\Core\Access\AccessResult;
use Drupal\Core\Session\AccountInterface;

protected function blockAccess(AccountInterface $account): AccessResult {
  // Only show to authenticated users with a specific permission.
  return AccessResult::allowedIfHasPermission($account, 'view promo banners')
    ->addCacheContexts(['user.permissions']);
}
```

Access results should carry cache metadata so Drupal knows when to re-evaluate:

```php
// Varies per user.
AccessResult::allowedIfHasPermission($account, 'some perm')
  ->addCacheContexts(['user.permissions']);

// Varies per role.
AccessResult::allowedIf($account->hasRole('editor'))
  ->addCacheContexts(['user.roles']);

// Forbidden for anonymous users.
AccessResult::forbiddenIf($account->isAnonymous())
  ->addCacheContexts(['user.roles:anonymous']);
```

---

## Block Context

Block context lets a block declare that it requires a specific type of data (e.g., the current node). Drupal supplies this data automatically based on the route.

### Drupal 10 (Annotation)

```php
/**
 * Provides a block that displays node author info.
 *
 * @Block(
 *   id = "my_module_node_author",
 *   admin_label = @Translation("Node author info"),
 *   category = @Translation("Custom"),
 *   context_definitions = {
 *     "node" = @ContextDefinition("entity:node", label = @Translation("Node"))
 *   }
 * )
 */
class NodeAuthorBlock extends BlockBase {

  public function build(): array {
    /** @var \Drupal\node\NodeInterface $node */
    $node = $this->getContextValue('node');
    $author = $node->getOwner();

    return [
      '#markup' => $this->t('Written by @name', ['@name' => $author->getDisplayName()]),
      '#cache' => [
        'contexts' => ['route'],
        'tags' => $node->getCacheTags(),
      ],
    ];
  }

}
```

### Drupal 11+ (Attribute)

```php
use Drupal\Core\Block\Attribute\Block;
use Drupal\Core\Plugin\Context\ContextDefinition;
use Drupal\Core\StringTranslation\TranslatableMarkup;

#[Block(
  id: 'my_module_node_author',
  admin_label: new TranslatableMarkup('Node author info'),
  category: new TranslatableMarkup('Custom'),
  context_definitions: [
    'node' => new ContextDefinition(
      data_type: 'entity:node',
      label: new TranslatableMarkup('Node'),
    ),
  ],
)]
class NodeAuthorBlock extends BlockBase {

  // build() is the same as the annotation version.

}
```

---

## Block Derivatives (Deriver)

Derivatives let a single block class generate multiple block instances dynamically. For example, one block per vocabulary, one block per content type, etc.

### The Deriver Class

```php
<?php

namespace Drupal\my_module\Plugin\Derivative;

use Drupal\Component\Plugin\Derivative\DeriverBase;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Plugin\Discovery\ContainerDeriverInterface;
use Drupal\Core\StringTranslation\StringTranslationTrait;
use Symfony\Component\DependencyInjection\ContainerInterface;

class VocabularyBlockDeriver extends DeriverBase implements ContainerDeriverInterface {

  use StringTranslationTrait;

  public function __construct(
    protected EntityTypeManagerInterface $entityTypeManager,
  ) {}

  /**
   * {@inheritdoc}
   */
  public static function create(ContainerInterface $container, $base_plugin_id): static {
    return new static(
      $container->get('entity_type.manager'),
    );
  }

  /**
   * {@inheritdoc}
   */
  public function getDerivativeDefinitions($base_plugin_definition): array {
    $vocabularies = $this->entityTypeManager->getStorage('taxonomy_vocabulary')->loadMultiple();

    foreach ($vocabularies as $id => $vocabulary) {
      $this->derivatives[$id] = $base_plugin_definition;
      $this->derivatives[$id]['admin_label'] = $this->t('Terms: @name', ['@name' => $vocabulary->label()]);
      $this->derivatives[$id]['vocabulary'] = $id;
    }

    return $this->derivatives;
  }

}
```

### The Block Class (Drupal 10 Annotation)

```php
/**
 * Provides a vocabulary terms block.
 *
 * @Block(
 *   id = "my_module_vocabulary_terms",
 *   admin_label = @Translation("Vocabulary terms"),
 *   category = @Translation("Custom"),
 *   deriver = "Drupal\my_module\Plugin\Derivative\VocabularyBlockDeriver"
 * )
 */
class VocabularyTermsBlock extends BlockBase {

  public function build(): array {
    $vocabulary_id = $this->getDerivativeId();
    // Load and display terms for this vocabulary...
    return [
      '#markup' => $this->t('Terms for vocabulary: @vocab', ['@vocab' => $vocabulary_id]),
      '#cache' => [
        'tags' => ['taxonomy_term_list:' . $vocabulary_id],
      ],
    ];
  }

}
```

### The Block Class (Drupal 11+ Attribute)

```php
use Drupal\Core\Block\Attribute\Block;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use Drupal\my_module\Plugin\Derivative\VocabularyBlockDeriver;

#[Block(
  id: 'my_module_vocabulary_terms',
  admin_label: new TranslatableMarkup('Vocabulary terms'),
  category: new TranslatableMarkup('Custom'),
  deriver: VocabularyBlockDeriver::class,
)]
class VocabularyTermsBlock extends BlockBase {

  // build() is identical.

}
```

---

## Cache Metadata on Blocks

Every block's `build()` method should return appropriate cache metadata:

```php
public function build(): array {
  return [
    '#markup' => $this->t('Some content'),
    '#cache' => [
      // When does this vary?
      'contexts' => [
        'user.roles',          // Different per role.
        'url.path',            // Different per page.
        'languages:language_interface', // Different per language.
      ],
      // When should this be invalidated?
      'tags' => [
        'node_list',           // When any node changes.
        'config:system.site',  // When site config changes.
      ],
      // How long is it valid?
      'max-age' => 3600,      // 1 hour. Use 0 for uncacheable, -1 for permanent.
    ],
  ];
}
```

For blocks with service dependencies, inject services via `ContainerFactoryPluginInterface`:

```php
use Drupal\Core\Plugin\ContainerFactoryPluginInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

class MyBlock extends BlockBase implements ContainerFactoryPluginInterface {

  public function __construct(
    array $configuration,
    $plugin_id,
    $plugin_definition,
    protected EntityTypeManagerInterface $entityTypeManager,
  ) {
    parent::__construct($configuration, $plugin_id, $plugin_definition);
  }

  public static function create(
    ContainerInterface $container,
    array $configuration,
    $plugin_id,
    $plugin_definition,
  ): static {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition,
      $container->get('entity_type.manager'),
    );
  }

}
```
