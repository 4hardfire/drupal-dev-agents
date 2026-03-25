# Field Plugins — Deep Dive

Field plugins in Drupal consist of three tightly coupled plugin types that work together:

1. **FieldType** -- defines how data is stored in the database.
2. **FieldWidget** -- defines the form element used to input data.
3. **FieldFormatter** -- defines how stored data is rendered for display.

---

## FieldType Plugin

### Drupal 10 (Annotation)

```php
<?php

namespace Drupal\my_module\Plugin\Field\FieldType;

use Drupal\Core\Field\FieldItemBase;
use Drupal\Core\Field\FieldStorageDefinitionInterface;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\TypedData\DataDefinition;

/**
 * Plugin implementation of the 'address_simple' field type.
 *
 * @FieldType(
 *   id = "address_simple",
 *   label = @Translation("Simple address"),
 *   description = @Translation("Stores a simple address with street, city, and postal code."),
 *   default_widget = "address_simple_default",
 *   default_formatter = "address_simple_default",
 *   category = @Translation("General"),
 * )
 */
class AddressSimpleItem extends FieldItemBase {

  /**
   * {@inheritdoc}
   */
  public static function schema(FieldStorageDefinitionInterface $field_definition): array {
    return [
      'columns' => [
        'street' => [
          'type' => 'varchar',
          'length' => 255,
          'not null' => FALSE,
        ],
        'city' => [
          'type' => 'varchar',
          'length' => 128,
          'not null' => FALSE,
        ],
        'postal_code' => [
          'type' => 'varchar',
          'length' => 20,
          'not null' => FALSE,
        ],
        'country' => [
          'type' => 'varchar',
          'length' => 2,
          'not null' => FALSE,
        ],
      ],
      'indexes' => [
        'postal_code' => ['postal_code'],
      ],
    ];
  }

  /**
   * {@inheritdoc}
   */
  public static function propertyDefinitions(FieldStorageDefinitionInterface $field_definition): array {
    $properties = [];

    $properties['street'] = DataDefinition::create('string')
      ->setLabel(t('Street address'));

    $properties['city'] = DataDefinition::create('string')
      ->setLabel(t('City'));

    $properties['postal_code'] = DataDefinition::create('string')
      ->setLabel(t('Postal code'));

    $properties['country'] = DataDefinition::create('string')
      ->setLabel(t('Country code'));

    return $properties;
  }

  /**
   * {@inheritdoc}
   */
  public function isEmpty(): bool {
    $street = $this->get('street')->getValue();
    $city = $this->get('city')->getValue();
    return ($street === NULL || $street === '') && ($city === NULL || $city === '');
  }

  /**
   * {@inheritdoc}
   */
  public static function defaultStorageSettings(): array {
    return [
      'max_street_length' => 255,
    ] + parent::defaultStorageSettings();
  }

  /**
   * {@inheritdoc}
   */
  public function storageSettingsForm(array &$form, FormStateInterface $form_state, $has_data): array {
    $element = [];
    $element['max_street_length'] = [
      '#type' => 'number',
      '#title' => t('Maximum street length'),
      '#default_value' => $this->getSetting('max_street_length'),
      '#min' => 1,
      '#max' => 255,
      '#disabled' => $has_data,
    ];
    return $element;
  }

  /**
   * {@inheritdoc}
   */
  public static function defaultFieldSettings(): array {
    return [
      'default_country' => 'US',
    ] + parent::defaultFieldSettings();
  }

  /**
   * {@inheritdoc}
   */
  public function fieldSettingsForm(array $form, FormStateInterface $form_state): array {
    $element = [];
    $element['default_country'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Default country code'),
      '#default_value' => $this->getSetting('default_country'),
      '#maxlength' => 2,
      '#size' => 4,
    ];
    return $element;
  }

}
```

### Drupal 11+ (Attribute)

