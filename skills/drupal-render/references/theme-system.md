# Theme System Reference

Complete reference for Drupal's theme layer: hook_theme() registration, template
naming, Twig template inheritance, the Attributes object, and asset libraries.

---

## hook_theme() Registration

`hook_theme()` tells Drupal's theme registry about your templates and their
variables. Without this registration, a `#theme` value in a render array will not
resolve.

### Full Syntax

```php
/**
 * Implements hook_theme().
 */
function mymodule_theme(): array {
  return [
    // Theme hook name.
    'mymodule_card' => [
      // Variables passed to the template. Keys become Twig variable names.
      // Values are defaults (usually NULL or empty arrays).
      'variables' => [
        'title' => NULL,
        'content' => NULL,
        'image' => NULL,
        'footer' => NULL,
        'attributes' => [],
      ],
      // Template file name without .html.twig extension.
      // If omitted, Drupal derives it from the hook name:
      //   mymodule_card => mymodule-card.html.twig
      'template' => 'mymodule-card',
      // Path to the template directory (relative to module root).
      // Defaults to 'templates'.
      'path' => 'templates',
    ],

    // A theme hook that wraps an existing render element.
    // Use 'render element' instead of 'variables' when the template
    // receives the full render array as a single variable.
    'mymodule_wrapper' => [
      'render element' => 'element',
    ],
  ];
}
```

### variables vs render element

| Key | Usage | Twig receives |
|---|---|---|
| `'variables'` | Custom template with named variables | Each key as a separate Twig variable |
| `'render element'` | Wrapping an existing render element | A single variable (the element array) |

Use `'variables'` for module-defined display templates. Use `'render element'` when
you are providing a template for a render element type or want the entire element
array passed through.

### Default template_preprocess_HOOK()

When you register a theme hook, you can define a `template_preprocess_HOOK()`
function in your `.module` file. Drupal calls it automatically before rendering.

```php
/**
 * Prepares variables for mymodule-card templates.
 *
 * Default template: mymodule-card.html.twig.
 */
function template_preprocess_mymodule_card(array &$variables): void {
  // Convert attributes array to an Attribute object if not already.
  if (is_array($variables['attributes'])) {
    $variables['attributes'] = new \Drupal\Core\Template\Attribute($variables['attributes']);
  }

  // Add computed variables.
  $variables['has_image'] = !empty($variables['image']);
}
```

---

## Template Naming Conventions

### Automatic Derivation

Drupal converts the theme hook name to a template filename by replacing underscores
with hyphens:

| Theme hook | Template file |
|---|---|
| `node` | `node.html.twig` |
| `node__article` | `node--article.html.twig` |
| `block__system_main_block` | `block--system-main-block.html.twig` |
| `mymodule_card` | `mymodule-card.html.twig` |
| `field__body` | `field--body.html.twig` |

### Double-Dash Convention for Suggestions

Template suggestions use double dashes (`--`) to indicate specificity:

- `node.html.twig` -- all nodes
- `node--article.html.twig` -- article nodes
- `node--article--full.html.twig` -- article nodes in full view mode
- `node--42.html.twig` -- node with ID 42

### Where Templates Are Discovered

1. The active theme's `templates/` directory (highest priority).
2. Base themes' `templates/` directories (in inheritance order).
3. The module that registered the theme hook via `hook_theme()`.

---

## Twig Template Inheritance

Drupal's Twig implementation supports the full Twig inheritance model.

### extends

Creates a child template that inherits from a parent and overrides specific blocks.

```twig
{# Parent: templates/layout-base.html.twig #}
<div class="layout">
  <header>{% block header %}Default header{% endblock %}</header>
  <main>{% block content %}{% endblock %}</main>
  <footer>{% block footer %}Default footer{% endblock %}</footer>
</div>
```

```twig
{# Child: templates/layout-article.html.twig #}
{% extends "layout-base.html.twig" %}

{% block content %}
  <article>{{ article_content }}</article>
{% endblock %}

{% block footer %}
  {{ parent() }}
  <p>Article-specific footer content</p>
{% endblock %}
```

**Note**: `extends` requires the child template to contain only `{% block %}` tags at
the top level. The `parent()` function renders the parent block's content.

### include

Embeds another template's rendered output. The included template has access to the
variables of the current context (plus any explicitly passed).

```twig
{# Include with current context. #}
{% include 'mymodule-card.html.twig' %}

{# Include with specific variables only. #}
{% include 'mymodule-card.html.twig' with {'title': item.title, 'content': item.body} only %}

{# Conditional include with fallback. #}
{% include ['node--' ~ node.bundle ~ '.html.twig', 'node.html.twig'] %}
```

### embed

