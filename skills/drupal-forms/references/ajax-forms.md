# AJAX Forms

## Overview

Drupal's AJAX framework lets forms update portions of the page without a full
reload. There are two approaches:

1. **Simple wrapper replacement** — The `#ajax` callback returns a render array
   and Drupal replaces the wrapper element automatically.
2. **AjaxResponse with commands** — The callback returns an `AjaxResponse`
   object containing one or more commands (replace, prepend, invoke, open
   modal, etc.) for fine-grained control.

---

## Simple AJAX: Wrapper Replacement

The most common pattern. Add `#ajax` to a trigger element, and return the part
of the form to re-render.

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Form;

use Drupal\Core\Form\FormBase;
use Drupal\Core\Form\FormStateInterface;

/**
 * Demonstrates simple AJAX wrapper replacement.
 */
class DependentDropdownForm extends FormBase {

  /**
   * {@inheritdoc}
   */
  public function getFormId(): string {
    return 'my_module_dependent_dropdown';
  }

  /**
   * {@inheritdoc}
   */
  public function buildForm(array $form, FormStateInterface $form_state): array {
    $form['country'] = [
      '#type' => 'select',
      '#title' => $this->t('Country'),
      '#options' => [
        '' => $this->t('- Select -'),
        'us' => $this->t('United States'),
        'ca' => $this->t('Canada'),
        'uk' => $this->t('United Kingdom'),
      ],
      '#ajax' => [
        'callback' => '::updateCityOptions',
        'wrapper' => 'city-wrapper',
        'event' => 'change',
        'progress' => [
          'type' => 'throbber',
          'message' => $this->t('Loading cities...'),
        ],
      ],
    ];

    $country = $form_state->getValue('country') ?? '';

    $form['city'] = [
      '#type' => 'select',
      '#title' => $this->t('City'),
      '#prefix' => '<div id="city-wrapper">',
      '#suffix' => '</div>',
      '#options' => $this->getCityOptions($country),
      '#default_value' => $form_state->getValue('city') ?? '',
      '#validated' => TRUE,
    ];

    $form['actions'] = ['#type' => 'actions'];
    $form['actions']['submit'] = [
      '#type' => 'submit',
      '#value' => $this->t('Submit'),
    ];

    return $form;
  }

  /**
   * AJAX callback: returns the city select element.
   */
  public function updateCityOptions(array &$form, FormStateInterface $form_state): array {
    return $form['city'];
  }

  /**
   * Returns city options based on the selected country.
   */
  protected function getCityOptions(string $country): array {
    $cities = [
      'us' => [
        '' => $this->t('- Select -'),
        'nyc' => $this->t('New York'),
        'la' => $this->t('Los Angeles'),
        'chi' => $this->t('Chicago'),
      ],
      'ca' => [
        '' => $this->t('- Select -'),
        'tor' => $this->t('Toronto'),
        'van' => $this->t('Vancouver'),
        'mtl' => $this->t('Montreal'),
      ],
      'uk' => [
        '' => $this->t('- Select -'),
        'lon' => $this->t('London'),
        'man' => $this->t('Manchester'),
        'edi' => $this->t('Edinburgh'),
      ],
    ];

    return $cities[$country] ?? ['' => $this->t('- Select a country first -')];
  }

  /**
   * {@inheritdoc}
   */
  public function submitForm(array &$form, FormStateInterface $form_state): void {
    $this->messenger()->addStatus($this->t('Selected: @country / @city', [
      '@country' => $form_state->getValue('country'),
      '@city' => $form_state->getValue('city'),
    ]));
  }

}
```

### Key points for wrapper replacement

- The `wrapper` value must match an `id` attribute on a DOM element.
- Use `#prefix`/`#suffix` to add the wrapper div, or `#attributes => ['id' => 'wrapper-id']` on container elements.
- The callback returns the form sub-array (e.g., `$form['city']`). Drupal
  serializes it and replaces the wrapper's innerHTML.
- Set `#validated => TRUE` on dependent elements to prevent "illegal choice"
  errors during AJAX rebuilds.

---

## AjaxResponse with Commands

For advanced use cases, return an `AjaxResponse` with explicit commands.

