# Cache Patterns and Advanced Techniques

## Custom Cache Backends

Drupal's cache system is backend-agnostic. The default backend stores data in
database tables, but you can swap to Redis, Memcached, APCu, or a custom
implementation.

### Configuring Redis as a Cache Backend

In `settings.php` (commonly used with the `redis` contributed module):

```php
$settings['cache']['default'] = 'cache.backend.redis';
$settings['redis.connection']['host'] = '127.0.0.1';
$settings['redis.connection']['port'] = '6379';
$settings['redis.connection']['base'] = 0;
$settings['cache_prefix']['default'] = 'mysite_';
```

### Per-Bin Backend Override

You can assign different backends to specific bins:

```php
// Keep the bootstrap bin in the database for reliability.
$settings['cache']['bins']['bootstrap'] = 'cache.backend.database';

// Use Redis for the render cache.
$settings['cache']['bins']['render'] = 'cache.backend.redis';

// Use APCu for the very-hot discovery bin.
$settings['cache']['bins']['discovery'] = 'cache.backend.apcu';

// Use a null backend to disable caching entirely for a bin (debugging only).
$settings['cache']['bins']['page'] = 'cache.backend.null';
```

### Implementing a Custom Cache Backend

Implement `\Drupal\Core\Cache\CacheBackendInterface` and register as a service:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Cache;

use Drupal\Core\Cache\CacheBackendInterface;
use Drupal\Core\Cache\CacheTagsChecksumInterface;
use Drupal\Core\Cache\Cache;

class MyCacheBackend implements CacheBackendInterface {

  public function __construct(
    protected readonly string $bin,
    protected readonly CacheTagsChecksumInterface $checksumProvider,
  ) {}

  /**
   * {@inheritdoc}
   */
  public function get($cid, $allow_invalid = FALSE): object|false {
    // Look up $cid in your storage.
    // Return an object with ->data, ->created, ->expire, ->tags properties.
    // Return FALSE on cache miss.
  }

  /**
   * {@inheritdoc}
   */
  public function getMultiple(&$cids, $allow_invalid = FALSE): array {
    $items = [];
    foreach ($cids as $index => $cid) {
      $item = $this->get($cid, $allow_invalid);
      if ($item) {
        $items[$cid] = $item;
        unset($cids[$index]);
      }
    }
    return $items;
  }

  /**
   * {@inheritdoc}
   */
  public function set($cid, $data, $expire = Cache::PERMANENT, array $tags = []): void {
    Cache::validateTags($tags);
    // Store $cid => $data along with $expire, $tags, and the current
    // tag checksum from $this->checksumProvider->getCurrentChecksum($tags).
  }

  /**
   * {@inheritdoc}
   */
  public function delete($cid): void {
    // Remove $cid from storage.
  }

  /**
   * {@inheritdoc}
   */
  public function deleteMultiple(array $cids): void {
    foreach ($cids as $cid) {
      $this->delete($cid);
    }
  }

  /**
   * {@inheritdoc}
   */
  public function deleteAll(): void {
    // Remove all entries in this bin.
  }

  /**
   * {@inheritdoc}
   */
  public function invalidate($cid): void {
    // Mark $cid as invalid without removing it.
  }

  /**
   * {@inheritdoc}
   */
  public function invalidateMultiple(array $cids): void {
    foreach ($cids as $cid) {
      $this->invalidate($cid);
    }
  }

  /**
   * {@inheritdoc}
   */
  public function invalidateAll(): void {
    // Mark all entries in this bin as invalid.
  }

  /**
   * {@inheritdoc}
   */
  public function garbageCollection(): void {
    // Clean up expired entries.
  }

  /**
   * {@inheritdoc}
   */
  public function removeBin(): void {
    // Destroy the entire bin.
  }

}
```

Register the backend factory in `my_module.services.yml`:

```yaml
services:
  cache.backend.my_custom:
    class: Drupal\my_module\Cache\MyCacheBackendFactory
    arguments: ['@cache_tags.invalidator.checksum']
```

---

## Cache Bin Definitions in Services

Every cache bin in Drupal is a tagged service. Core bins include `default`,
`bootstrap`, `config`, `data`, `discovery`, `entity`, `menu`, `render`,
`dynamic_page_cache`, and `page`.

### Defining a Custom Bin

```yaml
# my_module.services.yml
services:
  cache.my_module_results:
    class: Drupal\Core\Cache\CacheBackendInterface
    tags:
      - { name: cache.bin }
    factory: ['@cache_factory', 'get']
    arguments: ['my_module_results']
```

This creates a bin named `my_module_results`. If using the database backend, it
will be stored in a `cache_my_module_results` table that is automatically
created.

### Injecting a Cache Bin into a Service

```yaml
# my_module.services.yml
services:
  my_module.data_processor:
    class: Drupal\my_module\DataProcessor
    arguments: ['@cache.my_module_results']