Combines `include` and `extends` -- includes a template and overrides its blocks.

```twig
{% embed 'mymodule-card.html.twig' with {'title': article.title} %}
  {% block card_footer %}
    <a href="{{ path('entity.node.canonical', {'node': article.id}) }}">
      {{ 'Read more'|t }}
    </a>
  {% endblock %}
{% endembed %}
```

### macro

Defines reusable template functions (like partials).

```twig
{% macro icon(name, size) %}
  <svg class="icon icon--{{ name }}" width="{{ size|default(24) }}" height="{{ size|default(24) }}">
    <use xlink:href="#icon-{{ name }}"></use>
  </svg>
{% endmacro %}

{# Usage within the same template. #}
{{ _self.icon('search', 16) }}

{# Import from another template. #}
{% from 'mymodule-macros.html.twig' import icon %}
{{ icon('arrow-right') }}
```

---

## Drupal-Specific Twig Extensions

### The Attributes Object

Drupal passes an `Attribute` object (`\Drupal\Core\Template\Attribute`) to most
templates as the `attributes` variable. This object provides methods for
manipulating HTML attributes cleanly.

#### Methods

```twig
{# Add CSS classes. #}
<div{{ attributes.addClass('new-class', 'another-class') }}>

{# Remove CSS classes. #}
<div{{ attributes.removeClass('unwanted-class') }}>

{# Check for a class. #}
{% if attributes.hasClass('active') %}

{# Set an attribute. #}
<div{{ attributes.setAttribute('data-id', node.id) }}>

{# Remove an attribute. #}
<div{{ attributes.removeAttribute('role') }}>

{# Print all attributes (the most common usage). #}
<div{{ attributes }}>
```

#### Creating Attributes in Templates

```twig
{# Create a new Attribute object for a child element. #}
{% set wrapper_attributes = create_attribute() %}
{% set wrapper_attributes = wrapper_attributes.addClass('inner-wrapper') %}
<div{{ wrapper_attributes }}>
  {{ content }}
</div>
```

#### Splitting Attributes

A common pattern is to use one set of attributes on the outer element and additional
attributes on an inner element:

```twig
<article{{ attributes }}>
  {% set content_attributes = create_attribute({'class': ['node__content']}) %}
  <div{{ content_attributes }}>
    {{ content }}
  </div>
</article>
```

### The without Filter

Renders content while excluding specific child elements. Commonly used in node
templates:

```twig
{# Render all fields except the ones we handle separately. #}
<article{{ attributes }}>
  <header>
    {{ content.field_image }}
    <h2>{{ label }}</h2>
  </header>
  <div class="node__body">
    {{ content|without('field_image', 'field_tags') }}
  </div>
  <footer>
    {{ content.field_tags }}
  </footer>
</article>
```

### Translation in Templates

```twig
{# Simple translation. #}
<h1>{{ 'Welcome'|t }}</h1>

{# Translation with placeholders. #}
{% set count = items|length %}
{% trans %}
  1 item found.
{% plural count %}
  {{ count }} items found.
{% endtrans %}
```

---

## #theme vs #type

Understanding the difference between `#theme` and `#type` is critical.

| Property | Resolved by | Purpose |
|---|---|---|
| `#type` | Render element plugin manager | Defines element structure, processing, and default properties. The element plugin may internally set `#theme` or `#theme_wrappers`. |
| `#theme` | Theme registry (hook_theme) | Maps directly to a Twig template. Passes variables to the template. |

### How They Interact

When a render array has `#type`, Drupal:

1. Loads the render element plugin for that type.
2. The plugin's `getInfo()` method provides defaults (including `#theme` or
   `#theme_wrappers`).
3. The element is processed (`#pre_render`, `#process` callbacks).
4. The resulting `#theme` (if set) determines which template is used.

When a render array has `#theme` directly (without `#type`):

1. Drupal looks up the theme hook in the theme registry.
2. Preprocess functions run.
3. The associated template is rendered with the prepared variables.

### Example: #type Sets #theme Internally

The `'table'` render element (`#type => 'table'`) internally sets `#theme => 'table'`,
which maps to `table.html.twig`. You do not need to set both.

```php
// This is sufficient -- #type handles #theme internally.
$build['my_table'] = [
  '#type' => 'table',
  '#header' => $header,
  '#rows' => $rows,
];
```

---

## Asset Libraries (*.libraries.yml)

All CSS and JS in Drupal must be loaded through the library system. Direct `<link>`
or `<script>` tags in templates are strongly discouraged.

### Library Definition

In `mymodule.libraries.yml`:

