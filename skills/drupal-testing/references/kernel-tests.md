# Kernel Testing in Drupal — Deep Dive

## Overview

Kernel tests boot a minimal Drupal container with an in-memory SQLite database. They give you access to real services, entity storage, configuration, and the database layer without the overhead of a full HTTP stack. Kernel tests are the best choice when your code depends on Drupal APIs but does not require browser interaction.

## Base Class and Structure

```php
<?php

declare(strict_types=1);

namespace Drupal\Tests\my_module\Kernel;

use Drupal\KernelTests\KernelTestBase;

/**
 * Tests for the my_module data layer.
 *
 * @group my_module
 */
class DataLayerTest extends KernelTestBase {

  /**
   * Modules to enable.
   *
   * @var array<string>
   */
  protected static $modules = [
    'system',
    'user',
    'my_module',
  ];

  /**
   * {@inheritdoc}
   */
  protected function setUp(): void {
    parent::setUp();
    // Install schemas and configuration needed by the test.
  }

}
```

**Key rule:** `$modules` must be `protected static $modules` — an array of module machine names as strings.

## setUp() — Installing Schemas, Config, and Entity Schemas

### installEntitySchema()

Creates the database tables for an entity type. Call this for every entity type your test touches.

```php
protected function setUp(): void {
  parent::setUp();
  $this->installEntitySchema('user');
  $this->installEntitySchema('node');
  $this->installEntitySchema('taxonomy_term');
  $this->installEntitySchema('path_alias');
}
```

### installConfig()

Imports default configuration YAML files shipped with modules. Necessary when services or code reads from configuration.

```php
$this->installConfig(['system', 'node', 'my_module']);
```

### installSchema()

Installs non-entity database tables defined in `hook_schema()`.

```php
$this->installSchema('node', ['node_access']);
$this->installSchema('system', ['sequences']);
$this->installSchema('my_module', ['my_module_tracking']);
```

### Typical setUp() Ordering

```php
protected function setUp(): void {
  parent::setUp();

  // 1. Install entity schemas (creates tables for entity types).
  $this->installEntitySchema('user');
  $this->installEntitySchema('node');

  // 2. Install non-entity schemas.
  $this->installSchema('node', ['node_access']);

  // 3. Install module configuration.
  $this->installConfig(['system', 'field', 'node', 'my_module']);

  // 4. Create any required test fixtures.
  $this->createTestContent();
}
```

## Entity CRUD Testing

Kernel tests are ideal for verifying entity creation, loading, updating, and deletion.

```php
<?php

declare(strict_types=1);

namespace Drupal\Tests\my_module\Kernel;

use Drupal\KernelTests\KernelTestBase;
use Drupal\my_module\Entity\Report;

/**
 * Tests the Report entity CRUD operations.
 *
 * @group my_module
 */
class ReportEntityTest extends KernelTestBase {

  protected static $modules = [
    'system',
    'user',
    'my_module',
  ];

  /**
   * {@inheritdoc}
   */
  protected function setUp(): void {
    parent::setUp();
    $this->installEntitySchema('user');
    $this->installEntitySchema('report');
    $this->installConfig(['my_module']);
  }

  /**
   * Tests creating a Report entity.
   */
  public function testCreateReport(): void {
    $report = Report::create([
      'name' => 'Q1 Summary',
      'type' => 'quarterly',
    ]);
    $report->save();

    $this->assertNotEmpty($report->id());
    $this->assertEquals('Q1 Summary', $report->getName());
  }

  /**
   * Tests loading a Report entity.
   */
  public function testLoadReport(): void {
    $report = Report::create([
      'name' => 'Q1 Summary',
      'type' => 'quarterly',
    ]);
    $report->save();

    $loaded = Report::load($report->id());
    $this->assertInstanceOf(Report::class, $loaded);
    $this->assertEquals('Q1 Summary', $loaded->getName());
  }

  /**
   * Tests deleting a Report entity.
   */
  public function testDeleteReport(): void {
    $report = Report::create([
      'name' => 'Temporary',
      'type' => 'quarterly',
    ]);
    $report->save();
    $id = $report->id();

    $report->delete();

    $this->assertNull(Report::load($id));
  }

}
```

## Service Testing

Retrieve services from the container and test them with real (or partially real) dependencies.

```php
/**
 * Tests the report generator service.
 */
public function testReportGeneratorService(): void {
  /** @var \Drupal\my_module\Service\ReportGenerator $generator */
  $generator = $this->container->get('my_module.report_generator');
  $this->assertInstanceOf(ReportGenerator::class, $generator);

  $result = $generator->generate('quarterly');
  $this->assertIsArray($result);
  $this->assertNotEmpty($result);
}
```

### Overriding a Service in the Container

Sometimes you need to swap one dependency with a mock while keeping the rest real.