```

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module;

use Drupal\Core\Cache\CacheBackendInterface;
use Drupal\Core\Cache\Cache;

class DataProcessor {

  public function __construct(
    protected readonly CacheBackendInterface $cache,
  ) {}

  public function getProcessedData(string $key): array {
    $cid = 'my_module:processed:' . $key;
    $cached = $this->cache->get($cid);
    if ($cached) {
      return $cached->data;
    }

    $data = $this->doExpensiveProcessing($key);
    $this->cache->set($cid, $data, Cache::PERMANENT, ['my_module:processed']);
    return $data;
  }

}
```

---

## Vary-By-Page Patterns

When output should be cached per-page, use appropriate URL-based cache contexts:

```php
// Vary by the entire URL (path + query string).
$build['#cache']['contexts'][] = 'url';

// Vary by path only (ignoring query parameters).
$build['#cache']['contexts'][] = 'url.path';

// Vary by a specific query parameter only.
$build['#cache']['contexts'][] = 'url.query_args:search';

// Vary by route (useful for blocks visible on multiple routes).
$build['#cache']['contexts'][] = 'route';
```

A common pattern for a block that shows content related to the currently viewed
node:

```php
public function build(): array {
  $node = \Drupal::routeMatch()->getParameter('node');
  if (!$node instanceof \Drupal\node\NodeInterface) {
    return [];
  }

  $build = [
    '#theme' => 'my_related_content',
    '#items' => $this->getRelatedContent($node),
    '#cache' => [
      'contexts' => ['route'],
      'tags' => ['node:' . $node->id(), 'node_list:article'],
      'max-age' => Cache::PERMANENT,
    ],
  ];

  return $build;
}
```

---

## Cache Debug Module

During development, enable the `cache_debug` settings or use contributed
modules to inspect cache behavior.

### Built-in Cache Debug Headers

In `settings.php` or `settings.local.php`:

```php
// Show X-Drupal-Cache (HIT/MISS) for the page cache.
$settings['display_cache_headers'] = TRUE;
```

### Development Settings

In `development.services.yml`, disable the render cache and dynamic page cache
for debugging:

```yaml
parameters:
  twig.config:
    debug: true
    cache: false
services:
  cache.backend.null:
    class: Drupal\Core\Cache\NullBackendFactory
```

Then in `settings.local.php`:

```php
$settings['container_yamls'][] = DRUPAL_ROOT . '/sites/development.services.yml';
$settings['cache']['bins']['render'] = 'cache.backend.null';
$settings['cache']['bins']['dynamic_page_cache'] = 'cache.backend.null';
$settings['cache']['bins']['page'] = 'cache.backend.null';
```

### Inspecting Cache Tags on Responses

Drupal adds an `X-Drupal-Cache-Tags` header (when page cache is active) and
an `X-Drupal-Cache-Contexts` header. Use browser dev tools or curl:

```bash
curl -I https://mysite.ddev.site/node/1
# Look for:
# X-Drupal-Cache: HIT
# X-Drupal-Cache-Tags: node:1 node_view rendered ...
# X-Drupal-Cache-Contexts: languages:language_interface theme url.query_args ...
```

---

## BigPipe and Lazy Builders for Personalization

When a page is mostly cacheable but contains small personalized fragments
(e.g., a user greeting, a shopping cart count), use **lazy builders** to
isolate the uncacheable parts. Drupal's **BigPipe** module streams these
fragments after the main page is sent.

### Defining a Lazy Builder

```php
$build['user_greeting'] = [
  '#lazy_builder' => [
    'my_module.greeting_builder:build',
    [],
  ],
  '#create_placeholder' => TRUE,
];
```

The service:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module;

use Drupal\Core\Security\TrustedCallbackInterface;
use Drupal\Core\Session\AccountProxyInterface;

class GreetingBuilder implements TrustedCallbackInterface {

  public function __construct(
    protected readonly AccountProxyInterface $currentUser,
  ) {}

  public function build(): array {
    return [
      '#markup' => 'Welcome, ' . $this->currentUser->getDisplayName() . '!',
      '#cache' => [
        'contexts' => ['user'],
        'max-age' => 0,
      ],
    ];
  }

  /**
   * {@inheritdoc}
   */
  public static function trustedCallbacks(): array {
    return ['build'];
  }

}
```

Register it:

```yaml
services:
  my_module.greeting_builder:
    class: Drupal\my_module\GreetingBuilder
    arguments: ['@current_user']
```

### Auto-Placeholdering

Drupal automatically converts render elements with high-cardinality cache
contexts (like `user` or `session`) into placeholders. This is controlled by
the `renderer.config` parameter in `core.services.yml`:

```yaml
parameters:
  renderer.config:
    auto_placeholder_conditions:
      max-age: 0
      contexts:
        - 'session'
        - 'user'
      tags: []
```

Any render element whose cache metadata matches these conditions is
automatically turned into a lazy-builder placeholder — no manual
`#create_placeholder` needed. BigPipe will then replace these placeholders
during streaming.

This means you should **not** set `max-age = 0` on an entire page just because
one element is personalized. Instead, set `max-age = 0` (or `contexts = ['user']`)
on the personalized element, and Drupal's auto-placeholdering will handle the
rest.

---

## CacheableMetadata Merge Patterns