```yaml
card:
  version: 1.x
  css:
    # CSS categories (in load order): base, layout, component, state, theme.
    component:
      css/card.css: {}
    theme:
      css/card-theme.css: {}
  js:
    js/card.js: { minified: true }
  dependencies:
    - core/drupal
    - core/drupalSettings
    - core/once

admin-styles:
  version: 1.x
  css:
    theme:
      css/admin.css: {}

global:
  version: 1.x
  css:
    base:
      css/reset.css: { weight: -100 }
    theme:
      css/global.css: {}
  js:
    js/global.js: {}
  dependencies:
    - core/drupal
```

### CSS Categories and Load Order

Libraries define CSS within category keys that control load order:

| Category | Purpose | Weight |
|---|---|---|
| `base` | CSS resets, normalize, element defaults | -200 |
| `layout` | Grid systems, page structure | -100 |
| `component` | Self-contained UI components | 0 |
| `state` | States like `.is-active`, `.is-collapsed` | 100 |
| `theme` | Visual styling, colors, typography | 200 |

### Attaching Libraries

**In PHP (render array):**

```php
$build['#attached']['library'][] = 'mymodule/card';
```

**In Twig:**

```twig
{{ attach_library('mymodule/card') }}
```

**Globally for a theme** (in `mytheme.info.yml`):

```yaml
libraries:
  - mytheme/global
```

**Conditionally via hook:**

```php
/**
 * Implements hook_page_attachments().
 */
function mymodule_page_attachments(array &$attachments): void {
  $route = \Drupal::routeMatch()->getRouteName();
  if ($route === 'mymodule.dashboard') {
    $attachments['#attached']['library'][] = 'mymodule/admin-styles';
  }
}
```

### Overriding and Extending Libraries

Themes can override or extend module libraries in `mytheme.info.yml`:

```yaml
libraries-override:
  # Remove a library entirely.
  classy/node: false

  # Replace a specific CSS file within a library.
  core/drupal.vertical-tabs:
    css:
      component:
        misc/vertical-tabs.css: css/vertical-tabs-override.css

libraries-extend:
  # Add CSS/JS to an existing library whenever it is loaded.
  core/drupal.dialog:
    - mytheme/dialog-extras
```

### Passing Data to JavaScript (drupalSettings)

```php
$build['#attached']['drupalSettings']['mymodule'] = [
  'apiEndpoint' => '/api/v1/data',
  'refreshInterval' => 5000,
];
```

Access in JavaScript:

```js
(function (Drupal, drupalSettings, once) {
  'use strict';

  Drupal.behaviors.mymoduleRefresh = {
    attach(context) {
      once('mymodule-refresh', '[data-mymodule-refresh]', context).forEach((el) => {
        const endpoint = drupalSettings.mymodule.apiEndpoint;
        // Use endpoint...
      });
    },
  };
})(Drupal, drupalSettings, once);
```

---

## Theme Suggestions Workflow

### Built-in Suggestions

Drupal core provides default template suggestions for most elements. For nodes:

- `node.html.twig`
- `node--{bundle}.html.twig`
- `node--{view_mode}.html.twig`
- `node--{bundle}--{view_mode}.html.twig`
- `node--{nid}.html.twig`

### Adding Custom Suggestions

```php
/**
 * Implements hook_theme_suggestions_HOOK_alter() for block templates.
 */
function mytheme_theme_suggestions_block_alter(array &$suggestions, array $variables): void {
  // Add a suggestion based on the block's region.
  if (!empty($variables['elements']['#configuration']['region'])) {
    $region = $variables['elements']['#configuration']['region'];
    $suggestions[] = 'block__region__' . $region;
  }

  // Add a suggestion based on the providing module.
  if (isset($variables['elements']['#configuration']['provider'])) {
    $provider = $variables['elements']['#configuration']['provider'];
    $suggestions[] = 'block__' . $provider;
  }
}
```

### Suggestion Ordering

Suggestions are ordered from least to most specific. Drupal uses the last suggestion
for which a template file exists. When adding suggestions, append more specific ones
after less specific ones:

```php
// Less specific -- add first.
$suggestions[] = 'node__' . $node->bundle();
// More specific -- add after.
$suggestions[] = 'node__' . $node->bundle() . '__' . $custom_variant;
```

### Debugging Suggestions

With Twig debugging enabled, HTML comments show available suggestions:

```html
<!-- THEME DEBUG -->
<!-- THEME HOOK: 'node' -->
<!-- FILE NAME SUGGESTIONS:
   * node--article--full.html.twig
   * node--article.html.twig
   * node--1.html.twig
   x node.html.twig
-->
<!-- BEGIN OUTPUT from 'core/themes/classy/templates/content/node.html.twig' -->
```

The `x` marks the currently active template. Suggestions above it are available
but no matching template file was found.