```php
use Drupal\Core\Ajax\AjaxResponse;
use Drupal\Core\Ajax\HtmlCommand;
use Drupal\Core\Ajax\CssCommand;
use Drupal\Core\Ajax\InvokeCommand;
use Drupal\Core\Ajax\MessageCommand;

/**
 * AJAX callback that returns multiple commands.
 */
public function validateUsername(array &$form, FormStateInterface $form_state): AjaxResponse {
  $response = new AjaxResponse();
  $username = $form_state->getValue('username');

  // Check if username is taken.
  $exists = \Drupal::entityTypeManager()
    ->getStorage('user')
    ->loadByProperties(['name' => $username]);

  if ($exists) {
    $response->addCommand(new HtmlCommand(
      '#username-status',
      $this->t('Username "@name" is already taken.', ['@name' => $username])
    ));
    $response->addCommand(new CssCommand(
      '#username-status',
      ['color' => 'red']
    ));
  }
  else {
    $response->addCommand(new HtmlCommand(
      '#username-status',
      $this->t('Username "@name" is available.', ['@name' => $username])
    ));
    $response->addCommand(new CssCommand(
      '#username-status',
      ['color' => 'green']
    ));
  }

  return $response;
}
```

### Common AJAX commands

| Command | What it does |
|---|---|
| `HtmlCommand($selector, $content)` | Replace innerHTML of matched element |
| `ReplaceCommand($selector, $content)` | Replace the entire matched element |
| `AppendCommand($selector, $content)` | Append content to matched element |
| `PrependCommand($selector, $content)` | Prepend content to matched element |
| `RemoveCommand($selector)` | Remove matched element from the DOM |
| `CssCommand($selector, $styles)` | Apply CSS styles |
| `InvokeCommand($selector, $method, $args)` | Call a jQuery method |
| `MessageCommand($message, $type)` | Show a Drupal status message |
| `RedirectCommand($url)` | Redirect the browser |
| `OpenModalDialogCommand($title, $content, $opts)` | Open a modal dialog |
| `CloseModalDialogCommand()` | Close the current modal |

---

## Modal Dialogs

Open a form or content in a Drupal modal dialog via AJAX.

```php
use Drupal\Core\Ajax\AjaxResponse;
use Drupal\Core\Ajax\OpenModalDialogCommand;

/**
 * AJAX callback that opens a modal.
 */
public function openPreviewModal(array &$form, FormStateInterface $form_state): AjaxResponse {
  $response = new AjaxResponse();

  $content = [
    '#theme' => 'item_list',
    '#title' => $this->t('Preview'),
    '#items' => [
      $this->t('Name: @name', ['@name' => $form_state->getValue('name')]),
      $this->t('Email: @email', ['@email' => $form_state->getValue('email')]),
    ],
  ];

  $response->addCommand(new OpenModalDialogCommand(
    $this->t('Preview'),
    $content,
    ['width' => '600', 'dialogClass' => 'my-module-preview-modal']
  ));

  return $response;
}
```

To use modals, attach the dialog library in your form's `buildForm()`:

```php
$form['#attached']['library'][] = 'core/drupal.dialog.ajax';
```

---

## AJAX Add-More Pattern

Dynamically add and remove form rows.