```php
<?php

namespace Drupal\my_module\Plugin\Field\FieldType;

use Drupal\Core\Field\Attribute\FieldType;
use Drupal\Core\Field\FieldItemBase;
use Drupal\Core\Field\FieldStorageDefinitionInterface;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use Drupal\Core\TypedData\DataDefinition;

#[FieldType(
  id: 'address_simple',
  label: new TranslatableMarkup('Simple address'),
  description: new TranslatableMarkup('Stores a simple address with street, city, and postal code.'),
  default_widget: 'address_simple_default',
  default_formatter: 'address_simple_default',
  category: new TranslatableMarkup('General'),
)]
class AddressSimpleItem extends FieldItemBase {

  // All methods are identical to the annotation version above.

}
```

---

## FieldWidget Plugin

Widgets define how the field appears on entity edit forms.

### Drupal 10 (Annotation)

```php
<?php

namespace Drupal\my_module\Plugin\Field\FieldWidget;

use Drupal\Core\Field\FieldItemListInterface;
use Drupal\Core\Field\WidgetBase;
use Drupal\Core\Form\FormStateInterface;

/**
 * Plugin implementation of the 'address_simple_default' widget.
 *
 * @FieldWidget(
 *   id = "address_simple_default",
 *   label = @Translation("Simple address (default)"),
 *   field_types = {
 *     "address_simple"
 *   }
 * )
 */
class AddressSimpleDefaultWidget extends WidgetBase {

  /**
   * {@inheritdoc}
   */
  public static function defaultSettings(): array {
    return [
      'placeholder_street' => '',
      'placeholder_city' => '',
    ] + parent::defaultSettings();
  }

  /**
   * {@inheritdoc}
   */
  public function settingsForm(array $form, FormStateInterface $form_state): array {
    $element = [];
    $element['placeholder_street'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Street placeholder'),
      '#default_value' => $this->getSetting('placeholder_street'),
    ];
    $element['placeholder_city'] = [
      '#type' => 'textfield',
      '#title' => $this->t('City placeholder'),
      '#default_value' => $this->getSetting('placeholder_city'),
    ];
    return $element;
  }

  /**
   * {@inheritdoc}
   */
  public function settingsSummary(): array {
    $summary = [];
    if ($placeholder = $this->getSetting('placeholder_street')) {
      $summary[] = $this->t('Street placeholder: @placeholder', ['@placeholder' => $placeholder]);
    }
    return $summary;
  }

  /**
   * {@inheritdoc}
   */
  public function formElement(FieldItemListInterface $items, $delta, array $element, array &$form, FormStateInterface $form_state): array {
    $element['street'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Street'),
      '#default_value' => $items[$delta]->street ?? '',
      '#placeholder' => $this->getSetting('placeholder_street'),
      '#maxlength' => 255,
    ];
    $element['city'] = [
      '#type' => 'textfield',
      '#title' => $this->t('City'),
      '#default_value' => $items[$delta]->city ?? '',
      '#placeholder' => $this->getSetting('placeholder_city'),
      '#maxlength' => 128,
    ];
    $element['postal_code'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Postal code'),
      '#default_value' => $items[$delta]->postal_code ?? '',
      '#size' => 10,
      '#maxlength' => 20,
    ];
    $element['country'] = [
      '#type' => 'textfield',
      '#title' => $this->t('Country code'),
      '#default_value' => $items[$delta]->country ?? $items->getFieldDefinition()->getSetting('default_country'),
      '#size' => 4,
      '#maxlength' => 2,
    ];
    return $element;
  }

}
```

### Drupal 11+ (Attribute)

```php
<?php

namespace Drupal\my_module\Plugin\Field\FieldWidget;

use Drupal\Core\Field\Attribute\FieldWidget;
use Drupal\Core\Field\FieldItemListInterface;
use Drupal\Core\Field\WidgetBase;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\StringTranslation\TranslatableMarkup;

#[FieldWidget(
  id: 'address_simple_default',
  label: new TranslatableMarkup('Simple address (default)'),
  field_types: ['address_simple'],
)]
class AddressSimpleDefaultWidget extends WidgetBase {

  // All methods are identical to the annotation version above.

}
```

---

## FieldFormatter Plugin

Formatters control how field data is rendered on the frontend.

### Drupal 10 (Annotation)

