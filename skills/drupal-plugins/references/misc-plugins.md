# Condition, Constraint, Views, and Custom Plugin Types

## Condition Plugins

Condition plugins evaluate boolean logic and are used in block visibility, context system, and other conditional logic throughout Drupal.

### Drupal 10 (Annotation)

```php
<?php

namespace Drupal\my_module\Plugin\Condition;

use Drupal\Core\Condition\ConditionPluginBase;
use Drupal\Core\Form\FormStateInterface;

/**
 * Provides a condition based on the current day of the week.
 *
 * @Condition(
 *   id = "my_module_day_of_week",
 *   label = @Translation("Day of the week"),
 * )
 */
class DayOfWeek extends ConditionPluginBase {

  /**
   * {@inheritdoc}
   */
  public function buildConfigurationForm(array $form, FormStateInterface $form_state): array {
    $form['days'] = [
      '#type' => 'checkboxes',
      '#title' => $this->t('Days of the week'),
      '#options' => [
        1 => $this->t('Monday'),
        2 => $this->t('Tuesday'),
        3 => $this->t('Wednesday'),
        4 => $this->t('Thursday'),
        5 => $this->t('Friday'),
        6 => $this->t('Saturday'),
        7 => $this->t('Sunday'),
      ],
      '#default_value' => $this->configuration['days'] ?? [],
    ];
    return parent::buildConfigurationForm($form, $form_state);
  }

  /**
   * {@inheritdoc}
   */
  public function submitConfigurationForm(array &$form, FormStateInterface $form_state): void {
    $this->configuration['days'] = array_filter($form_state->getValue('days'));
    parent::submitConfigurationForm($form, $form_state);
  }

  /**
   * {@inheritdoc}
   */
  public function evaluate(): bool {
    if (empty($this->configuration['days'])) {
      return TRUE;
    }
    $current_day = (int) date('N');
    return in_array($current_day, $this->configuration['days'], FALSE);
  }

  /**
   * {@inheritdoc}
   */
  public function summary(): string {
    $days = $this->configuration['days'] ?? [];
    if (empty($days)) {
      return $this->t('Any day');
    }
    return $this->t('Active on selected days');
  }

  /**
   * {@inheritdoc}
   */
  public function defaultConfiguration(): array {
    return ['days' => []] + parent::defaultConfiguration();
  }

}
```

### Drupal 11+ (Attribute)

```php
<?php

namespace Drupal\my_module\Plugin\Condition;

use Drupal\Core\Condition\Attribute\Condition;
use Drupal\Core\Condition\ConditionPluginBase;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\StringTranslation\TranslatableMarkup;

#[Condition(
  id: 'my_module_day_of_week',
  label: new TranslatableMarkup('Day of the week'),
)]
class DayOfWeek extends ConditionPluginBase {

  // Methods are identical to the annotation version.

}
```

---

## Constraint (Validation) Plugins

Constraints validate typed data and entity fields. Each constraint has two parts: the constraint definition class and the validator class.

### Constraint Definition

```php
<?php

namespace Drupal\my_module\Plugin\Validation\Constraint;

use Symfony\Component\Validator\Constraint;

/**
 * Validates that a value is a valid hex color code.
 *
 * @Constraint(
 *   id = "ValidHexColor",
 *   label = @Translation("Valid hex color", context = "Validation"),
 * )
 */
class ValidHexColorConstraint extends Constraint {

  /**
   * The error message for invalid hex colors.
   */
  public string $message = 'The value %value is not a valid hex color code.';

}
```

### Constraint Validator

```php
<?php

namespace Drupal\my_module\Plugin\Validation\Constraint;

use Symfony\Component\Validator\Constraint;
use Symfony\Component\Validator\ConstraintValidator;

class ValidHexColorConstraintValidator extends ConstraintValidator {

  /**
   * {@inheritdoc}
   */
  public function validate(mixed $value, Constraint $constraint): void {
    /** @var \Drupal\Core\Field\FieldItemInterface $value */
    if (!isset($value)) {
      return;
    }

    $color = $value->value ?? (string) $value;
    if ($color !== '' && !preg_match('/^#[0-9A-Fa-f]{6}$/', $color)) {
      $this->context->addViolation($constraint->message, ['%value' => $color]);
    }
  }

}
```