```php
public function buildForm(array $form, FormStateInterface $form_state): array {
  $num_items = $form_state->get('num_items') ?? 1;
  $form_state->set('num_items', $num_items);

  $form['items'] = [
    '#type' => 'container',
    '#attributes' => ['id' => 'items-wrapper'],
    '#tree' => TRUE,
  ];

  for ($i = 0; $i < $num_items; $i++) {
    $form['items'][$i] = [
      '#type' => 'container',
      '#attributes' => ['class' => ['item-row']],
    ];

    $form['items'][$i]['value'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Item @num', ['@num' => $i + 1]),
      '#default_value' => $form_state->getValue(['items', $i, 'value']) ?? '',
    ];

    if ($i > 0) {
      $form['items'][$i]['remove'] = [
        '#type' => 'submit',
        '#value' => $this->t('Remove'),
        '#name' => 'remove_' . $i,
        '#submit' => ['::removeItem'],
        '#ajax' => [
          'callback' => '::updateItems',
          'wrapper' => 'items-wrapper',
        ],
        '#limit_validation_errors' => [],
      ];
    }
  }

  $form['add_item'] = [
    '#type' => 'submit',
    '#value' => $this->t('Add another item'),
    '#submit' => ['::addItem'],
    '#ajax' => [
      'callback' => '::updateItems',
      'wrapper' => 'items-wrapper',
    ],
    '#limit_validation_errors' => [],
  ];

  $form['actions'] = ['#type' => 'actions'];
  $form['actions']['submit'] = [
    '#type' => 'submit',
    '#value' => $this->t('Save'),
  ];

  return $form;
}

/**
 * Submit handler: add one more item.
 */
public function addItem(array &$form, FormStateInterface $form_state): void {
  $num = $form_state->get('num_items');
  $form_state->set('num_items', $num + 1);
  $form_state->setRebuild();
}

/**
 * Submit handler: remove an item.
 */
public function removeItem(array &$form, FormStateInterface $form_state): void {
  $trigger = $form_state->getTriggeringElement();
  // Extract the index from the button name (e.g., "remove_2").
  $index = (int) str_replace('remove_', '', $trigger['#name']);

  // Get current values and remove the item.
  $items = $form_state->getValue('items') ?? [];
  unset($items[$index]);
  // Re-index the array.
  $form_state->setValue('items', array_values($items));

  $num = $form_state->get('num_items');
  $form_state->set('num_items', $num - 1);
  $form_state->setRebuild();
}

/**
 * AJAX callback: return the items container.
 */
public function updateItems(array &$form, FormStateInterface $form_state): array {
  return $form['items'];
}
```

---

## AJAX File Uploads

Use `managed_file` with AJAX to upload files without a full page reload.
The `managed_file` element has built-in AJAX support.

```php
$form['photo'] = [
  '#type' => 'managed_file',
  '#title' => $this->t('Profile photo'),
  '#upload_location' => 'public://profile-photos/',
  '#upload_validators' => [
    'file_validate_extensions' => ['png jpg jpeg gif'],
    'file_validate_image_resolution' => ['1920x1080', '100x100'],
    'file_validate_size' => [2097152], // 2 MB.
  ],
  '#description' => $this->t('Allowed formats: png, jpg, gif. Max 2 MB.'),
];

// To show a preview after upload, add an AJAX callback
// to a wrapper that re-renders a preview area.
$form['photo_preview'] = [
  '#type' => 'container',
  '#attributes' => ['id' => 'photo-preview-wrapper'],
];

$fid = $form_state->getValue('photo');
if (!empty($fid)) {
  $file = \Drupal\file\Entity\File::load(reset($fid));
  if ($file) {
    $form['photo_preview']['image'] = [
      '#theme' => 'image_style',
      '#style_name' => 'thumbnail',
      '#uri' => $file->getFileUri(),
      '#alt' => $this->t('Preview'),
    ];
  }
}
```

### Making uploaded files permanent

Files uploaded via `managed_file` are temporary by default. Make them permanent
in `submitForm()`:

```php
public function submitForm(array &$form, FormStateInterface $form_state): void {
  $fid = $form_state->getValue('photo');
  if (!empty($fid)) {
    $file = \Drupal\file\Entity\File::load(reset($fid));
    if ($file) {
      $file->setPermanent();
      $file->save();

      // Register file usage so it is not garbage-collected.
      \Drupal::service('file.usage')->add($file, 'my_module', 'user', \Drupal::currentUser()->id());
    }
  }
}
```

---

## Troubleshooting AJAX Forms

| Symptom | Likely cause |
|---|---|
| AJAX does nothing, no network request | Missing `core/drupal.ajax` library or JavaScript error in console. |
| "An AJAX HTTP error occurred" | PHP fatal error in the callback. Check the Drupal watchdog log. |
| "Illegal choice" validation error | A select/radios element's options changed via AJAX but the element was not marked `#validated => TRUE`. |
| Wrapper not found | The `wrapper` ID does not match any element's `id` attribute. Inspect the rendered HTML. |
| Values lost after AJAX rebuild | Defaults not read from `$form_state->getValue()` in `buildForm()`. Always check `$form_state` before config/hardcoded defaults. |
| AJAX works once then breaks | The wrapper element was replaced but the new HTML lacks the wrapper `id`. Ensure the returned element includes the wrapper div. |
