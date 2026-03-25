# Functional Testing in Drupal — Deep Dive

## BrowserTestBase

Functional tests extend `Drupal\Tests\BrowserTestBase`. They install Drupal into a real database, make internal HTTP requests via Mink, and assert against the rendered page.

```php
<?php

declare(strict_types=1);

namespace Drupal\Tests\my_module\Functional;

use Drupal\Tests\BrowserTestBase;

/**
 * Tests the public-facing pages of my_module.
 *
 * @group my_module
 */
class PublicPagesTest extends BrowserTestBase {

  /**
   * {@inheritdoc}
   */
  protected static $modules = ['my_module', 'node', 'block'];

  /**
   * {@inheritdoc}
   */
  protected $defaultTheme = 'stark';

}
```

**Important:** Always set `$defaultTheme` to `'stark'`, `'olivero'`, or another theme. Omitting it triggers a deprecation warning (and will become an error in future Drupal versions).

## User Management

### drupalCreateUser()

Creates a user with the given permissions.

```php
public function testAdminPageAccess(): void {
  // Create a user with specific permissions.
  $admin = $this->drupalCreateUser([
    'access administration pages',
    'administer my_module',
  ]);

  $this->drupalLogin($admin);
  $this->drupalGet('/admin/config/my-module/settings');
  $this->assertSession()->statusCodeEquals(200);
}

public function testAnonymousAccessDenied(): void {
  $this->drupalGet('/admin/config/my-module/settings');
  $this->assertSession()->statusCodeEquals(403);
}
```

### drupalLogin() and drupalLogout()

```php
$user = $this->drupalCreateUser(['access content']);
$this->drupalLogin($user);

// Perform actions as logged-in user...

$this->drupalLogout();
```

### drupalCreateRole()

```php
$role_id = $this->drupalCreateRole([
  'access content',
  'view my_module reports',
], 'report_viewer', 'Report Viewer');
```

## Page Navigation

### drupalGet()

Navigates to a URL and stores the response.

```php
$this->drupalGet('/node/1');
$this->drupalGet('/admin/config/my-module/settings');
$this->drupalGet('/node/1', ['query' => ['page' => '2']]);
```

### clickLink()

```php
$this->drupalGet('/my-module/reports');
$this->clickLink('View quarterly report');
$this->assertSession()->addressEquals('/my-module/reports/quarterly');
```

## Form Submission

### submitForm()

The primary way to submit forms in functional tests. Pass an array of form field values and the submit button label.

```php
public function testSettingsFormSave(): void {
  $admin = $this->drupalCreateUser(['administer my_module']);
  $this->drupalLogin($admin);
  $this->drupalGet('/admin/config/my-module/settings');

  $this->submitForm([
    'api_key' => 'new-key-456',
    'report_limit' => '25',
    'enabled' => TRUE,
  ], 'Save configuration');

  $this->assertSession()->pageTextContains('The configuration options have been saved.');

  // Verify the configuration was actually saved.
  $config = $this->config('my_module.settings');
  $this->assertEquals('new-key-456', $config->get('api_key'));
  $this->assertEquals(25, $config->get('report_limit'));
}
```

### Form Field Types

```php
$this->submitForm([
  // Text field.
  'title[0][value]' => 'My Article',

  // Select list.
  'field_category' => 'news',

  // Checkbox.
  'status[value]' => TRUE,

  // Radio buttons.
  'field_layout' => 'grid',

  // Textarea.
  'body[0][value]' => 'Body content here.',

  // File upload (provide absolute path).
  'files[field_document_0]' => '/path/to/test-file.pdf',

  // Autocomplete entity reference.
  'field_author[0][target_id]' => 'Admin (1)',
], 'Save');
```

## Assertions with assertSession()

### Page Content

```php
$this->assertSession()->pageTextContains('Welcome to the site');
$this->assertSession()->pageTextNotContains('Access denied');
$this->assertSession()->responseContains('<meta name="generator" content="Drupal');
$this->assertSession()->responseNotContains('Fatal error');
```

### Status Codes and Redirects

```php
$this->assertSession()->statusCodeEquals(200);
$this->assertSession()->statusCodeEquals(403);
$this->assertSession()->statusCodeEquals(404);
$this->assertSession()->addressEquals('/expected/path');
```

### Elements and CSS

```php
// Element exists by CSS selector.
$this->assertSession()->elementExists('css', '.report-table');
$this->assertSession()->elementNotExists('css', '.error-message');

// Element contains text.
$this->assertSession()->elementTextContains('css', 'h1.page-title', 'Reports');

// Element count.
$this->assertSession()->elementsCount('css', 'table.report-table tbody tr', 5);

// Element attributes.
$element = $this->assertSession()->elementExists('css', 'a.download-link');
$this->assertEquals('https://example.com/report.pdf', $element->getAttribute('href'));
```

### Form Fields

```php
$this->assertSession()->fieldExists('api_key');
$this->assertSession()->fieldNotExists('secret_field');
$this->assertSession()->fieldValueEquals('api_key', 'current-key-123');
$this->assertSession()->fieldDisabled('locked_field');
$this->assertSession()->checkboxChecked('enabled');
$this->assertSession()->checkboxNotChecked('debug_mode');
```

### Links

```php
$this->assertSession()->linkExists('View all reports');
$this->assertSession()->linkNotExists('Delete everything');
$this->assertSession()->linkByHrefExists('/my-module/reports');
```

## Creating Test Content

### Nodes

