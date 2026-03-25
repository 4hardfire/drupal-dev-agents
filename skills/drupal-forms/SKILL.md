---
name: drupal-forms
description: Use when creating Drupal forms, config forms, multi-step wizard forms, AJAX forms, form validation, form submission handlers, or form_alter implementations.
version: 1.0.0
---

# Drupal Forms

## Overview

The Drupal Form API provides a structured, secure way to build HTML forms. All
forms are PHP classes that extend a base class and implement a standard
lifecycle: **build -> validate -> submit**. Drupal automatically handles CSRF
protection, caching, and rebuild logic.

### Base classes

| Base class | When to use |
|---|---|
| `Drupal\Core\Form\FormBase` | Custom forms that do not store configuration (contact forms, search forms, entity operation forms). |
| `Drupal\Core\Form\ConfigFormBase` | Admin settings forms that save values to the configuration system. |
| `Drupal\Core\Form\ConfirmFormBase` | "Are you sure?" confirmation pages before destructive operations. |

All form classes live in `src/Form/` inside your module directory.

---

## ConfigFormBase — Complete Example

A configuration form that stores admin settings.

### File: `modules/custom/my_module/src/Form/MyModuleSettingsForm.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Form;

use Drupal\Core\Form\ConfigFormBase;
use Drupal\Core\Form\FormStateInterface;

/**
 * Configure My Module settings.
 */
class MyModuleSettingsForm extends ConfigFormBase {

  /**
   * {@inheritdoc}
   */
  public function getFormId(): string {
    return 'my_module_settings';
  }

  /**
   * {@inheritdoc}
   */
  protected function getEditableConfigNames(): array {
    return ['my_module.settings'];
  }

  /**
   * {@inheritdoc}
   */
  public function buildForm(array $form, FormStateInterface $form_state): array {
    $config = $this->config('my_module.settings');

    $form['site_name_override'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Site name override'),
      '#default_value' => $config->get('site_name_override'),
      '#maxlength' => 255,
      '#required' => TRUE,
    ];

    $form['items_per_page'] = [
      '#type' => 'number',
      '#title' => $this->t('Items per page'),
      '#default_value' => $config->get('items_per_page') ?? 10,
      '#min' => 1,
      '#max' => 100,
      '#required' => TRUE,
    ];

    $form['display_mode'] = [
      '#type' => 'select',
      '#title' => $this->t('Display mode'),
      '#options' => [
        'grid' => $this->t('Grid'),
        'list' => $this->t('List'),
        'table' => $this->t('Table'),
      ],
      '#default_value' => $config->get('display_mode') ?? 'list',
    ];

    $form['enable_feature'] = [
      '#type' => 'checkbox',
      '#title' => $this->t('Enable special feature'),
      '#default_value' => $config->get('enable_feature') ?? FALSE,
    ];

    return parent::buildForm($form, $form_state);
  }

  /**
   * {@inheritdoc}
   */
  public function validateForm(array &$form, FormStateInterface $form_state): void {
    $items = $form_state->getValue('items_per_page');
    if ($items < 1 || $items > 100) {
      $form_state->setErrorByName('items_per_page', $this->t('Items per page must be between 1 and 100.'));
    }
  }

  /**
   * {@inheritdoc}
   */
  public function submitForm(array &$form, FormStateInterface $form_state): void {
    $this->config('my_module.settings')
      ->set('site_name_override', $form_state->getValue('site_name_override'))
      ->set('items_per_page', (int) $form_state->getValue('items_per_page'))
      ->set('display_mode', $form_state->getValue('display_mode'))
      ->set('enable_feature', (bool) $form_state->getValue('enable_feature'))
      ->save();

    parent::submitForm($form, $form_state);
  }

}
```

### Config schema: `modules/custom/my_module/config/schema/my_module.schema.yml`

```yaml
my_module.settings:
  type: config_object
  label: 'My Module settings'
  mapping:
    site_name_override:
      type: string
      label: 'Site name override'
    items_per_page:
      type: integer
      label: 'Items per page'
    display_mode:
      type: string
      label: 'Display mode'
    enable_feature:
      type: boolean
      label: 'Enable special feature'
```

### Default config: `modules/custom/my_module/config/install/my_module.settings.yml`

```yaml
site_name_override: ''
items_per_page: 10
display_mode: list
enable_feature: false
```

### Route: `modules/custom/my_module/my_module.routing.yml`

```yaml
my_module.settings:
  path: '/admin/config/my-module/settings'
  defaults:
    _form: '\Drupal\my_module\Form\MyModuleSettingsForm'
    _title: 'My Module Settings'
  requirements:
    _permission: 'administer site configuration'
```

### Menu link (optional): `modules/custom/my_module/my_module.links.menu.yml`

```yaml
my_module.settings:
  title: 'My Module Settings'
  route_name: my_module.settings
  parent: system.admin_config
  weight: 100
