# Config Forms — Deep Dive

## Complex Element Types in Config Forms

Config forms often need more than simple textfields. Here are patterns for
complex element types that store values in Drupal's configuration system.

### Nested fieldsets with multiple values

```php
public function buildForm(array $form, FormStateInterface $form_state): array {
  $config = $this->config('my_module.settings');

  $form['api'] = [
    '#type' => 'details',
    '#title' => $this->t('API Settings'),
    '#open' => TRUE,
  ];

  $form['api']['endpoint'] = [
    '#type' => 'url',
    '#title' => $this->t('API endpoint'),
    '#default_value' => $config->get('api.endpoint'),
    '#required' => TRUE,
  ];

  $form['api']['api_key'] = [
    '#type' => 'textfield',
    '#title' => $this->t('API key'),
    '#default_value' => $config->get('api.api_key'),
    '#required' => TRUE,
  ];

  $form['api']['timeout'] = [
    '#type' => 'number',
    '#title' => $this->t('Request timeout (seconds)'),
    '#default_value' => $config->get('api.timeout') ?? 30,
    '#min' => 5,
    '#max' => 300,
  ];

  $form['notifications'] = [
    '#type' => 'details',
    '#title' => $this->t('Notification Settings'),
    '#open' => FALSE,
  ];

  $form['notifications']['email_recipients'] = [
    '#type' => 'textarea',
    '#title' => $this->t('Email recipients'),
    '#description' => $this->t('One email address per line.'),
    '#default_value' => $config->get('notifications.email_recipients'),
  ];

  $form['notifications']['notify_on'] = [
    '#type' => 'checkboxes',
    '#title' => $this->t('Notify on'),
    '#options' => [
      'create' => $this->t('Content creation'),
      'update' => $this->t('Content update'),
      'delete' => $this->t('Content deletion'),
    ],
    '#default_value' => $config->get('notifications.notify_on') ?? [],
  ];

  return parent::buildForm($form, $form_state);
}
```

### Saving nested config values

```php
public function submitForm(array &$form, FormStateInterface $form_state): void {
  // For elements inside a details/fieldset wrapper, values are nested
  // automatically when the element names match the tree structure.
  $this->config('my_module.settings')
    ->set('api.endpoint', $form_state->getValue('endpoint'))
    ->set('api.api_key', $form_state->getValue('api_key'))
    ->set('api.timeout', (int) $form_state->getValue('timeout'))
    ->set('notifications.email_recipients', $form_state->getValue('email_recipients'))
    ->set('notifications.notify_on', array_filter($form_state->getValue('notify_on')))
    ->save();

  parent::submitForm($form, $form_state);
}
```

> **Note:** When elements are placed inside a `details` or `fieldset` wrapper
> but NOT using `#tree => TRUE`, their values are still at the top level of
> `$form_state->getValues()`. Use `$form_state->getValue('element_name')`
> directly. If you enable `#tree`, values become nested and you must use
> `$form_state->getValue(['parent', 'child'])`.

### Using #tree for truly nested values

```php
$form['cache'] = [
  '#type' => 'details',
  '#title' => $this->t('Cache settings'),
  '#tree' => TRUE,
];

$form['cache']['enabled'] = [
  '#type' => 'checkbox',
  '#title' => $this->t('Enable caching'),
  '#default_value' => $config->get('cache.enabled') ?? TRUE,
];

$form['cache']['max_age'] = [
  '#type' => 'number',
  '#title' => $this->t('Max age (seconds)'),
  '#default_value' => $config->get('cache.max_age') ?? 3600,
  '#min' => 0,
];

// In submitForm, access nested values:
// $cache = $form_state->getValue('cache');
// $cache['enabled'], $cache['max_age']
```

---

## Config Schema for Nested Config