```php
use Drupal\Tests\node\Traits\NodeCreationTrait;

class MyContentTest extends BrowserTestBase {

  use NodeCreationTrait;

  protected static $modules = ['my_module', 'node'];
  protected $defaultTheme = 'stark';

  public function testNodeDisplay(): void {
    $this->drupalCreateContentType(['type' => 'article', 'name' => 'Article']);

    $node = $this->createNode([
      'type' => 'article',
      'title' => 'Test Article',
      'body' => [['value' => 'Article body text.', 'format' => 'plain_text']],
      'status' => 1,
    ]);

    $this->drupalGet('/node/' . $node->id());
    $this->assertSession()->statusCodeEquals(200);
    $this->assertSession()->pageTextContains('Test Article');
    $this->assertSession()->pageTextContains('Article body text.');
  }

}
```

### Blocks

```php
$this->drupalPlaceBlock('system_powered_by_block', [
  'region' => 'content',
  'label' => 'Powered by Drupal',
]);
```

## WebDriverTestBase — JavaScript Testing

For testing JavaScript interactions, AJAX-driven forms, and dynamic UI behavior.

```php
<?php

declare(strict_types=1);

namespace Drupal\Tests\my_module\FunctionalJavascript;

use Drupal\FunctionalJavascriptTests\WebDriverTestBase;

/**
 * Tests the dynamic report filter form.
 *
 * @group my_module
 */
class ReportFilterTest extends WebDriverTestBase {

  protected static $modules = ['my_module', 'node'];
  protected $defaultTheme = 'stark';

  /**
   * Tests AJAX-driven category filter.
   */
  public function testCategoryFilter(): void {
    $admin = $this->drupalCreateUser(['view my_module reports']);
    $this->drupalLogin($admin);
    $this->drupalGet('/my-module/reports');

    $page = $this->getSession()->getPage();

    // Select a category, which triggers AJAX.
    $page->selectFieldOption('category', 'quarterly');
    $this->assertSession()->assertWaitOnAjaxRequest();

    // Verify the results updated.
    $this->assertSession()->pageTextContains('Q1 Report');
    $this->assertSession()->pageTextNotContains('Monthly Summary');
  }

}
```

### Waiting for Elements

Never use `sleep()`. Use explicit waits instead.

```php
// Wait for AJAX requests to complete.
$this->assertSession()->assertWaitOnAjaxRequest();

// Wait for a specific element to appear (timeout in ms).
$this->assertSession()->waitForElement('css', '.results-loaded', 5000);

// Wait for text to appear.
$this->assertSession()->waitForText('Results loaded');

// Wait for an element to be removed.
$this->assertSession()->waitForElementRemoved('css', '.loading-spinner');

// Custom wait with callback.
$result = $this->getSession()->getPage()->waitFor(10, function ($page) {
  $element = $page->find('css', '.dynamic-content');
  return $element && $element->isVisible();
});
$this->assertTrue($result, 'Dynamic content became visible.');
```

### Interacting with JavaScript Elements

```php
$page = $this->getSession()->getPage();

// Click a button.
$page->pressButton('Apply Filters');

// Fill in a field.
$page->fillField('search', 'quarterly');

// Check/uncheck a checkbox.
$page->checkField('include_archived');
$page->uncheckField('include_archived');

// Work with elements directly.
$element = $page->find('css', '.dropdown-toggle');
$element->click();

// Interact with a dialog/modal.
$this->assertSession()->waitForElement('css', '.ui-dialog');
$dialog = $page->find('css', '.ui-dialog');
$dialog->pressButton('Confirm');
```

### Taking Screenshots

Useful for debugging test failures.

```php
// Capture a screenshot at a specific point in the test.
$this->createScreenshot('/tmp/test-screenshot.png');

// In DDEV context the output directory is typically:
// /var/www/html/web/sites/simpletest/browser_output/
```

## Testing Access Control

```php
/**
 * Tests permission-based access control.
 */
public function testAccessControl(): void {
  // User without permission.
  $viewer = $this->drupalCreateUser(['access content']);
  $this->drupalLogin($viewer);
  $this->drupalGet('/admin/config/my-module/settings');
  $this->assertSession()->statusCodeEquals(403);

  // User with permission.
  $admin = $this->drupalCreateUser(['administer my_module']);
  $this->drupalLogin($admin);
  $this->drupalGet('/admin/config/my-module/settings');
  $this->assertSession()->statusCodeEquals(200);
}
```

## Testing Views Pages (if your module provides Views)

```php
public function testReportsViewPage(): void {
  $this->drupalCreateContentType(['type' => 'report']);

  // Create several test nodes.
  for ($i = 1; $i <= 5; $i++) {
    $this->createNode([
      'type' => 'report',
      'title' => "Report $i",
      'status' => 1,
    ]);
  }

  $user = $this->drupalCreateUser(['access content']);
  $this->drupalLogin($user);
  $this->drupalGet('/reports');

  $this->assertSession()->statusCodeEquals(200);
  $this->assertSession()->elementsCount('css', '.views-row', 5);
  $this->assertSession()->pageTextContains('Report 1');
}
```

## Common Pitfalls

1. **Missing `$defaultTheme`** — Always set it. Without it you get deprecation warnings or errors.
2. **Form field names** — Use the HTML `name` attribute, not the Drupal Form API key. Inspect the rendered form to find the correct name (e.g., `title[0][value]` not `title`).
3. **Permissions strings** — Must exactly match the permission machine name defined in `my_module.permissions.yml` or `hook_permission()`.
4. **Content type setup** — If your test needs nodes, ensure the `node` module is in `$modules` and call `drupalCreateContentType()` before creating nodes.
5. **AJAX in BrowserTestBase** — `BrowserTestBase` does not execute JavaScript. You must use `WebDriverTestBase` for any AJAX or JS-dependent behavior.
6. **Test isolation** — Each test method gets a fresh Drupal installation. Do not rely on state from another test method.
7. **Slow test execution** — Functional tests are inherently slower. Combine related assertions in a single test method when they share the same setup to reduce total execution time.
