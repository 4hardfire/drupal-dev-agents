---
name: drupal-testing
description: Use when writing Drupal tests, unit tests, kernel tests, functional tests, or browser tests using PHPUnit and Drupal test base classes.
version: 1.0.0
---

# Drupal Testing

## Overview

Drupal uses PHPUnit as its testing framework and provides four distinct test types, each with increasing scope and execution cost.

| Test Type | Base Class | Boots Drupal? | Database? | Browser? | Speed |
|---|---|---|---|---|---|
| Unit | `UnitTestCase` | No | No | No | Fastest |
| Kernel | `KernelTestBase` | Partial | Yes (SQLite) | No | Fast |
| Functional | `BrowserTestBase` | Full | Yes | Simulated | Slow |
| FunctionalJavascript | `WebDriverTestBase` | Full | Yes | Real (Chrome) | Slowest |

**When to use each:**

- **Unit tests** — Pure logic: utility functions, value objects, data transformations, services that can be fully mocked. No Drupal API calls.
- **Kernel tests** — Code that interacts with Drupal services, entities, database, configuration, or the plugin system, but does not need a full HTTP request cycle.
- **Functional tests (BrowserTestBase)** — Testing page output, forms, access control, routes, and user-facing behavior that does not require JavaScript.
- **FunctionalJavascript tests (WebDriverTestBase)** — Testing AJAX interactions, JavaScript-dependent UI components, and dynamic page behavior.

## Test Directory Structure

Tests live within a module's `tests/` directory, following PSR-4 namespacing:

```
modules/custom/my_module/
  tests/
    src/
      Unit/
        MyServiceTest.php
      Kernel/
        MyEntityTest.php
      Functional/
        MyPageTest.php
      FunctionalJavascript/
        MyDynamicFormTest.php
    modules/
      my_module_test/       # Optional: test-only sub-module
        my_module_test.info.yml
        my_module_test.module
```

Namespace pattern:

```
Drupal\Tests\my_module\Unit\MyServiceTest
Drupal\Tests\my_module\Kernel\MyEntityTest
Drupal\Tests\my_module\Functional\MyPageTest
Drupal\Tests\my_module\FunctionalJavascript\MyDynamicFormTest
```

## Unit Tests

Unit tests extend `Drupal\Tests\UnitTestCase` and run without bootstrapping Drupal.

```php
<?php

declare(strict_types=1);

namespace Drupal\Tests\my_module\Unit;

use Drupal\my_module\Utility\StringHelper;
use Drupal\Tests\UnitTestCase;

/**
 * @coversDefaultClass \Drupal\my_module\Utility\StringHelper
 * @group my_module
 */
class StringHelperTest extends UnitTestCase {

  /**
   * @covers ::sanitizeLabel
   * @dataProvider sanitizeLabelProvider
   */
  public function testSanitizeLabel(string $input, string $expected): void {
    $helper = new StringHelper();
    $this->assertEquals($expected, $helper->sanitizeLabel($input));
  }

  /**
   * Data provider for testSanitizeLabel().
   */
  public static function sanitizeLabelProvider(): array {
    return [
      'simple string' => ['Hello World', 'hello_world'],
      'special chars' => ['Hello@World!', 'helloworld'],
      'empty string' => ['', ''],
    ];
  }

}
```

### Mocking with createMock()

```php
$entity_storage = $this->createMock(EntityStorageInterface::class);
$entity_storage->method('load')
  ->with('node_1')
  ->willReturn($this->createMock(NodeInterface::class));

$entity_type_manager = $this->createMock(EntityTypeManagerInterface::class);
$entity_type_manager->method('getStorage')
  ->with('node')
  ->willReturn($entity_storage);
```

### Prophecy (alternative mocking)

```php
use Prophecy\PhpUnit\ProphecyTrait;

class MyServiceTest extends UnitTestCase {

  use ProphecyTrait;

  public function testProcess(): void {
    $logger = $this->prophesize(LoggerInterface::class);
    $logger->info('Processing item')->shouldBeCalledOnce();

    $service = new MyService($logger->reveal());
    $service->process();
  }

}
```

## Kernel Tests

Kernel tests extend `Drupal\KernelTests\KernelTestBase` and boot a minimal Drupal container.

```php
<?php

declare(strict_types=1);

namespace Drupal\Tests\my_module\Kernel;

use Drupal\KernelTests\KernelTestBase;
use Drupal\my_module\Service\DataProcessor;

/**
 * Tests the DataProcessor service.
 *
 * @group my_module
 */
class DataProcessorTest extends KernelTestBase {

  /**
   * {@inheritdoc}
   */
  protected static $modules = [
    'system',
    'user',
    'node',
    'field',
    'text',
    'my_module',
  ];

  /**
   * {@inheritdoc}
   */
  protected function setUp(): void {
    parent::setUp();
    $this->installEntitySchema('user');
    $this->installEntitySchema('node');
    $this->installConfig(['system', 'my_module']);
    $this->installSchema('node', ['node_access']);
  }

  /**
   * Tests that the data processor service resolves from the container.
   */
  public function testServiceResolution(): void {
    $service = $this->container->get('my_module.data_processor');
    $this->assertInstanceOf(DataProcessor::class, $service);
  }

}
```

Key setUp() methods:
- `installEntitySchema('entity_type_id')` — Creates entity database tables.
- `installConfig(['module_name'])` — Imports default configuration.
- `installSchema('module', ['table_name'])` — Installs specific non-entity tables.

## Functional Tests (BrowserTestBase)

Functional tests extend `Drupal\Tests\BrowserTestBase` and perform full HTTP requests through an internal browser.