```yaml
my_module.settings:
  type: config_object
  label: 'My Module settings'
  mapping:
    api:
      type: mapping
      label: 'API settings'
      mapping:
        endpoint:
          type: string
          label: 'API endpoint'
        api_key:
          type: string
          label: 'API key'
        timeout:
          type: integer
          label: 'Timeout'
    notifications:
      type: mapping
      label: 'Notification settings'
      mapping:
        email_recipients:
          type: string
          label: 'Email recipients'
        notify_on:
          type: sequence
          label: 'Notify on events'
          sequence:
            type: string
            label: 'Event'
```

---

## Config Form with AJAX Partial Refresh

Refresh part of the form without a full page reload. Useful for previewing
settings or loading dependent options.

```php
use Drupal\Core\Ajax\AjaxResponse;
use Drupal\Core\Ajax\ReplaceCommand;
use Drupal\Core\Form\ConfigFormBase;
use Drupal\Core\Form\FormStateInterface;

class MyModuleSettingsForm extends ConfigFormBase {

  public function getFormId(): string {
    return 'my_module_settings';
  }

  protected function getEditableConfigNames(): array {
    return ['my_module.settings'];
  }

  public function buildForm(array $form, FormStateInterface $form_state): array {
    $config = $this->config('my_module.settings');

    $form['source'] = [
      '#type' => 'select',
      '#title' => $this->t('Data source'),
      '#options' => [
        'database' => $this->t('Database'),
        'api' => $this->t('External API'),
        'file' => $this->t('File import'),
      ],
      '#default_value' => $config->get('source') ?? 'database',
      '#ajax' => [
        'callback' => '::updateSourceOptions',
        'wrapper' => 'source-options-wrapper',
        'event' => 'change',
      ],
    ];

    $source = $form_state->getValue('source') ?? $config->get('source') ?? 'database';

    $form['source_options'] = [
      '#type' => 'container',
      '#attributes' => ['id' => 'source-options-wrapper'],
    ];

    switch ($source) {
      case 'api':
        $form['source_options']['api_url'] = [
          '#type' => 'url',
          '#title' => $this->t('API URL'),
          '#default_value' => $config->get('api_url'),
        ];
        break;

      case 'file':
        $form['source_options']['file_path'] = [
          '#type' => 'textfield',
          '#title' => $this->t('File path'),
          '#default_value' => $config->get('file_path'),
        ];
        break;

      case 'database':
        $form['source_options']['table_name'] = [
          '#type' => 'textfield',
          '#title' => $this->t('Table name'),
          '#default_value' => $config->get('table_name'),
        ];
        break;
    }

    return parent::buildForm($form, $form_state);
  }

  /**
   * AJAX callback to update the source options container.
   */
  public function updateSourceOptions(array &$form, FormStateInterface $form_state): array {
    return $form['source_options'];
  }

  public function submitForm(array &$form, FormStateInterface $form_state): void {
    $source = $form_state->getValue('source');
    $config = $this->config('my_module.settings')
      ->set('source', $source);

    match ($source) {
      'api' => $config->set('api_url', $form_state->getValue('api_url')),
      'file' => $config->set('file_path', $form_state->getValue('file_path')),
      'database' => $config->set('table_name', $form_state->getValue('table_name')),
    };

    $config->save();
    parent::submitForm($form, $form_state);
  }

}
```

### Key AJAX pattern for config forms

1. Add `#ajax` to the trigger element with a `callback` and `wrapper`.
2. The wrapper ID must match an `id` attribute on the element to replace.
3. The callback method returns the portion of `$form` to re-render.
4. In `buildForm()`, always check `$form_state->getValue()` first, then fall
   back to config for defaults — this ensures the rebuilt form reflects the
   user's selection.

---

## Config Dependencies

When your config form depends on another module's configuration (e.g., a
content type must exist), declare that dependency in your install config file:

```yaml
# config/install/my_module.settings.yml
dependencies:
  module:
    - node
  config:
    - node.type.article
```

This ensures that if `node.type.article` is deleted, Drupal warns about your
module's dependency on it.
