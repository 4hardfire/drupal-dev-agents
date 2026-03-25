# Render Elements Reference

This is a catalog of core render element types available via the `#type` property
in Drupal render arrays. Each element is a plugin extending `RenderElement` or
`FormElement`.

---

## Form Elements

Form elements extend `FormElement` and are typically used inside form render arrays
returned by `buildForm()`. They can also appear in non-form render arrays when
combined with the form API.

### textfield

Single-line text input.

```php
$element['name'] = [
  '#type' => 'textfield',
  '#title' => t('Name'),
  '#default_value' => $config->get('name'),
  '#maxlength' => 128,
  '#size' => 60,
  '#required' => TRUE,
  '#description' => t('Enter your full name.'),
  '#placeholder' => t('John Doe'),
  '#attributes' => ['autocomplete' => 'name'],
];
```

### textarea

Multi-line text input.

```php
$element['description'] = [
  '#type' => 'textarea',
  '#title' => t('Description'),
  '#default_value' => $config->get('description'),
  '#rows' => 5,
  '#cols' => 60,
  '#resizable' => 'vertical', // 'none', 'vertical', 'both'.
  '#description' => t('Provide a brief description.'),
];
```

### select

Dropdown select list.

```php
$element['color'] = [
  '#type' => 'select',
  '#title' => t('Favorite color'),
  '#options' => [
    'red' => t('Red'),
    'green' => t('Green'),
    'blue' => t('Blue'),
  ],
  '#default_value' => 'blue',
  '#empty_option' => t('- Select -'),
  '#empty_value' => '',
  '#required' => TRUE,
  '#multiple' => FALSE,
];
```

### checkboxes

Group of checkboxes for multiple selections.

```php
$element['toppings'] = [
  '#type' => 'checkboxes',
  '#title' => t('Toppings'),
  '#options' => [
    'cheese' => t('Cheese'),
    'pepperoni' => t('Pepperoni'),
    'mushrooms' => t('Mushrooms'),
  ],
  '#default_value' => ['cheese', 'mushrooms'],
];
```

### radios

Group of radio buttons for single selection.

```php
$element['size'] = [
  '#type' => 'radios',
  '#title' => t('Size'),
  '#options' => [
    'small' => t('Small'),
    'medium' => t('Medium'),
    'large' => t('Large'),
  ],
  '#default_value' => 'medium',
];
```

### checkbox

Single checkbox (boolean).

```php
$element['agree'] = [
  '#type' => 'checkbox',
  '#title' => t('I agree to the terms'),
  '#default_value' => FALSE,
];
```

### number

Numeric input with optional min/max/step.

```php
$element['quantity'] = [
  '#type' => 'number',
  '#title' => t('Quantity'),
  '#min' => 1,
  '#max' => 100,
  '#step' => 1,
  '#default_value' => 1,
];
```

### email

Email input with built-in validation.

```php
$element['email'] = [
  '#type' => 'email',
  '#title' => t('Email address'),
  '#default_value' => $config->get('email'),
  '#required' => TRUE,
];
```

### password

Password input (value never sent back to client).

```php
$element['pass'] = [
  '#type' => 'password',
  '#title' => t('Password'),
  '#size' => 25,
  '#required' => TRUE,
];
```

### hidden

Hidden form element.

```php
$element['entity_id'] = [
  '#type' => 'hidden',
  '#value' => $entity->id(),
];
```

### managed_file

File upload widget with AJAX upload support.

```php
$element['document'] = [
  '#type' => 'managed_file',
  '#title' => t('Upload document'),
  '#upload_location' => 'public://documents/',
  '#upload_validators' => [
    'FileExtension' => ['extensions' => 'pdf doc docx'],
    'FileSizeLimit' => ['fileLimit' => 10 * 1024 * 1024],
  ],
  '#default_value' => $config->get('document') ? [$config->get('document')] : [],
];
```

### entity_autocomplete

Autocomplete widget for referencing entities.

```php
$element['author'] = [
  '#type' => 'entity_autocomplete',
  '#title' => t('Author'),
  '#target_type' => 'user',
  '#selection_handler' => 'default',
  '#selection_settings' => [
    'include_anonymous' => FALSE,
  ],
  '#default_value' => $user_entity,
  '#required' => TRUE,
];
```

### date

HTML5 date picker.

```php
$element['start_date'] = [
  '#type' => 'date',
  '#title' => t('Start date'),
  '#default_value' => date('Y-m-d'),
];
```

---

## Display Elements

Display elements extend `RenderElement` and produce non-form HTML output.

### container

Wraps children in a `<div>`.

```php
$element['wrapper'] = [
  '#type' => 'container',
  '#attributes' => ['class' => ['content-wrapper']],
  'children' => [
    '#markup' => '<p>Wrapped content</p>',
  ],
];
```

### details

Collapsible `<details>` / `<summary>` element.

```php
$element['advanced'] = [
  '#type' => 'details',
  '#title' => t('Advanced settings'),
  '#open' => FALSE,
  '#description' => t('Configure advanced options.'),
  '#group' => 'tabs', // Optional: group into vertical tabs.
  'setting_a' => [
    '#type' => 'textfield',
    '#title' => t('Setting A'),
  ],
];
```

### fieldset

Groups form elements with a visible legend.

```php
$element['contact_info'] = [
  '#type' => 'fieldset',
  '#title' => t('Contact information'),
  '#attributes' => ['class' => ['contact-fieldset']],
  'phone' => [
    '#type' => 'tel',
    '#title' => t('Phone'),
  ],
];
```

