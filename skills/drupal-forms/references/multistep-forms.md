# Multi-Step Wizard Forms

## Overview

Multi-step forms in Drupal use `$form_state` storage to persist data across
steps. The form class tracks the current step, shows/hides fields accordingly,
and aggregates all values on final submission.

Key methods:

- `$form_state->set('key', $value)` — store a value across rebuilds.
- `$form_state->get('key')` — retrieve a stored value.
- `$form_state->setRebuild()` — tell Drupal to rebuild the form instead of
  redirecting after submit.

---

## Complete Working Example

### File: `modules/custom/my_module/src/Form/RegistrationWizardForm.php`

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Form;

use Drupal\Core\Form\FormBase;
use Drupal\Core\Form\FormStateInterface;

/**
 * A multi-step registration wizard form.
 */
class RegistrationWizardForm extends FormBase {

  /**
   * {@inheritdoc}
   */
  public function getFormId(): string {
    return 'my_module_registration_wizard';
  }

  /**
   * {@inheritdoc}
   */
  public function buildForm(array $form, FormStateInterface $form_state): array {
    // Initialize step on first load.
    if (!$form_state->get('step')) {
      $form_state->set('step', 1);
    }

    $step = $form_state->get('step');

    $form['#prefix'] = '<div id="wizard-wrapper">';
    $form['#suffix'] = '</div>';

    // Progress indicator.
    $form['progress'] = [
      '#markup' => $this->t('Step @current of @total', [
        '@current' => $step,
        '@total' => 3,
      ]),
      '#prefix' => '<div class="wizard-progress">',
      '#suffix' => '</div>',
    ];

    match ($step) {
      1 => $this->buildStepOne($form, $form_state),
      2 => $this->buildStepTwo($form, $form_state),
      3 => $this->buildStepThree($form, $form_state),
    };

    // Navigation buttons.
    $form['actions'] = [
      '#type' => 'actions',
    ];

    if ($step > 1) {
      $form['actions']['back'] = [
        '#type' => 'submit',
        '#value' => $this->t('Back'),
        '#submit' => ['::goBack'],
        // Skip validation when going back.
        '#limit_validation_errors' => [],
        '#ajax' => [
          'callback' => '::ajaxRebuild',
          'wrapper' => 'wizard-wrapper',
        ],
      ];
    }

    if ($step < 3) {
      $form['actions']['next'] = [
        '#type' => 'submit',
        '#value' => $this->t('Next'),
        '#submit' => ['::goNext'],
        '#ajax' => [
          'callback' => '::ajaxRebuild',
          'wrapper' => 'wizard-wrapper',
        ],
      ];
    }

    if ($step === 3) {
      $form['actions']['submit'] = [
        '#type' => 'submit',
        '#value' => $this->t('Complete registration'),
      ];
    }

    return $form;
  }

  /**
   * Step 1: Personal information.
   */
  protected function buildStepOne(array &$form, FormStateInterface $form_state): void {
    $form['first_name'] = [
      '#type' => 'textfield',
      '#title' => $this->t('First name'),
      '#required' => TRUE,
      '#default_value' => $form_state->get('first_name') ?? '',
    ];

    $form['last_name'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Last name'),
      '#required' => TRUE,
      '#default_value' => $form_state->get('last_name') ?? '',
    ];

    $form['email'] = [
      '#type' => 'email',
      '#title' => $this->t('Email address'),
      '#required' => TRUE,
      '#default_value' => $form_state->get('email') ?? '',
    ];

    $form['phone'] = [
      '#type' => 'tel',
      '#title' => $this->t('Phone number'),
      '#default_value' => $form_state->get('phone') ?? '',
    ];
  }

  /**
   * Step 2: Account preferences.
   */
  protected function buildStepTwo(array &$form, FormStateInterface $form_state): void {
    $form['username'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Preferred username'),
      '#required' => TRUE,
      '#default_value' => $form_state->get('username') ?? '',
      '#maxlength' => 60,
    ];

    $form['plan'] = [
      '#type' => 'radios',
      '#title' => $this->t('Subscription plan'),
      '#options' => [
        'free' => $this->t('Free'),
        'basic' => $this->t('Basic — $9/month'),
        'premium' => $this->t('Premium — $29/month'),
      ],
      '#required' => TRUE,
      '#default_value' => $form_state->get('plan') ?? 'free',
    ];

    $form['newsletter'] = [
      '#type' => 'checkbox',
      '#title' => $this->t('Subscribe to newsletter'),
      '#default_value' => $form_state->get('newsletter') ?? TRUE,
    ];
  }

  /**
   * Step 3: Review and confirm.
   */
  protected function buildStepThree(array &$form, FormStateInterface $form_state): void {
    $form['review'] = [
      '#theme' => 'item_list',
      '#title' => $this->t('Review your information'),
      '#items' => [
        $this->t('Name: @first @last', [
          '@first' => $form_state->get('first_name'),
          '@last' => $form_state->get('last_name'),
        ]),
        $this->t('Email: @email', ['@email' => $form_state->get('email')]),
        $this->t('Phone: @phone', ['@phone' => $form_state->get('phone') ?: $this->t('Not provided')]),
        $this->t('Username: @username', ['@username' => $form_state->get('username')]),
        $this->t('Plan: @plan', ['@plan' => $form_state->get('plan')]),
      ],
    ];

    $form['agree_terms'] = [
      '#type' => 'checkbox',
      '#title' => $this->t('I agree to the terms and conditions'),
      '#required' => TRUE,
    ];
  }

