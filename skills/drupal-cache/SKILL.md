---
name: drupal-cache
description: Use when working with Drupal caching, cache tags, cache contexts, max-age, cache bins, CacheableDependencyInterface, or cache invalidation strategies.
version: 1.0.0
---

# Drupal Cache System

## Overview

Drupal's cache system is built around three pillars of **cache metadata**: cache
tags, cache contexts, and max-age. Together these determine *what* can be cached,
*when* it varies, and *how long* it remains valid. Cache metadata is attached to
render arrays and propagated upward through the render tree via **bubbling**.

Beyond render arrays, Drupal provides **cache bins** — named storage buckets
backed by a **cache backend** (database by default, or Redis, Memcached, etc.).
You interact with bins through `CacheBackendInterface`.

Understanding caching is essential for performance and correctness. Forgetting
cache metadata is the most common cause of "stale content" bugs in custom
modules.

---

## Cache Metadata on Render Arrays

Every render array can carry a `#cache` key with three sub-keys:

```php
$build = [
  '#theme' => 'my_template',
  '#data' => $data,
  '#cache' => [
    'tags' => ['node:42', 'user:7'],
    'contexts' => ['user.permissions', 'url.query_args'],
    'max-age' => 3600,
  ],
];
```

| Key | Purpose | Example values |
|---|---|---|
| `tags` | Invalidation identifiers — when any tag is invalidated the cached version is discarded | `node:1`, `node_list`, `config:system.site` |
| `contexts` | Vary-by dimensions — a separate cached variant is stored per unique context value | `user`, `url`, `languages:language_interface` |
| `max-age` | Time-based expiration in seconds | `Cache::PERMANENT` (forever), `3600`, `0` (uncacheable) |

If you omit `#cache` entirely on a render array that produces varying output,
Drupal may serve stale results to the wrong users.

---

## CacheableDependencyInterface

When your custom object or service response carries cache metadata, implement
`\Drupal\Core\Cache\CacheableDependencyInterface`:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module;

use Drupal\Core\Cache\CacheableDependencyInterface;
use Drupal\Core\Cache\Cache;

class WeatherData implements CacheableDependencyInterface {

  public function __construct(
    protected readonly string $city,
    protected readonly array $forecast,
  ) {}

  /**
   * {@inheritdoc}
   */
  public function getCacheTags(): array {
    return ['my_module:weather:' . $this->city];
  }

  /**
   * {@inheritdoc}
   */
  public function getCacheContexts(): array {
    // Varies by nothing beyond what the caller adds.
    return [];
  }

  /**
   * {@inheritdoc}
   */
  public function getCacheMaxAge(): int {
    // Cache for 15 minutes.
    return 900;
  }

}
```

You can then merge this metadata into a render array:

```php
use Drupal\Core\Cache\CacheableMetadata;

$cacheability = CacheableMetadata::createFromObject($weather_data);
$cacheability->applyTo($build);
```

For cacheable JSON responses (REST, JSON:API custom routes), implement
`CacheableDependencyInterface` on the resource response or use
`CacheableJsonResponse` with `addCacheableDependency()`.

---

## Cache Tags

Cache tags enable precise, targeted invalidation. When an entity is saved,
Drupal automatically invalidates its tags, so every render array or cache entry
referencing that entity is cleared.

### Standard Entity Tags

| Tag format | Meaning | Invalidated when |
|---|---|---|
| `node:42` | A specific node | That node is saved or deleted |
| `node_list` | The list of all nodes | Any node is created, saved, or deleted |
| `node_list:article` | Nodes of bundle "article" | Any article node is created, saved, or deleted |
| `user:7` | A specific user | That user is saved |
| `taxonomy_term:5` | A specific term | That term is saved or deleted |
| `config:system.site` | A config object | That config is saved |

### Custom Tags and Invalidation

Define your own tags for custom data. Invalidate with `Cache::invalidateTags()`:

```php
use Drupal\Core\Cache\Cache;

// In a render array.
$build['#cache']['tags'][] = 'my_module:external_feed';

// When external feed data changes (e.g., in a cron hook).
Cache::invalidateTags(['my_module:external_feed']);
```

You can also invalidate via the service:

```php
\Drupal::service('cache_tags.invalidator')->invalidateTags(['my_module:external_feed']);
```

### List Tags for Custom Entities

If you define a custom entity type `project`, Drupal automatically provides
`project:{id}` and `project_list` tags. Entity list builders and Views that
list your entity will use `project_list` so that adding a new project
invalidates listings.

---

## Cache Contexts

Cache contexts describe the **dimensions** along which output varies. Drupal
stores a separate cached variant for each unique combination of context values.

### Built-in Contexts

| Context | Varies by |
|---|---|
| `user` | The individual user (user ID) |
| `user.permissions` | The calculated permissions hash for the user |
| `user.roles` | The set of roles assigned to the user |
| `user.is_super_user` | Whether the user is UID 1 |
| `url` | The full URL (path + query string) |
| `url.path` | The URL path only |
| `url.query_args` | All query string parameters |
| `url.query_args:page` | A specific query argument |
| `session` | The session ID |
| `theme` | The active theme |
| `languages:language_interface` | The interface language |
| `languages:language_content` | The content language |
| `timezone` | The user's timezone |
| `ip` | The client IP address |
| `route` | The route name |
| `route.name` | Same as `route` |

### Choosing the Right Context

Use the **narrowest** context that correctly represents the variation:

- If output changes per role, use `user.roles` (not `user`). This avoids
  storing one variant per individual user when roles are the real differentiator.
- If output depends on a query parameter, use `url.query_args:param_name`
  rather than the full `url.query_args`.
- `user.permissions` is preferred over `user.roles` when the variation is truly
  about what the user can do, because two different roles may have the same
  permissions.

---

## Max-Age

Max-age controls time-based cache expiration:

```php
use Drupal\Core\Cache\Cache;

