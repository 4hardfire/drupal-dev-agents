# Unit Testing in Drupal — Deep Dive

## Base Class and Namespace

All Drupal unit tests extend `Drupal\Tests\UnitTestCase`, which itself extends `PHPUnit\Framework\TestCase`. UnitTestCase provides a few Drupal-specific helpers such as `getStringTranslationStub()` and `getContainerWithCacheTagsInvalidator()`.

```php
<?php

declare(strict_types=1);

namespace Drupal\Tests\my_module\Unit;

use Drupal\Tests\UnitTestCase;

/**
 * @group my_module
 */
class ExampleTest extends UnitTestCase {
}
```

## Mocking Services

When testing a service in isolation, mock all its dependencies.

### Constructor Injection Pattern

Given a service class:

```php
namespace Drupal\my_module\Service;

use Drupal\Core\Config\ConfigFactoryInterface;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Psr\Log\LoggerInterface;

class ReportGenerator {

  public function __construct(
    protected readonly EntityTypeManagerInterface $entityTypeManager,
    protected readonly ConfigFactoryInterface $configFactory,
    protected readonly LoggerInterface $logger,
  ) {}

  public function generate(string $type): array {
    $config = $this->configFactory->get('my_module.settings');
    $limit = $config->get('report_limit') ?? 10;
    // ...
    return $results;
  }

}
```

The corresponding unit test:

```php
<?php

declare(strict_types=1);

namespace Drupal\Tests\my_module\Unit\Service;

use Drupal\Core\Config\ConfigFactoryInterface;
use Drupal\Core\Config\ImmutableConfig;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\my_module\Service\ReportGenerator;
use Drupal\Tests\UnitTestCase;
use Psr\Log\LoggerInterface;

/**
 * @coversDefaultClass \Drupal\my_module\Service\ReportGenerator
 * @group my_module
 */
class ReportGeneratorTest extends UnitTestCase {

  protected ReportGenerator $generator;
  protected EntityTypeManagerInterface $entityTypeManager;
  protected ConfigFactoryInterface $configFactory;
  protected LoggerInterface $logger;

  /**
   * {@inheritdoc}
   */
  protected function setUp(): void {
    parent::setUp();

    $this->entityTypeManager = $this->createMock(EntityTypeManagerInterface::class);
    $this->logger = $this->createMock(LoggerInterface::class);

    $config = $this->createMock(ImmutableConfig::class);
    $config->method('get')
      ->with('report_limit')
      ->willReturn(25);

    $this->configFactory = $this->createMock(ConfigFactoryInterface::class);
    $this->configFactory->method('get')
      ->with('my_module.settings')
      ->willReturn($config);

    $this->generator = new ReportGenerator(
      $this->entityTypeManager,
      $this->configFactory,
      $this->logger,
    );
  }

  /**
   * @covers ::generate
   */
  public function testGenerateReturnsArray(): void {
    $result = $this->generator->generate('summary');
    $this->assertIsArray($result);
  }

}
```

## Mocking Entity Objects

Entities are complex objects. In unit tests, mock only the interfaces you need.

```php
use Drupal\node\NodeInterface;
use Drupal\Core\Field\FieldItemListInterface;

$field_list = $this->createMock(FieldItemListInterface::class);
$field_list->method('getString')->willReturn('Published');

$node = $this->createMock(NodeInterface::class);
$node->method('id')->willReturn('42');
$node->method('getTitle')->willReturn('Test Article');
$node->method('bundle')->willReturn('article');
$node->method('isPublished')->willReturn(TRUE);
$node->method('get')
  ->with('field_status')
  ->willReturn($field_list);
```

### Mocking Entity Storage and Queries

```php
use Drupal\Core\Entity\EntityStorageInterface;
use Drupal\Core\Entity\Query\QueryInterface;

$query = $this->createMock(QueryInterface::class);
$query->method('condition')->willReturnSelf();
$query->method('range')->willReturnSelf();
$query->method('accessCheck')->willReturnSelf();
$query->method('execute')->willReturn(['1' => '1', '2' => '2']);

$storage = $this->createMock(EntityStorageInterface::class);
$storage->method('getQuery')->willReturn($query);
$storage->method('loadMultiple')
  ->with(['1', '2'])
  ->willReturn([$node1, $node2]);

$entity_type_manager = $this->createMock(EntityTypeManagerInterface::class);
$entity_type_manager->method('getStorage')
  ->with('node')
  ->willReturn($storage);
```

## Data Providers

Data providers keep tests clean when verifying multiple input/output combinations.

```php
/**
 * @covers ::calculateDiscount
 * @dataProvider discountProvider
 */
public function testCalculateDiscount(float $price, int $quantity, float $expected): void {
  $calculator = new PriceCalculator();
  $this->assertEqualsWithDelta($expected, $calculator->calculateDiscount($price, $quantity), 0.01);
}

/**
 * Data provider for testCalculateDiscount().
 */
public static function discountProvider(): array {
  return [
    'no discount under 5 items' => [100.00, 3, 0.00],
    '10% discount at 5 items' => [100.00, 5, 10.00],
    '20% discount at 10+ items' => [50.00, 10, 10.00],
    'zero price' => [0.00, 10, 0.00],
  ];
}
```

**Rules for data providers:**
- The method must be `public static` (PHPUnit 10+) or `public` (PHPUnit 9).
- It returns an array of arrays. Each inner array maps to the test method parameters in order.
- Use descriptive string keys for readable failure output.

## Test Doubles Patterns

### Stub vs. Mock

- **Stub**: Provides predetermined return values. You do not verify that methods are called.
- **Mock**: Sets expectations on which methods are called, how many times, and with what arguments.

```php
// Stub: we only care about return values.
$cache = $this->createMock(CacheBackendInterface::class);
$cache->method('get')->willReturn(FALSE);

// Mock: we verify the method is called.
$cache = $this->createMock(CacheBackendInterface::class);
$cache->expects($this->once())
  ->method('set')
  ->with('my_cache_key', $this->anything(), $this->anything());
```

### willReturn Variants

```php
$mock->method('load')->willReturn($entity);
$mock->method('load')->willReturnMap([
  ['1', $entity_one],
  ['2', $entity_two],
]);
$mock->method('process')->willThrowException(new \RuntimeException('fail'));
$mock->method('next')->willReturnOnConsecutiveCalls('first', 'second', 'third');
$mock->method('transform')->willReturnCallback(function ($value) {
  return strtoupper($value);
});
```

## Testing Exceptions

```php
public function testInvalidInputThrowsException(): void {
  $this->expectException(\InvalidArgumentException::class);
  $this->expectExceptionMessage('Type cannot be empty');

  $service = new MyService($this->createMock(LoggerInterface::class));
  $service->process('');
}
```

## The getStringTranslationStub() Helper

UnitTestCase provides a stub for the string translation service. Use it when your class depends on `StringTranslationTrait`:

```php
protected function setUp(): void {
  parent::setUp();

  $this->service = new MyService();
  $this->service->setStringTranslation($this->getStringTranslationStub());
}
```

This stub returns the input string unchanged, so assertions can match against the raw text.

## Testing Traits and Abstract Classes

```php
// For traits, create an anonymous class that uses the trait.
$object = new class() {
  use MyCustomTrait;
};
$this->assertEquals('expected', $object->traitMethod('input'));

// For abstract classes, use getMockForAbstractClass().
$mock = $this->getMockForAbstractClass(AbstractProcessor::class, [
  $this->createMock(LoggerInterface::class),
]);
$mock->method('abstractMethod')->willReturn('value');
$this->assertEquals('processed: value', $mock->concreteMethod());
```