  /**
   * Submit handler for the "Next" button.
   */
  public function goNext(array &$form, FormStateInterface $form_state): void {
    $this->saveCurrentStepValues($form_state);
    $step = $form_state->get('step');
    $form_state->set('step', $step + 1);
    $form_state->setRebuild();
  }

  /**
   * Submit handler for the "Back" button.
   */
  public function goBack(array &$form, FormStateInterface $form_state): void {
    $step = $form_state->get('step');
    $form_state->set('step', $step - 1);
    $form_state->setRebuild();
  }

  /**
   * AJAX callback: return the rebuilt form wrapper.
   */
  public function ajaxRebuild(array &$form, FormStateInterface $form_state): array {
    return $form;
  }

  /**
   * Persist the current step's field values into form_state storage.
   */
  protected function saveCurrentStepValues(FormStateInterface $form_state): void {
    $step = $form_state->get('step');

    match ($step) {
      1 => $this->saveValues($form_state, ['first_name', 'last_name', 'email', 'phone']),
      2 => $this->saveValues($form_state, ['username', 'plan', 'newsletter']),
    };
  }

  /**
   * Helper to save a list of field values into form_state storage.
   */
  protected function saveValues(FormStateInterface $form_state, array $keys): void {
    foreach ($keys as $key) {
      $form_state->set($key, $form_state->getValue($key));
    }
  }

  /**
   * {@inheritdoc}
   */
  public function validateForm(array &$form, FormStateInterface $form_state): void {
    $step = $form_state->get('step');

    if ($step === 1) {
      $email = $form_state->getValue('email');
      if ($email && !\Drupal::service('email.validator')->isValid($email)) {
        $form_state->setErrorByName('email', $this->t('Please enter a valid email address.'));
      }
    }

    if ($step === 2) {
      $username = $form_state->getValue('username');
      if ($username && preg_match('/[^a-zA-Z0-9_]/', $username)) {
        $form_state->setErrorByName('username', $this->t('Username may only contain letters, numbers, and underscores.'));
      }
    }
  }

  /**
   * {@inheritdoc}
   */
  public function submitForm(array &$form, FormStateInterface $form_state): void {
    // This runs only on final submission (step 3).
    // All previous step values are in form_state storage.
    $data = [
      'first_name' => $form_state->get('first_name'),
      'last_name' => $form_state->get('last_name'),
      'email' => $form_state->get('email'),
      'phone' => $form_state->get('phone'),
      'username' => $form_state->get('username'),
      'plan' => $form_state->get('plan'),
      'newsletter' => (bool) $form_state->get('newsletter'),
    ];

    // Process registration — e.g., create user, send email, etc.
    \Drupal::logger('my_module')->notice('New registration: @data', [
      '@data' => print_r($data, TRUE),
    ]);

    $this->messenger()->addStatus($this->t('Registration complete! Welcome, @name.', [
      '@name' => $data['first_name'],
    ]));

    $form_state->setRedirect('<front>');
  }

}
```

### Route

```yaml
my_module.registration_wizard:
  path: '/register'
  defaults:
    _form: '\Drupal\my_module\Form\RegistrationWizardForm'
    _title: 'Register'
  requirements:
    _access: 'TRUE'
```

---

## Key Patterns and Tips

### Preserving values across steps

Always save values from the current step before advancing. Use
`$form_state->set()` in the "Next" submit handler, **before** calling
`$form_state->setRebuild()`. When rendering each step, use
`$form_state->get()` for `#default_value` so users see their previously
entered data when navigating back.

### Skipping validation on "Back"

Use `#limit_validation_errors` on the back button to skip validation:

```php
$form['actions']['back'] = [
  '#type' => 'submit',
  '#value' => $this->t('Back'),
  '#submit' => ['::goBack'],
  '#limit_validation_errors' => [],
];
```

### Adding AJAX to avoid full page reloads

Wrap the entire form in a container with an ID, then add `#ajax` to navigation
buttons referencing that wrapper. The `ajaxRebuild` callback simply returns
`$form` to replace the entire form contents.

### Step-specific validation

Check `$form_state->get('step')` in `validateForm()` to apply validation rules
only for the currently visible step. The triggering element name can also be
checked via `$form_state->getTriggeringElement()['#name']` if needed.

### Alternative: separate form classes per step

For very complex wizards, consider creating a separate form class for each
step and using `TempStore` (either `\Drupal\Core\TempStore\PrivateTempStoreFactory`
or `\Drupal\Core\TempStore\SharedTempStoreFactory`) to share data between them.
This provides better separation of concerns but requires more routing setup.
