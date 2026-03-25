---
name: drupal-render
description: Use when building Drupal render arrays, working with render elements, implementing hook_theme(), creating template suggestions, or working with Twig templates and lazy builders.
version: 1.0.0
---

# Drupal Render System

## Overview

Drupal's render pipeline transforms structured PHP arrays (render arrays) into HTML
output. Every piece of visible content in Drupal -- page markup, blocks, fields, form
elements -- passes through this pipeline. Understanding render arrays is fundamental
to building and customizing Drupal output.

### Render Pipeline Flow

1. A controller, block plugin, or field formatter returns a **render array**.
2. The **Renderer** service (`renderer`) processes the array recursively.
3. For each element, the renderer resolves `#type` (render element) or `#theme`
   (theme hook) to determine how to convert the array to markup.
4. **Preprocess functions** enrich template variables.
5. **Twig templates** produce the final HTML string.
6. **Cache metadata** (#cache) is bubbled up to the response.
7. **Attached assets** (#attached) are collected for CSS/JS aggregation.

---

## Render Arrays

A render array is a nested associative array where keys starting with `#` are
properties and keys without `#` are child elements.

### Core Properties

| Property | Description |
|---|---|
| `#type` | Render element type (resolved via plugin manager) |
| `#theme` | Theme hook name (resolved via theme registry) |
| `#markup` | Raw HTML markup (automatically XSS-filtered via Xss::filterAdmin) |
| `#plain_text` | Plain text that will be escaped on output (safer than #markup) |
| `#prefix` | HTML prepended before the element |
| `#suffix` | HTML appended after the element |
| `#weight` | Integer controlling rendering order among siblings |
| `#access` | Boolean or AccessResultInterface controlling visibility |
| `#cache` | Cache metadata array (contexts, tags, max-age) |
| `#attached` | Libraries, drupalSettings, or HTML head elements |
| `#lazy_builder` | Callable + args for deferred rendering of uncacheable content |

### Basic Examples

```php
// Simple markup.
$build['greeting'] = [
  '#markup' => '<p>' . t('Hello, world!') . '</p>',
];

// Plain text (auto-escaped).
$build['safe_text'] = [
  '#plain_text' => $user_input,
];

// Using a render element type.
$build['wrapper'] = [
  '#type' => 'container',
  '#attributes' => ['class' => ['my-wrapper']],
  'child' => [
    '#markup' => '<p>Wrapped content</p>',
  ],
];

// Using a theme hook.
$build['custom'] = [
  '#theme' => 'mymodule_card',
  '#title' => t('Card Title'),
  '#content' => $some_render_array,
];

// Weight controls sibling order.
$build['first'] = [
  '#markup' => '<p>Appears first</p>',
  '#weight' => -10,
];
$build['second'] = [
  '#markup' => '<p>Appears second</p>',
  '#weight' => 10,
];

// Access control.
$build['admin_only'] = [
  '#markup' => '<p>Secret content</p>',
  '#access' => $account->hasPermission('administer site configuration'),
];
```

### Cache Metadata

Every render array should declare its cache metadata so Drupal can cache and
invalidate correctly.

```php
$build['content'] = [
  '#markup' => $generated_html,
  '#cache' => [
    // Vary by these contexts.
    'contexts' => ['user.roles', 'languages:language_interface', 'url.query_args'],
    // Invalidate when these tags fire.
    'tags' => ['node:42', 'node_list'],
    // Maximum cache lifetime in seconds (0 = uncacheable, -1 = permanent).
    'max-age' => 3600,
  ],
];
```

---

## Common Render Element Types

Render elements are plugins (implementing `ElementInterface`) registered via
`#type`. Each element defines default properties and a rendering process.

### container

Wraps children in a `<div>` element.

```php
$build['box'] = [
  '#type' => 'container',
  '#attributes' => ['class' => ['highlight-box'], 'id' => 'main-box'],
  'content' => [
    '#markup' => '<p>Inside a container</p>',
  ],
];
```

### details

Collapsible `<details>` element with a `<summary>`.

```php
$build['info'] = [
  '#type' => 'details',
  '#title' => t('More information'),
  '#open' => FALSE,
  'body' => [
    '#markup' => '<p>Hidden until expanded.</p>',
  ],
];
```

### html_tag

Renders any HTML tag.

```php
$build['heading'] = [
  '#type' => 'html_tag',
  '#tag' => 'h2',
  '#value' => t('Section heading'),
  '#attributes' => ['class' => ['section-title']],
];
```

### link

Renders an `<a>` element using a Url object.

```php
use Drupal\Core\Url;

$build['my_link'] = [
  '#type' => 'link',
  '#title' => t('Visit the dashboard'),
  '#url' => Url::fromRoute('mymodule.dashboard'),
  '#attributes' => ['class' => ['button', 'button--primary']],
];
```

### table

Renders an HTML table with headers and rows.

```php
$build['data_table'] = [
  '#type' => 'table',
  '#header' => [t('Name'), t('Status'), t('Operations')],
  '#rows' => [
    ['Alice', t('Active'), t('Edit')],
    ['Bob', t('Inactive'), t('Edit')],
  ],
  '#empty' => t('No records found.'),
  '#attributes' => ['class' => ['my-data-table']],
];
```

### inline_template

Renders an inline Twig template string.

```php
$build['inline'] = [
  '#type' => 'inline_template',
  '#template' => '<span class="{{ class }}">{{ label }}</span>',
  '#context' => [
    'class' => 'badge badge--new',
    'label' => t('New'),
  ],
];
```

See `references/render-elements.md` for a complete catalog of element types and their
properties.

---

## hook_theme() -- Registering Templates

`hook_theme()` registers theme hooks, mapping them to Twig templates and declaring
their variables.

```php
/**
 * Implements hook_theme().
 */
function mymodule_theme(): array {
  return [
    // Theme hook name => definition.
    'mymodule_card' => [
      'variables' => [
        'title' => NULL,
        'content' => NULL,
        'image_url' => NULL,
        'attributes' => [],
      ],
      // Optional: explicit template file name (without .html.twig).
      // Defaults to the hook name with underscores replaced by hyphens.
      'template' => 'mymodule-card',
    ],
    'mymodule_stats_block' => [
      'variables' => [
        'items' => [],
        'total' => 0,
      ],
      // Will look for templates/mymodule-stats-block.html.twig by default.
    ],
  ];
}
```

### Template File Naming

- Template files live in the module's `templates/` directory.
- The file name is the theme hook name with underscores converted to hyphens,
  plus the `.html.twig` extension.
- `mymodule_card` => `templates/mymodule-card.html.twig`
- You can override the name with the `'template'` key.

### OOP Hook (Drupal 11+)

```php
<?php

declare(strict_types=1);

namespace Drupal\mymodule\Hook;

use Drupal\Core\Hook\Attribute\Hook;

class ThemeHooks {

  #[Hook('theme')]
  public function theme(): array {
    return [
      'mymodule_card' => [
        'variables' => [
          'title' => NULL,
          'content' => NULL,
          'image_url' => NULL,
        ],
      ],
    ];
  }

}
```

---

## Twig Templates

Drupal uses Twig as its template engine. Template files use the `.html.twig`
extension.

### Basic Syntax

```twig
{# This is a comment. #}

{# Print a variable (auto-escaped). #}
{{ title }}

{# Execute a statement. #}
{% if content %}
  <div class="card__body">{{ content }}</div>
{% endif %}

{# Loop. #}
{% for item in items %}
  <li>{{ item.label }}</li>
{% endfor %}

{# Set a variable. #}
{% set classes = ['card', 'card--' ~ variant] %}
```

### Drupal-Specific Twig Functions

| Function | Description |
|---|---|
| `url('route_name', {params})` | Generates an absolute URL from a route |
| `path('route_name', {params})` | Generates a relative path from a route |
| `link(text, url)` | Creates an `<a>` element |
| `file_url(uri)` | Converts a file URI (public://...) to a URL |
| `attach_library('module/library')` | Attaches a CSS/JS library |
| `create_attribute()` | Creates an Attributes object in the template |
| `active_theme_path()` | Returns path to the active theme |
| `active_theme()` | Returns the active theme machine name |

### Drupal-Specific Twig Filters

| Filter | Description |
|---|---|
| `t` | Translates a string: `{{ 'Hello'|t }}` |
| `safe_join(', ')` | Joins an array with a separator (markup-safe) |
| `clean_class` | Converts a string to a valid CSS class |
| `clean_id` | Converts a string to a valid HTML ID |
| `without('field_a', 'field_b')` | Renders content excluding specified children |
| `placeholder` | Wraps text in an `<em>` placeholder tag |
| `render` | Forces rendering of a render array to markup |

### Template Example

```twig
{# templates/mymodule-card.html.twig #}
{%
  set classes = [
    'card',
    image_url ? 'card--has-image',
  ]
%}
<article{{ attributes.addClass(classes) }}>
  {% if image_url %}
    <div class="card__image">
      <img src="{{ file_url(image_url) }}" alt="">
    </div>
  {% endif %}
  {% if title %}
    <h3 class="card__title">{{ title }}</h3>
  {% endif %}
  {% if content %}
    <div class="card__content">{{ content }}</div>
  {% endif %}
</article>
```

### Twig Debugging

Enable Twig debugging in `sites/default/services.yml` (or `development.services.yml`):

```yaml
parameters:
  twig.config:
    debug: true
    cache: false
```

Then use `dump()` in templates:

```twig
{# Dump all available variables. #}
{{ dump() }}

{# Dump a specific variable. #}
{{ dump(content) }}
```

With debugging enabled, HTML comments show template suggestions and file paths in
the page source.

---

## Template Suggestions

Template suggestions allow themes and modules to provide more specific template
overrides for particular contexts.

### hook_theme_suggestions_HOOK_alter()

```php
/**
 * Implements hook_theme_suggestions_HOOK_alter() for node templates.
 */
function mymodule_theme_suggestions_node_alter(array &$suggestions, array $variables): void {
  $node = $variables['elements']['#node'];
  $sanitized_view_mode = strtr($variables['elements']['#view_mode'], '.', '_');

  // Add suggestion based on a field value.
  if ($node->hasField('field_layout') && !$node->get('field_layout')->isEmpty()) {
    $layout = $node->get('field_layout')->value;
    $suggestions[] = 'node__' . $node->bundle() . '__' . $layout;
    $suggestions[] = 'node__' . $node->bundle() . '__' . $sanitized_view_mode . '__' . $layout;
  }
}
```

Suggestions are ordered from least specific to most specific. The last matching
template file wins. For a node of type "article" with layout "wide":

- `node.html.twig` (base)
- `node--article.html.twig`
- `node--article--wide.html.twig` (custom suggestion)

---

## Lazy Builders

Lazy builders defer rendering of personalized or uncacheable content, allowing the
rest of the page to be cached normally. They are essential for high-performance sites.

```php
// In a render array, replace dynamic content with a lazy builder.
$build['user_greeting'] = [
  '#lazy_builder' => [
    // Callback: service_id:method or a callable.
    'mymodule.greeting_builder:build',
    // Arguments (must be scalar values only).
    [$greeting_type],
  ],
  // Create a placeholder for BigPipe / dynamic page cache.
  '#create_placeholder' => TRUE,
];
```

The lazy builder service:

```php
<?php

namespace Drupal\mymodule;

use Drupal\Core\Security\TrustedCallbackInterface;
use Drupal\Core\Session\AccountProxyInterface;

class GreetingBuilder implements TrustedCallbackInterface {

  public function __construct(
    protected readonly AccountProxyInterface $currentUser,
  ) {}

  public function build(string $greeting_type): array {
    return [
      '#markup' => '<p>' . t('Hello, @name!', [
        '@name' => $this->currentUser->getDisplayName(),
      ]) . '</p>',
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

Register the service in `mymodule.services.yml`:

```yaml
services:
  mymodule.greeting_builder:
    class: Drupal\mymodule\GreetingBuilder
    autowire: true
```

---

## Preprocess Functions

Preprocess functions prepare and modify variables before they reach Twig templates.

### template_preprocess_HOOK()

```php
/**
 * Prepares variables for mymodule-card templates.
 *
 * Default template: mymodule-card.html.twig.
 *
 * @param array $variables
 *   An associative array containing:
 *   - title: The card title.
 *   - content: The card body content.
 *   - image_url: Optional image URI.
 */
function template_preprocess_mymodule_card(array &$variables): void {
  // Ensure attributes is an Attribute object.
  $variables['attributes'] = new \Drupal\Core\Template\Attribute($variables['attributes'] ?? []);

  // Add a default CSS class.
  $variables['attributes']->addClass('card');

  // Process image URL if provided.
  if (!empty($variables['image_url'])) {
    $variables['attributes']->addClass('card--has-image');
  }
}
```

### Module-specific preprocess

```php
/**
 * Implements hook_preprocess_HOOK() for node templates.
 */
function mymodule_preprocess_node(array &$variables): void {
  /** @var \Drupal\node\NodeInterface $node */
  $node = $variables['node'];

  if ($node->bundle() === 'article') {
    $variables['read_time'] = ceil(str_word_count(strip_tags($node->body->value ?? '')) / 200);
    $variables['#attached']['library'][] = 'mymodule/article-extras';
  }
}
```

---

## Attaching Libraries (#attached)

CSS and JS are attached to render arrays via the `#attached` property. Libraries are
defined in `*.libraries.yml` files.

### Defining a Library

In `mymodule.libraries.yml`:

```yaml
card-styles:
  version: 1.x
  css:
    component:
      css/card.css: {}
  js:
    js/card.js: {}
  dependencies:
    - core/drupal
    - core/once
```

### Attaching in a Render Array

```php
$build['card'] = [
  '#theme' => 'mymodule_card',
  '#title' => t('My Card'),
  '#content' => $body,
  '#attached' => [
    'library' => [
      'mymodule/card-styles',
    ],
    'drupalSettings' => [
      'mymodule' => [
        'animationSpeed' => 300,
      ],
    ],
  ],
];
```

### Attaching in a Twig Template

```twig
{{ attach_library('mymodule/card-styles') }}
<article class="card">
  {{ content }}
</article>
```

---

## Related Skills

- **drupal-hooks** -- Hook system for `hook_theme()`, `hook_preprocess_HOOK()`, and alter hooks.
- **drupal-forms** -- Form API render elements (`#type` => textfield, select, etc.).
- **drupal-cache** -- Cache contexts, tags, max-age, and the cache render pipeline.
- **drupal-plugins** -- Block plugins and other plugins that produce render arrays.
- **drupal-module-development** -- Module structure, services, and library definitions.