### Applying Constraints to Entity Fields

In a `hook_entity_bundle_field_info_alter()` or in your field type's `getConstraints()` method:

```php
/**
 * Implements hook_entity_bundle_field_info_alter().
 */
function my_module_entity_bundle_field_info_alter(array &$fields, \Drupal\Core\Entity\EntityTypeInterface $entity_type, string $bundle): void {
  if ($entity_type->id() === 'node' && $bundle === 'article') {
    if (isset($fields['field_color'])) {
      $fields['field_color']->addConstraint('ValidHexColor');
    }
  }
}
```

Or inside a FieldType class:

```php
public function getConstraints(): array {
  $constraints = parent::getConstraints();
  $constraint_manager = \Drupal::typedDataManager()->getValidationConstraintManager();
  $constraints[] = $constraint_manager->create('ValidHexColor', []);
  return $constraints;
}
```

---

## Views Plugins

Views uses its own plugin types for fields, filters, sorts, arguments, relationships, and display modes.

### Views Field Handler

```php
<?php

namespace Drupal\my_module\Plugin\views\field;

use Drupal\views\Plugin\views\field\FieldPluginBase;
use Drupal\views\ResultRow;

/**
 * Displays the word count for the body field.
 *
 * @ViewsField("my_module_word_count")
 */
class WordCount extends FieldPluginBase {

  /**
   * {@inheritdoc}
   */
  public function query(): void {
    // No query alteration needed; we compute from rendered data.
  }

  /**
   * {@inheritdoc}
   */
  public function render(ResultRow $values): string {
    /** @var \Drupal\node\NodeInterface $entity */
    $entity = $this->getEntity($values);
    if ($entity && $entity->hasField('body') && !$entity->get('body')->isEmpty()) {
      $text = strip_tags($entity->get('body')->value);
      return (string) str_word_count($text);
    }
    return '0';
  }

}
```

### Views Filter Handler

```php
<?php

namespace Drupal\my_module\Plugin\views\filter;

use Drupal\views\Plugin\views\filter\BooleanOperator;

/**
 * Filters nodes by whether they have a body field.
 *
 * @ViewsFilter("my_module_has_body")
 */
class HasBody extends BooleanOperator {

  /**
   * {@inheritdoc}
   */
  public function query(): void {
    $this->ensureMyTable();
    $field = "$this->tableAlias.body_value";

    if ($this->value) {
      $this->query->addWhereExpression($this->options['group'], "$field IS NOT NULL AND $field != ''");
    }
    else {
      $this->query->addWhereExpression($this->options['group'], "$field IS NULL OR $field = ''");
    }
  }

}
```

### Registering Views Plugins

In `my_module.views.inc`:

```php
/**
 * Implements hook_views_data_alter().
 */
function my_module_views_data_alter(array &$data): void {
  $data['node']['my_module_word_count'] = [
    'title' => t('Word count'),
    'help' => t('Displays the word count of the body field.'),
    'field' => [
      'id' => 'my_module_word_count',
    ],
  ];

  $data['node_field_data']['my_module_has_body'] = [
    'title' => t('Has body'),
    'help' => t('Filter nodes by whether they have body content.'),
    'filter' => [
      'id' => 'my_module_has_body',
    ],
  ];
}
```

---

## Creating a Custom Plugin Type

A complete example of creating your own plugin type with a manager, attribute, interface, and base class.

### 1. Attribute Class

```php
<?php

namespace Drupal\my_module\Attribute;

use Drupal\Component\Plugin\Attribute\Plugin;
use Drupal\Core\StringTranslation\TranslatableMarkup;

#[\Attribute(\Attribute::TARGET_CLASS)]
class DataExporter extends Plugin {

  public function __construct(
    public readonly string $id,
    public readonly TranslatableMarkup $label,
    public readonly ?TranslatableMarkup $description = NULL,
    public readonly string $file_extension = 'txt',
  ) {
    parent::__construct($id);
  }

}
```