`\Drupal\Core\Cache\CacheableMetadata` is the utility class for working with
cache metadata outside of render arrays.

### Creating from Various Sources

```php
use Drupal\Core\Cache\CacheableMetadata;

// From a render array.
$metadata = CacheableMetadata::createFromRenderArray($build);

// From any CacheableDependencyInterface object (entities, config, access results).
$metadata = CacheableMetadata::createFromObject($node);

// Empty metadata to start merging into.
$metadata = new CacheableMetadata();
```

### Merging Cache Metadata

Merging combines tags (union), contexts (union), and max-age (minimum):

```php
$metadata = new CacheableMetadata();

// Merge metadata from multiple entities.
foreach ($nodes as $node) {
  $metadata->addCacheableDependency($node);
}

// Merge metadata from an access result.
$access = $entity->access('view', $account, TRUE);
$metadata->addCacheableDependency($access);

// Merge metadata from a config object.
$config = \Drupal::config('my_module.settings');
$metadata->addCacheableDependency($config);

// Apply all merged metadata to a render array.
$metadata->applyTo($build);
```

### Practical Pattern: Aggregating a List

```php
public function buildList(): array {
  $storage = $this->entityTypeManager->getStorage('project');
  $ids = $storage->getQuery()
    ->condition('status', TRUE)
    ->accessCheck(TRUE)
    ->execute();
  $projects = $storage->loadMultiple($ids);

  $metadata = new CacheableMetadata();
  // The list itself should invalidate when any project changes.
  $metadata->addCacheTags(['project_list']);
  $metadata->addCacheContexts(['user.permissions']);

  $items = [];
  foreach ($projects as $project) {
    $metadata->addCacheableDependency($project);
    $items[] = [
      '#markup' => $project->label(),
    ];
  }

  $build = [
    '#theme' => 'item_list',
    '#items' => $items,
  ];
  $metadata->applyTo($build);

  return $build;
}
```

---

## Cache Tag Invalidation for Custom Entities

When you define a custom content entity, Drupal's Entity API automatically
handles standard cache tag invalidation on CRUD operations. However, there are
scenarios where you need custom invalidation.

### Invalidating on Related Entity Changes

If entity A displays data derived from entity B, you need to invalidate A's
cache when B changes:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module;

use Drupal\Core\Cache\Cache;
use Drupal\Core\Entity\EntityInterface;
use Drupal\my_module\Entity\Project;

/**
 * Implements hook_entity_update() to cross-invalidate cache tags.
 */
function my_module_entity_update(EntityInterface $entity): void {
  // When a team member (user) is updated, invalidate the project they belong to.
  if ($entity->getEntityTypeId() === 'user') {
    $project_ids = \Drupal::entityQuery('project')
      ->condition('field_team_members', $entity->id())
      ->accessCheck(FALSE)
      ->execute();

    if ($project_ids) {
      $tags = [];
      foreach ($project_ids as $id) {
        $tags[] = 'project:' . $id;
      }
      Cache::invalidateTags($tags);
    }
  }
}
```

### Custom List Cache Tags

Beyond the standard `{entity_type}_list` tag, you can define more granular list
tags. For example, a per-status list tag:

```php
// In a render array listing active projects.
$build['#cache']['tags'][] = 'project_list:active';

// When a project changes status, invalidate the relevant list tag.
function my_module_project_update(Project $project): void {
  $original = $project->original;
  if ($original && $original->get('status')->value !== $project->get('status')->value) {
    Cache::invalidateTags([
      'project_list:active',
      'project_list:archived',
    ]);
  }
}
```

---

## Cache Redirects

Cache redirects are an internal mechanism used by Drupal's render cache to
handle situations where cache contexts need to be resolved in a specific order.
When Drupal encounters a render array with contexts that can be optimized, it
may store a **redirect** entry that points to a more specific cache entry.

You do not create cache redirects manually. However, understanding them helps
when debugging:

- A cache redirect is a cache item whose data is a `CacheRedirect` object.
- If you inspect the `render` cache bin and see redirect entries, this is normal.
- Redirects allow Drupal to avoid storing redundant variants. For example,
  if `url.path` is a parent context of `url`, Drupal can redirect from the
  `url.path` variant to the full `url` variant.

The key takeaway: if you see unexpected behavior in the render cache, check
whether cache redirects are involved by inspecting the raw cache entries in
the `cache_render` table or your cache backend.

---

## Summary of Best Practices

1. **Always add cache metadata** to render arrays, even in blocks and
   controllers. Never return a render array without `#cache`.

2. **Prefer cache tags over max-age** for invalidation. Tags give you precise
   control; max-age is a blunt timeout.

3. **Use the narrowest cache context** that represents your variation. Do not
   use `user` if `user.permissions` suffices.

4. **Use lazy builders** for personalized or uncacheable fragments. Do not make
   an entire page uncacheable because of one dynamic element.

5. **Merge cache metadata** from all data sources using `CacheableMetadata` when
   building complex render arrays.

6. **Define custom cache bins** for module-specific data that has different
   lifecycle or backend requirements.

7. **Test cache behavior** by checking response headers and using cache debug
   settings during development.