```

---

## Custom FormBase — Complete Example

A custom form with dependency injection that creates or updates content.

### File: `modules/custom/my_module/src/Form/ContactMessageForm.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Form;

use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Form\FormBase;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\Messenger\MessengerInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Provides a contact message form.
 */
class ContactMessageForm extends FormBase {

  /**
   * Constructs a ContactMessageForm.
   */
  public function __construct(
    protected readonly EntityTypeManagerInterface $entityTypeManager,
    protected readonly MessengerInterface $messenger,
  ) {}

  /**
   * {@inheritdoc}
   */
  public static function create(ContainerInterface $container): static {
    return new static(
      $container->get('entity_type.manager'),
      $container->get('messenger'),
    );
  }

  /**
   * {@inheritdoc}
   */
  public function getFormId(): string {
    return 'my_module_contact_message';
  }

  /**
   * {@inheritdoc}
   */
  public function buildForm(array $form, FormStateInterface $form_state): array {
    $form['name'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Your name'),
      '#required' => TRUE,
      '#maxlength' => 128,
    ];

    $form['email'] = [
      '#type' => 'email',
      '#title' => $this->t('Email address'),
      '#required' => TRUE,
    ];

    $form['subject'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Subject'),
      '#required' => TRUE,
      '#maxlength' => 255,
    ];

    $form['message'] = [
      '#type' => 'textarea',
      '#title' => $this->t('Message'),
      '#required' => TRUE,
      '#rows' => 6,
    ];

    $form['category'] = [
      '#type' => 'select',
      '#title' => $this->t('Category'),
      '#options' => [
        'general' => $this->t('General inquiry'),
        'support' => $this->t('Technical support'),
        'billing' => $this->t('Billing'),
      ],
      '#required' => TRUE,
      '#empty_option' => $this->t('- Select -'),
    ];

    $form['actions'] = [
      '#type' => 'actions',
    ];

    $form['actions']['submit'] = [
      '#type' => 'submit',
      '#value' => $this->t('Send message'),
    ];

    return $form;
  }

  /**
   * {@inheritdoc}
   */
  public function validateForm(array &$form, FormStateInterface $form_state): void {
    $email = $form_state->getValue('email');
    if ($email && !\Drupal::service('email.validator')->isValid($email)) {
      $form_state->setErrorByName('email', $this->t('Please enter a valid email address.'));
    }

    $message = $form_state->getValue('message');
    if (mb_strlen($message) < 10) {
      $form_state->setErrorByName('message', $this->t('Message must be at least 10 characters.'));
    }
  }

  /**
   * {@inheritdoc}
   */
  public function submitForm(array &$form, FormStateInterface $form_state): void {
    // Example: create a node to store the message.
    $storage = $this->entityTypeManager->getStorage('node');
    $node = $storage->create([
      'type' => 'contact_message',
      'title' => $form_state->getValue('subject'),
      'field_sender_name' => $form_state->getValue('name'),
      'field_sender_email' => $form_state->getValue('email'),
      'field_message_body' => $form_state->getValue('message'),
      'field_category' => $form_state->getValue('category'),
    ]);
    $node->save();

    $this->messenger->addStatus($this->t('Your message has been sent. Thank you!'));
    $form_state->setRedirect('<front>');
  }

}
```

### Route

```yaml
my_module.contact_message:
  path: '/contact-us'
  defaults:
    _form: '\Drupal\my_module\Form\ContactMessageForm'
    _title: 'Contact Us'
  requirements:
    _permission: 'access content'
```

---

## Common Form Elements Reference

### Text inputs

```php
// Textfield — single line.
$form['name'] = [
  '#type' => 'textfield',
  '#title' => $this->t('Name'),
  '#default_value' => $default,
  '#maxlength' => 255,
  '#required' => TRUE,
  '#description' => $this->t('Enter your full name.'),
];

// Textarea — multi-line.
$form['bio'] = [
  '#type' => 'textarea',
  '#title' => $this->t('Biography'),
  '#default_value' => $default,
  '#rows' => 5,
];

// Email.
$form['email'] = [
  '#type' => 'email',
  '#title' => $this->t('Email'),
  '#required' => TRUE,
];

// Number.
$form['quantity'] = [
  '#type' => 'number',
  '#title' => $this->t('Quantity'),
  '#min' => 1,
  '#max' => 999,
  '#step' => 1,
  '#default_value' => 1,
];
```

### Selection elements

```php
// Select dropdown.
$form['color'] = [
  '#type' => 'select',
  '#title' => $this->t('Color'),
  '#options' => [
    'red' => $this->t('Red'),
    'blue' => $this->t('Blue'),
    'green' => $this->t('Green'),
  ],
  '#empty_option' => $this->t('- Select -'),
  '#default_value' => 'blue',
];