```php
use Drupal\Core\DependencyInjection\ContainerBuilder;

/**
 * {@inheritdoc}
 */
public function register(ContainerBuilder $container): void {
  parent::register($container);

  // Replace the HTTP client with a mock.
  $http_client = $this->createMock(ClientInterface::class);
  $http_client->method('request')->willReturn(
    new Response(200, [], '{"status": "ok"}')
  );
  $container->set('http_client', $http_client);
}
```

## Database Testing

Kernel tests provide direct access to the database connection.

```php
use Drupal\Core\Database\Connection;

public function testCustomTableInsert(): void {
  /** @var \Drupal\Core\Database\Connection $database */
  $database = $this->container->get('database');

  $database->insert('my_module_tracking')
    ->fields([
      'entity_id' => 1,
      'action' => 'view',
      'timestamp' => \Drupal::time()->getRequestTime(),
    ])
    ->execute();

  $count = $database->select('my_module_tracking', 't')
    ->countQuery()
    ->execute()
    ->fetchField();

  $this->assertEquals(1, (int) $count);
}
```

## Testing Configuration

```php
/**
 * Tests that module installation creates default config.
 */
public function testDefaultConfiguration(): void {
  $config = $this->config('my_module.settings');

  $this->assertEquals(10, $config->get('report_limit'));
  $this->assertTrue($config->get('enabled'));
  $this->assertEquals('default', $config->get('display_mode'));
}

/**
 * Tests programmatic configuration changes.
 */
public function testConfigUpdate(): void {
  $config = $this->config('my_module.settings');
  $config->set('report_limit', 50)->save();

  $reloaded = $this->config('my_module.settings');
  $this->assertEquals(50, $reloaded->get('report_limit'));
}
```

## Testing Events and Hooks

### Testing Event Subscribers

```php
use Drupal\my_module\Event\ReportGeneratedEvent;
use Symfony\Component\EventDispatcher\EventDispatcherInterface;

public function testReportGeneratedEventFires(): void {
  $dispatched = FALSE;

  /** @var \Symfony\Component\EventDispatcher\EventDispatcherInterface $dispatcher */
  $dispatcher = $this->container->get('event_dispatcher');
  $dispatcher->addListener('my_module.report_generated', function () use (&$dispatched) {
    $dispatched = TRUE;
  });

  // Trigger the action that fires the event.
  $generator = $this->container->get('my_module.report_generator');
  $generator->generate('quarterly');

  $this->assertTrue($dispatched, 'The report_generated event was dispatched.');
}
```

### Testing Hook Implementations

If your module implements a hook that alters data, create an entity or trigger the process that invokes the hook and verify the outcome.

```php
/**
 * Tests that my_module_entity_presave sets a default value.
 */
public function testEntityPresaveHook(): void {
  $report = Report::create([
    'name' => 'No Status',
    'type' => 'quarterly',
    // Intentionally omit 'status'.
  ]);
  $report->save();

  // The hook should have set the default status.
  $loaded = Report::load($report->id());
  $this->assertEquals('draft', $loaded->get('status')->value);
}
```

## Using KernelTestBase with User Entities

Many tests require a user context. Create users directly via the entity API.

```php
use Drupal\user\Entity\User;

protected function createTestUser(): User {
  $user = User::create([
    'name' => 'testuser',
    'mail' => 'test@example.com',
    'status' => 1,
  ]);
  $user->save();
  return $user;
}
```

To set the current user for access checks:

```php
$this->container->get('current_user')->setAccount($user);
```

## Testing Queue Workers

```php
public function testQueueWorkerProcessesItem(): void {
  /** @var \Drupal\Core\Queue\QueueFactory $queue_factory */
  $queue_factory = $this->container->get('queue');
  $queue = $queue_factory->get('my_module_import');
  $queue->createItem(['entity_id' => 42, 'action' => 'sync']);

  $this->assertEquals(1, $queue->numberOfItems());

  /** @var \Drupal\Core\Queue\QueueWorkerManagerInterface $manager */
  $manager = $this->container->get('plugin.manager.queue_worker');
  $worker = $manager->createInstance('my_module_import');

  $item = $queue->claimItem();
  $worker->processItem($item->data);
  $queue->deleteItem($item);

  $this->assertEquals(0, $queue->numberOfItems());
}
```

## Common Pitfalls

1. **Missing `$modules` entries** — If your module depends on another module, list it. Kernel tests do not read `.info.yml` dependencies automatically.
2. **Forgetting `installEntitySchema()`** — You will get "table not found" database errors.
3. **Order of install calls** — Install entity schemas before config that references those entities.
4. **Static `$modules`** — Must be `protected static $modules`, not `protected $modules`. A non-static declaration will silently fail.
5. **Accessing services before `parent::setUp()`** — The container is not available until after `parent::setUp()` runs.