### 2. Plugin Interface

```php
<?php

namespace Drupal\my_module\Plugin;

use Drupal\Component\Plugin\PluginInspectionInterface;

interface DataExporterInterface extends PluginInspectionInterface {

  /**
   * Exports data to a string in this exporter's format.
   *
   * @param array $data
   *   The data to export.
   *
   * @return string
   *   The exported string.
   */
  public function export(array $data): string;

  /**
   * Returns the file extension for this export format.
   */
  public function getFileExtension(): string;

}
```

### 3. Base Class (Optional)

```php
<?php

namespace Drupal\my_module\Plugin;

use Drupal\Core\Plugin\PluginBase;

abstract class DataExporterBase extends PluginBase implements DataExporterInterface {

  /**
   * {@inheritdoc}
   */
  public function getFileExtension(): string {
    return $this->pluginDefinition['file_extension'] ?? 'txt';
  }

}
```

### 4. Plugin Manager

```php
<?php

namespace Drupal\my_module\Plugin;

use Drupal\Core\Cache\CacheBackendInterface;
use Drupal\Core\Extension\ModuleHandlerInterface;
use Drupal\Core\Plugin\DefaultPluginManager;
use Drupal\my_module\Attribute\DataExporter;

class DataExporterPluginManager extends DefaultPluginManager {

  public function __construct(
    \Traversable $namespaces,
    CacheBackendInterface $cache_backend,
    ModuleHandlerInterface $module_handler,
  ) {
    parent::__construct(
      'Plugin/DataExporter',
      $namespaces,
      $module_handler,
      DataExporterInterface::class,
      DataExporter::class,
    );
    $this->setCacheBackend($cache_backend, 'data_exporter_plugins');
    $this->alterInfo('data_exporter_info');
  }

}
```

### 5. Service Registration

In `my_module.services.yml`:

```yaml
services:
  plugin.manager.data_exporter:
    class: Drupal\my_module\Plugin\DataExporterPluginManager
    parent: default_plugin_manager
```

### 6. A Concrete Plugin Implementation

```php
<?php

namespace Drupal\my_module\Plugin\DataExporter;

use Drupal\Core\StringTranslation\TranslatableMarkup;
use Drupal\my_module\Attribute\DataExporter;
use Drupal\my_module\Plugin\DataExporterBase;

#[DataExporter(
  id: 'csv',
  label: new TranslatableMarkup('CSV exporter'),
  description: new TranslatableMarkup('Exports data as comma-separated values.'),
  file_extension: 'csv',
)]
class CsvExporter extends DataExporterBase {

  /**
   * {@inheritdoc}
   */
  public function export(array $data): string {
    if (empty($data)) {
      return '';
    }

    $output = fopen('php://temp', 'r+');
    // Write header row from array keys.
    fputcsv($output, array_keys(reset($data)));
    foreach ($data as $row) {
      fputcsv($output, $row);
    }
    rewind($output);
    $csv = stream_get_contents($output);
    fclose($output);

    return $csv;
  }

}
```

### 7. Using the Custom Plugin Type

```php
/** @var \Drupal\my_module\Plugin\DataExporterPluginManager $manager */
$manager = \Drupal::service('plugin.manager.data_exporter');

// List all available exporters.
$definitions = $manager->getDefinitions();

// Create a specific exporter instance.
$exporter = $manager->createInstance('csv');
$csv_output = $exporter->export($data);
```

---

## Key Points

- Condition plugins support a **negate** option out of the box via the parent class -- this lets users invert the condition.
- Constraint validators must be named `{ConstraintClassName}Validator` in the same namespace.
- Views plugins must be declared in `hook_views_data()` or `hook_views_data_alter()` to be available.
- Custom plugin types need a `parent: default_plugin_manager` in services.yml to inherit the standard constructor arguments (`container.namespaces`, `cache.discovery`, `module_handler`).
- The `alterInfo()` call enables `hook_{alter_name}_alter()` for other modules to modify plugin definitions.