// Checkboxes — multiple selection, returns array.
$form['toppings'] = [
  '#type' => 'checkboxes',
  '#title' => $this->t('Toppings'),
  '#options' => [
    'cheese' => $this->t('Cheese'),
    'bacon' => $this->t('Bacon'),
    'mushrooms' => $this->t('Mushrooms'),
  ],
  '#default_value' => ['cheese'],
];

// Radios — single selection.
$form['size'] = [
  '#type' => 'radios',
  '#title' => $this->t('Size'),
  '#options' => [
    'small' => $this->t('Small'),
    'medium' => $this->t('Medium'),
    'large' => $this->t('Large'),
  ],
  '#default_value' => 'medium',
  '#required' => TRUE,
];

// Single checkbox.
$form['agree'] = [
  '#type' => 'checkbox',
  '#title' => $this->t('I agree to the terms'),
];
```

### Special elements

```php
// Date.
$form['start_date'] = [
  '#type' => 'date',
  '#title' => $this->t('Start date'),
  '#default_value' => date('Y-m-d'),
];

// Entity autocomplete.
$form['referenced_node'] = [
  '#type' => 'entity_autocomplete',
  '#title' => $this->t('Related article'),
  '#target_type' => 'node',
  '#selection_settings' => [
    'target_bundles' => ['article'],
  ],
];

// Managed file upload.
$form['document'] = [
  '#type' => 'managed_file',
  '#title' => $this->t('Upload document'),
  '#upload_location' => 'public://documents/',
  '#upload_validators' => [
    'file_validate_extensions' => ['pdf doc docx'],
    'file_validate_size' => [25600000],
  ],
];
```

---

## Form States (#states) — Conditional Visibility

The `#states` property lets you show, hide, enable, or require elements based
on the values of other elements, entirely client-side with no AJAX needed.

```php
$form['has_discount'] = [
  '#type' => 'checkbox',
  '#title' => $this->t('Apply a discount code'),
];

// This field only appears when the checkbox above is checked.
$form['discount_code'] = [
  '#type' => 'textfield',
  '#title' => $this->t('Discount code'),
  '#states' => [
    'visible' => [
      ':input[name="has_discount"]' => ['checked' => TRUE],
    ],
    'required' => [
      ':input[name="has_discount"]' => ['checked' => TRUE],
    ],
  ],
];

// Show a container only when a specific select value is chosen.
$form['advanced_options'] = [
  '#type' => 'details',
  '#title' => $this->t('Advanced options'),
  '#states' => [
    'visible' => [
      ':input[name="display_mode"]' => ['value' => 'table'],
    ],
  ],
];
```

### Available state conditions

| State | Trigger |
|---|---|
| `visible` / `invisible` | Show / hide the element |
| `enabled` / `disabled` | Enable / disable the element |
| `required` / `optional` | Make the element required or not |
| `checked` / `unchecked` | Check / uncheck a checkbox |
| `expanded` / `collapsed` | Expand / collapse a details element |

### Available triggers

```php
['checked' => TRUE]       // Checkbox is checked.
['value' => 'foo']        // Input has the value "foo".
['filled' => TRUE]        // Input is not empty.
['empty' => TRUE]         // Input is empty.
```

> **Important:** `#states` visibility is client-side only. Always validate
> server-side that conditionally required fields are actually filled in.

---

## hook_form_alter and hook_form_FORM_ID_alter

Alter existing forms without modifying the originating module.

```php
/**
 * Implements hook_form_FORM_ID_alter() for user_login_form.
 */
function my_module_form_user_login_form_alter(array &$form, FormStateInterface $form_state, string $form_id): void {
  $form['name']['#description'] = t('Enter your email address or username.');

  // Add a custom validation handler.
  $form['#validate'][] = '_my_module_user_login_validate';

  // Add a custom submit handler (runs after default).
  $form['actions']['submit']['#submit'][] = '_my_module_user_login_submit';
}
```

---

## Scaffolding with Drush

Generate form boilerplate quickly:

```bash
# Generate a config form.
ddev drush gen form:config

# Generate a simple (FormBase) form.
ddev drush gen form:simple

# Generate a confirm form.
ddev drush gen form:confirm
```

These generators prompt for module name, form class name, route path, and
permission, then create the PHP class file and routing entry.

---

## Related Skills

- **drupal-config** — Configuration system, config schema, config overrides.
- **drupal-routing** — Route definitions, route parameters, access checking.
- **drupal-services** — Dependency injection, service definitions, service tags.
- **drupal-entities** — Entity types, entity storage, entity queries.
- **drupal-render** — Render arrays, theme hooks, templates.
- **drupal-hooks** — Hook system, hook implementations, alter hooks.