### table

Renders an HTML `<table>`.

```php
$element['results'] = [
  '#type' => 'table',
  '#header' => [
    t('Title'),
    t('Author'),
    ['data' => t('Date'), 'class' => ['date-column']],
  ],
  '#rows' => [
    [
      'Article one',
      'Alice',
      '2025-01-15',
    ],
  ],
  '#empty' => t('No results found.'),
  '#sticky' => TRUE,
  '#responsive' => TRUE,
];
```

**Draggable table** (for reordering):

```php
$element['items'] = [
  '#type' => 'table',
  '#header' => [t('Label'), t('Weight')],
  '#tabledrag' => [
    [
      'action' => 'order',
      'relationship' => 'sibling',
      'group' => 'item-weight',
    ],
  ],
];

foreach ($items as $id => $item) {
  $element['items'][$id]['#attributes']['class'][] = 'draggable';
  $element['items'][$id]['#weight'] = $item['weight'];
  $element['items'][$id]['label'] = [
    '#plain_text' => $item['label'],
  ];
  $element['items'][$id]['weight'] = [
    '#type' => 'weight',
    '#title' => t('Weight for @title', ['@title' => $item['label']]),
    '#title_display' => 'invisible',
    '#default_value' => $item['weight'],
    '#attributes' => ['class' => ['item-weight']],
  ];
}
```

### html_tag

Renders any arbitrary HTML tag.

```php
// A <div> with custom data attributes.
$element['banner'] = [
  '#type' => 'html_tag',
  '#tag' => 'div',
  '#value' => t('Welcome!'),
  '#attributes' => [
    'class' => ['site-banner'],
    'data-role' => 'banner',
  ],
];

// A self-closing <meta> tag.
$element['meta'] = [
  '#type' => 'html_tag',
  '#tag' => 'meta',
  '#attributes' => [
    'name' => 'robots',
    'content' => 'noindex, nofollow',
  ],
];
```

### link

Renders an anchor element. Requires a `Url` object.

```php
use Drupal\Core\Url;

// Internal route link.
$element['dashboard_link'] = [
  '#type' => 'link',
  '#title' => t('Go to dashboard'),
  '#url' => Url::fromRoute('mymodule.dashboard', ['user' => $uid]),
  '#attributes' => ['class' => ['button']],
];

// External link.
$element['external_link'] = [
  '#type' => 'link',
  '#title' => t('Drupal.org'),
  '#url' => Url::fromUri('https://www.drupal.org'),
  '#attributes' => ['target' => '_blank', 'rel' => 'noopener'],
];
```

### image

Renders an `<img>` tag.

```php
$element['logo'] = [
  '#type' => 'image',
  '#uri' => 'public://logos/site-logo.png',
  '#alt' => t('Site logo'),
  '#title' => t('Home'),
  '#width' => 200,
  '#height' => 80,
  '#attributes' => ['class' => ['site-logo']],
];
```

### pager

Renders the pager navigation for a paged query.

```php
$element['pager'] = [
  '#type' => 'pager',
  '#quantity' => 5, // Number of page links to show.
];
```

### status_messages

Renders Drupal status messages (info, warning, error).

```php
$element['messages'] = [
  '#type' => 'status_messages',
];
```

---

## Page-Level Elements

These are used internally by Drupal's page rendering system. You rarely create these
directly, but understanding them helps when working with page templates.

### html

The outermost page wrapper. Renders the `<html>` tag. Provides variables to
`html.html.twig`: `html_attributes`, `head_title`, `styles`, `scripts`, `page`.

### page

The page-level element. Renders the `<body>` content area. Provides variables to
`page.html.twig`: `page.header`, `page.content`, `page.sidebar_first`, etc.

### region

Represents a theme region. Contains blocks assigned to that region.

---

## Special Elements

### inline_template

Renders a Twig template string inline. Useful for small, dynamic templates that do
not warrant a separate `.html.twig` file.

```php
$element['badge'] = [
  '#type' => 'inline_template',
  '#template' => '<span class="badge badge--{{ type|clean_class }}">{{ label }}</span>',
  '#context' => [
    'type' => $badge_type,
    'label' => $badge_label,
  ],
];
```

### processed_text

Renders text through a text format's filters (e.g., basic_html, full_html).

```php
$element['body'] = [
  '#type' => 'processed_text',
  '#text' => $raw_text,
  '#format' => 'basic_html',
];
```

### view

Embeds a Views display. Requires the Views module.

```php
$element['recent_articles'] = [
  '#type' => 'view',
  '#name' => 'content_recent', // View machine name.
  '#display_id' => 'block_1',  // Display ID.
  '#arguments' => ['article'],  // Contextual filters.
  '#embed' => TRUE,
];
```

### more_link

Renders a "More" link, commonly used at the bottom of listing blocks.

```php
$element['more'] = [
  '#type' => 'more_link',
  '#url' => Url::fromRoute('view.content_recent.page_1'),
  '#title' => t('View all articles'),
];
```

### dropbutton

Renders a button with a dropdown list of actions.

```php
$element['operations'] = [
  '#type' => 'dropbutton',
  '#links' => [
    'edit' => [
      'title' => t('Edit'),
      'url' => Url::fromRoute('entity.node.edit_form', ['node' => $nid]),
    ],
    'delete' => [
      'title' => t('Delete'),
      'url' => Url::fromRoute('entity.node.delete_form', ['node' => $nid]),
    ],
  ],
];
```