```php
<?php

declare(strict_types=1);

namespace Drupal\Tests\my_module\Functional;

use Drupal\Tests\BrowserTestBase;

/**
 * Tests the settings form for my_module.
 *
 * @group my_module
 */
class SettingsFormTest extends BrowserTestBase {

  /**
   * {@inheritdoc}
   */
  protected static $modules = ['my_module'];

  /**
   * {@inheritdoc}
   */
  protected $defaultTheme = 'stark';

  /**
   * Tests the settings form submission.
   */
  public function testSettingsForm(): void {
    $admin = $this->drupalCreateUser(['administer my_module']);
    $this->drupalLogin($admin);

    $this->drupalGet('/admin/config/my-module/settings');
    $this->assertSession()->statusCodeEquals(200);
    $this->assertSession()->pageTextContains('My Module Settings');

    $this->submitForm([
      'api_key' => 'test-key-123',
      'enabled' => TRUE,
    ], 'Save configuration');

    $this->assertSession()->pageTextContains('The configuration options have been saved.');
  }

}
```

## Running Tests

### With DDEV

```bash
# Run all tests for a module.
ddev exec phpunit --group my_module

# Run a specific test class.
ddev exec phpunit web/modules/custom/my_module/tests/src/Unit/MyServiceTest.php

# Run a specific test method.
ddev exec phpunit --filter testSanitizeLabel web/modules/custom/my_module/tests/src/Unit/MyServiceTest.php

# Run only unit tests for a module.
ddev exec phpunit --testsuite unit --group my_module

# Run with verbose output.
ddev exec phpunit -v --group my_module
```

### phpunit.xml Configuration in DDEV

A typical `phpunit.xml` placed in the project root (or `web/core/`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/9.3/phpunit.xsd"
  bootstrap="web/core/tests/bootstrap.php"
  colors="true"
  beStrictAboutTestsThatDoNotTestAnything="true"
  beStrictAboutOutputDuringTests="true"
  beStrictAboutChangesToGlobalState="true">
  <php>
    <ini name="error_reporting" value="32767"/>
    <ini name="memory_limit" value="-1"/>
    <env name="SIMPLETEST_BASE_URL" value="http://web"/>
    <env name="SIMPLETEST_DB" value="mysql://db:db@db:3306/db"/>
    <env name="BROWSERTEST_OUTPUT_DIRECTORY" value="/var/www/html/web/sites/simpletest/browser_output"/>
    <env name="MINK_DRIVER_ARGS_WEBDRIVER" value='["chrome", {"browserName":"chrome","goog:chromeOptions":{"args":["--disable-gpu","--headless","--no-sandbox"]}}, "http://chrome:9222"]'/>
  </php>
  <testsuites>
    <testsuite name="unit">
      <directory>web/modules/custom/*/tests/src/Unit</directory>
    </testsuite>
    <testsuite name="kernel">
      <directory>web/modules/custom/*/tests/src/Kernel</directory>
    </testsuite>
    <testsuite name="functional">
      <directory>web/modules/custom/*/tests/src/Functional</directory>
    </testsuite>
    <testsuite name="functional-javascript">
      <directory>web/modules/custom/*/tests/src/FunctionalJavascript</directory>
    </testsuite>
  </testsuites>
</phpunit>
```

## Common Assertions

### PHPUnit Standard

```php
$this->assertEquals('expected', $actual);
$this->assertSame('expected', $actual);       // Strict type comparison.
$this->assertTrue($value);
$this->assertFalse($value);
$this->assertNull($value);
$this->assertCount(3, $array);
$this->assertInstanceOf(NodeInterface::class, $entity);
$this->assertArrayHasKey('key', $array);
$this->assertStringContainsString('needle', $haystack);
$this->assertEmpty($collection);
```

### Browser Test Assertions (assertSession)

```php
$this->assertSession()->statusCodeEquals(200);
$this->assertSession()->pageTextContains('Expected text');
$this->assertSession()->pageTextNotContains('Unwanted text');
$this->assertSession()->fieldValueEquals('field_name', 'expected_value');
$this->assertSession()->linkExists('Click me');
$this->assertSession()->elementExists('css', '.my-class');
$this->assertSession()->addressEquals('/expected/path');
$this->assertSession()->responseContains('<div class="custom">');
```

## Scaffolding with Drush

Use `drush generate` (via DDEV) to scaffold test files:

```bash
# Generate a test interactively.
ddev drush gen test

# Specify the test type (phpunit:unit, phpunit:kernel, phpunit:functional, phpunit:javascript).
ddev drush gen phpunit:unit
ddev drush gen phpunit:kernel
ddev drush gen phpunit:functional
```

This creates a properly namespaced test file with the correct base class and boilerplate.

## Testing Best Practices

1. **One assertion concept per test method.** A test can contain multiple `assert*()` calls if they verify the same logical outcome.
2. **Use data providers** for testing the same logic with different inputs.
3. **Name tests descriptively**: `testAnonymousUserCannotAccessAdminPage()` over `testAccess()`.
4. **Use the lightest test type possible.** Prefer Unit over Kernel, Kernel over Functional.
5. **Declare `$modules` statically** — it must be `protected static $modules`, not a dynamic property.
6. **Set `$defaultTheme`** in all BrowserTestBase and WebDriverTestBase tests to avoid deprecation warnings.
7. **Clean up** — Kernel and Functional tests handle teardown automatically; avoid manual database cleanup.
8. **Avoid sleep() in tests** — use `$this->assertSession()->waitForElement()` or similar wait methods in JS tests.

## Related Skills

- **drupal-module-development** — Module structure, services, plugins, hooks, and routing that tests validate.
- **drupal-entity-development** — Entity types, fields, and storage that kernel tests commonly exercise.
- **drupal-forms** — Form building and validation logic tested via functional and unit tests.
- **ddev** — Local environment commands for running tests (`ddev exec phpunit`).