// Cache forever (invalidated only by tags).
$build['#cache']['max-age'] = Cache::PERMANENT;

// Cache for one hour.
$build['#cache']['max-age'] = 3600;

// Never cache — the element must be rebuilt every request.
$build['#cache']['max-age'] = 0;
```

When cache metadata bubbles up, the **minimum** max-age wins. A single
`max-age = 0` element makes the entire page uncacheable, unless it is
auto-placeholdered (see the references for lazy builders and BigPipe).

---

## Cache Bins

A cache bin is a named partition of cache storage. Each bin can use a different
backend.

### Using a Cache Bin

```php
// Retrieve the default cache bin.
$cache = \Drupal::cache();

// Use a specific bin.
$cache = \Drupal::cache('data');

// Get a cached item.
$item = $cache->get('my_module:computed_result');
if ($item) {
  $data = $item->data;
}
else {
  $data = $this->expensiveComputation();
  $cache->set('my_module:computed_result', $data, Cache::PERMANENT, ['my_module:computed']);
}
```

### CacheBackendInterface Methods

| Method | Description |
|---|---|
| `get($cid)` | Retrieve a single cache item; returns `FALSE` if miss |
| `getMultiple(&$cids)` | Retrieve multiple items at once |
| `set($cid, $data, $expire, $tags)` | Store a cache item with optional expiry and tags |
| `delete($cid)` | Delete a single cache item |
| `deleteMultiple(array $cids)` | Delete multiple items |
| `deleteAll()` | Delete all items in the bin |
| `invalidate($cid)` | Mark an item as invalid (stale but available if `allowInvalid` is used) |
| `invalidateMultiple(array $cids)` | Mark multiple items as invalid |
| `invalidateAll()` | Mark all items in the bin as invalid |
| `removeBin()` | Remove the entire bin |

### Defining a Custom Cache Bin

In your module's `my_module.services.yml`:

```yaml
services:
  cache.my_module:
    class: Drupal\Core\Cache\CacheBackendInterface
    tags:
      - { name: cache.bin }
    factory: ['@cache_factory', 'get']
    arguments: ['my_module']
```

Then use it:

```php
$cache = \Drupal::cache('my_module');
$cache->set('key', $value, Cache::PERMANENT, ['my_module:data']);
```

---

## Cache Metadata Bubbling

Drupal's render system automatically **bubbles** cache metadata from child
render arrays to their parents. This means if a deeply nested element adds a
cache tag or context, the page-level cache entry will include it.

```php
$build = [
  '#cache' => [
    'tags' => ['config:system.site'],
    'contexts' => ['languages:language_interface'],
  ],
  'child' => [
    '#markup' => $user_greeting,
    '#cache' => [
      'contexts' => ['user'],
      'tags' => ['user:' . $uid],
    ],
  ],
];
// After rendering, the page-level cache will include:
// tags: config:system.site, user:{uid}
// contexts: languages:language_interface, user
```

Bubbling ensures correctness without requiring you to manually propagate
metadata to the top. However, it means a single child with `max-age = 0` will
make the entire page uncacheable unless you use **lazy builders** (see
references).

---

## Common Pitfalls

1. **Forgetting `#cache` on blocks.** Custom block plugins must return cache
   metadata. Omitting it causes stale output or, worse, data meant for one
   user shown to another.

2. **Using `max-age = 0` instead of the right cache context.** If output varies
   per user, add `'contexts' => ['user']` instead of disabling caching. This
   lets Drupal cache per-user variants.

3. **Missing list tags.** If your custom code lists entities, add the
   `{entity_type}_list` tag so the listing invalidates when new entities are
   created.

4. **Stale external data.** When pulling data from an external API, assign a
   custom cache tag and invalidate it on cron or webhook to avoid serving
   stale data indefinitely.

5. **Overly broad cache contexts.** Using `user` when `user.permissions` would
   suffice dramatically increases the number of cache variants.

---

## Related Skills

- **drupal-render** — Render arrays, themes, and how `#cache` integrates with the render pipeline.
- **drupal-entities** — Entity cache tags are automatically managed; understand how entity CRUD triggers invalidation.
- **drupal-services** — Dependency injection for cache bin services.
- **drupal-hooks** — Hook into entity presave/insert/delete for custom invalidation logic.
- **drupal-config** — Configuration objects carry their own cache tags (`config:name`).