```php
<?php

namespace Drupal\my_module\Plugin\Field\FieldFormatter;

use Drupal\Core\Field\FieldItemListInterface;
use Drupal\Core\Field\FormatterBase;
use Drupal\Core\Form\FormStateInterface;

/**
 * Plugin implementation of the 'address_simple_default' formatter.
 *
 * @FieldFormatter(
 *   id = "address_simple_default",
 *   label = @Translation("Simple address (default)"),
 *   field_types = {
 *     "address_simple"
 *   }
 * )
 */
class AddressSimpleDefaultFormatter extends FormatterBase {

  /**
   * {@inheritdoc}
   */
  public static function defaultSettings(): array {
    return [
      'show_country' => TRUE,
    ] + parent::defaultSettings();
  }

  /**
   * {@inheritdoc}
   */
  public function settingsForm(array $form, FormStateInterface $form_state): array {
    $form['show_country'] = [
      '#type' => 'checkbox',
      '#title' => $this->t('Show country code'),
      '#default_value' => $this->getSetting('show_country'),
    ];
    return $form;
  }

  /**
   * {@inheritdoc}
   */
  public function settingsSummary(): array {
    $summary = [];
    $summary[] = $this->getSetting('show_country')
      ? $this->t('Country code: shown')
      : $this->t('Country code: hidden');
    return $summary;
  }

  /**
   * {@inheritdoc}
   */
  public function viewElements(FieldItemListInterface $items, $langcode): array {
    $elements = [];

    foreach ($items as $delta => $item) {
      $parts = array_filter([
        $item->street,
        $item->city,
        $item->postal_code,
        $this->getSetting('show_country') ? $item->country : NULL,
      ]);

      $elements[$delta] = [
        '#theme' => 'my_module_address_simple',
        '#street' => $item->street,
        '#city' => $item->city,
        '#postal_code' => $item->postal_code,
        '#country' => $this->getSetting('show_country') ? $item->country : NULL,
        '#address_line' => implode(', ', $parts),
      ];
    }

    return $elements;
  }

}
```

### Drupal 11+ (Attribute)

```php
<?php

namespace Drupal\my_module\Plugin\Field\FieldFormatter;

use Drupal\Core\Field\Attribute\FieldFormatter;
use Drupal\Core\Field\FieldItemListInterface;
use Drupal\Core\Field\FormatterBase;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\StringTranslation\TranslatableMarkup;

#[FieldFormatter(
  id: 'address_simple_default',
  label: new TranslatableMarkup('Simple address (default)'),
  field_types: ['address_simple'],
)]
class AddressSimpleDefaultFormatter extends FormatterBase {

  // All methods are identical to the annotation version above.

}
```

---

## Registering Theme Templates for Field Formatters

If your formatter uses `#theme`, register the template in your module's `.module` file:

```php
/**
 * Implements hook_theme().
 */
function my_module_theme(): array {
  return [
    'my_module_address_simple' => [
      'variables' => [
        'street' => NULL,
        'city' => NULL,
        'postal_code' => NULL,
        'country' => NULL,
        'address_line' => NULL,
      ],
    ],
  ];
}
```

Then create the template at `templates/my-module-address-simple.html.twig`:

```twig
<div class="address-simple">
  {% if street %}<div class="address-simple__street">{{ street }}</div>{% endif %}
  {% if city or postal_code %}
    <div class="address-simple__locality">
      {{ city }}{% if city and postal_code %}, {% endif %}{{ postal_code }}
    </div>
  {% endif %}
  {% if country %}<div class="address-simple__country">{{ country }}</div>{% endif %}
</div>
```

---

## Key Points

- **schema()** defines the database columns. Changing schema on an existing field requires a database update.
- **propertyDefinitions()** defines typed data properties that map to schema columns.
- **isEmpty()** determines whether a field value counts as empty (affects required validation).
- **field_types** in widgets and formatters is an array -- a widget/formatter can support multiple field types.
- Widget and formatter **settings** are stored per form/view display mode and configured in "Manage form display" / "Manage display" admin pages.
- Always run `ddev drush cr` after creating or modifying field plugin classes.
