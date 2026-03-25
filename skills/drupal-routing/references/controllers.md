# Controllers Reference

Complete reference for Drupal route controllers, response types, and related patterns.

## ControllerBase

`Drupal\Core\Controller\ControllerBase` is the recommended base class. It provides
shortcut methods for common services and implements `ContainerInjectionInterface`
for dependency injection.

### Built-in Helper Methods

| Method | Returns | Description |
|---|---|---|
| `entityTypeManager()` | `EntityTypeManagerInterface` | Access entity storage and view builders. |
| `config($name)` | `ImmutableConfig` | Load a configuration object. |
| `currentUser()` | `AccountProxyInterface` | The current user account. |
| `moduleHandler()` | `ModuleHandlerInterface` | Invoke hooks and check module status. |
| `formBuilder()` | `FormBuilderInterface` | Build and retrieve forms. |
| `languageManager()` | `LanguageManagerInterface` | Get language information. |
| `redirect($route, $params, $options)` | `RedirectResponse` | Redirect to a route. |
| `t($string, $args, $options)` | `TranslatableMarkup` | Translate a string. |

### Full Controller Example

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Controller;

use Drupal\Core\Controller\ControllerBase;
use Drupal\Core\Datetime\DateFormatterInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;
use Symfony\Component\HttpFoundation\Request;

class ReportController extends ControllerBase {

  public function __construct(
    protected DateFormatterInterface $dateFormatter,
  ) {}

  public static function create(ContainerInterface $container): static {
    return new static(
      $container->get('date.formatter'),
    );
  }

  public function overview(Request $request): array {
    $build = [];
    $build['header'] = [
      '#markup' => '<p>' . $this->t('Report generated on @date.', [
        '@date' => $this->dateFormatter->format(\time(), 'short'),
      ]) . '</p>',
    ];
    $build['table'] = [
      '#type' => 'table',
      '#header' => [$this->t('ID'), $this->t('Title'), $this->t('Status')],
      '#rows' => $this->getRows(),
      '#empty' => $this->t('No data available.'),
    ];
    return $build;
  }

  protected function getRows(): array {
    // Build table rows from data source.
    return [];
  }

}
```

## ContainerInjectionInterface (Without ControllerBase)

When you do not want the helper methods from `ControllerBase`, implement the
interface directly:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\Controller;

use Drupal\Core\DependencyInjection\ContainerInjectionInterface;
use Drupal\Core\StringTranslation\StringTranslationTrait;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

class LightweightController implements ContainerInjectionInterface {

  use StringTranslationTrait;

  public function __construct(
    protected EntityTypeManagerInterface $entityTypeManager,
  ) {}

  public static function create(ContainerInterface $container): static {
    return new static(
      $container->get('entity_type.manager'),
    );
  }

  public function list(): array {
    $storage = $this->entityTypeManager->getStorage('node');
    $nodes = $storage->loadMultiple();
    $items = [];
    foreach ($nodes as $node) {
      $items[] = $node->label();
    }
    return [
      '#theme' => 'item_list',
      '#items' => $items,
    ];
  }

}
```

## Controller as a Service

Register the controller in `my_module.services.yml` for explicit dependency
wiring. This is an alternative to the `create()` factory.

```yaml
services:
  my_module.custom_controller:
    class: Drupal\my_module\Controller\CustomController
    arguments: ['@entity_type.manager', '@logger.factory']
```

Reference it in routing YAML with the service ID:

```yaml
my_module.custom:
  path: '/custom'
  defaults:
    _controller: 'my_module.custom_controller::content'
    _title: 'Custom Page'
  requirements:
    _permission: 'access content'
```

## Response Types

### Render Array (Default)

The most common return type. Drupal's render pipeline converts it to HTML.

```php
public function page(): array {
  return [
    '#theme' => 'my_template',
    '#data' => $this->getData(),
    '#cache' => [
      'contexts' => ['user.permissions'],
      'tags' => ['node_list'],
      'max-age' => 3600,
    ],
  ];
}
```

### HtmlResponse

Return raw HTML without going through the render pipeline:

```php
use Drupal\Core\Render\HtmlResponse;

public function rawHtml(): HtmlResponse {
  $html = '<html><body><h1>Raw HTML</h1></body></html>';
  return new HtmlResponse($html);
}
```

### JsonResponse

Return JSON data. Ideal for AJAX callbacks and lightweight API endpoints.

```php
use Symfony\Component\HttpFoundation\JsonResponse;

public function data(): JsonResponse {
  $data = [
    'items' => $this->loadItems(),
    'total' => $this->countItems(),
  ];
  return new JsonResponse($data);
}
```

### RedirectResponse

Redirect to another route within the same site:

```php
use Symfony\Component\HttpFoundation\RedirectResponse;
use Drupal\Core\Url;

public function afterSubmit(): RedirectResponse {
  $url = Url::fromRoute('my_module.overview')->toString();
  return new RedirectResponse($url);
}
```

Or use the `ControllerBase::redirect()` shortcut:

```php
public function afterSubmit(): RedirectResponse {
  return $this->redirect('my_module.overview');
}
```

### TrustedRedirectResponse

Redirect to an external URL. Drupal blocks external redirects by default as a
security measure; `TrustedRedirectResponse` explicitly allows it.

```php
use Drupal\Core\Routing\TrustedRedirectResponse;

public function externalRedirect(): TrustedRedirectResponse {
  return new TrustedRedirectResponse('https://example.com/callback');
}
```

### BinaryFileResponse

Serve a file download:

```php
use Symfony\Component\HttpFoundation\BinaryFileResponse;
use Symfony\Component\HttpFoundation\ResponseHeaderBag;

public function download(): BinaryFileResponse {
  $path = \Drupal::service('file_system')->realpath('public://reports/export.csv');
  $response = new BinaryFileResponse($path);
  $response->setContentDisposition(
    ResponseHeaderBag::DISPOSITION_ATTACHMENT,
    'export.csv'
  );
  return $response;
}
```

### StreamedResponse

Stream large output without buffering the entire response in memory:

```php
use Symfony\Component\HttpFoundation\StreamedResponse;

public function stream(): StreamedResponse {
  return new StreamedResponse(function () {
    $handle = fopen('php://output', 'w');
    fputcsv($handle, ['ID', 'Title', 'Created']);
    foreach ($this->getRecords() as $record) {
      fputcsv($handle, [$record->id, $record->title, $record->created]);
    }
    fclose($handle);
  }, 200, [
    'Content-Type' => 'text/csv',
    'Content-Disposition' => 'attachment; filename="data.csv"',
  ]);
}
```

### CacheableJsonResponse

A `JsonResponse` that carries cacheability metadata so Drupal's Dynamic Page
Cache can cache it properly:

```php
use Drupal\Core\Cache\CacheableJsonResponse;
use Drupal\Core\Cache\CacheableMetadata;

public function cachedJson(): CacheableJsonResponse {
  $data = ['items' => $this->loadItems()];
  $response = new CacheableJsonResponse($data);
  $cache_metadata = new CacheableMetadata();
  $cache_metadata->setCacheTags(['node_list']);
  $cache_metadata->setCacheContexts(['url.query_args']);
  $response->addCacheableDependency($cache_metadata);
  return $response;
}
```

## Title Callbacks

### Static Title

Defined in routing YAML:

```yaml
defaults:
  _title: 'My Page Title'
```

### Dynamic Title Callback

```yaml
defaults:
  _title_callback: '\Drupal\my_module\Controller\ItemController::title'
```

The callback method receives route parameters via upcasting:

```php
use Drupal\node\NodeInterface;

public function title(NodeInterface $node): string {
  return $this->t('Edit: @title', ['@title' => $node->label()]);
}
```

The title callback is invoked independently from the main controller method. It
must be a public method on any class that Drupal can instantiate (typically the
same controller).

## Lazy Builders

For expensive or highly dynamic content on an otherwise cacheable page, use lazy
builders to defer rendering:

```php
public function page(): array {
  return [
    'static_content' => [
      '#markup' => '<p>This part is cacheable.</p>',
    ],
    'dynamic_content' => [
      '#lazy_builder' => [
        static::class . '::lazyDynamic',
        [],
      ],
      '#create_placeholder' => TRUE,
    ],
  ];
}

public static function lazyDynamic(): array {
  return [
    '#markup' => '<p>User-specific content: ' . \Drupal::currentUser()->getDisplayName() . '</p>',
    '#cache' => [
      'contexts' => ['user'],
    ],
  ];
}
```

The lazy builder must be a `public static` method. Its arguments must be scalar
values only (strings, integers, booleans). Drupal replaces the placeholder with
the rendered result after the main response is cached.

## Breadcrumb Integration

Controllers do not set breadcrumbs directly. Instead, implement a breadcrumb
builder service:

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module;

use Drupal\Core\Breadcrumb\Breadcrumb;
use Drupal\Core\Breadcrumb\BreadcrumbBuilderInterface;
use Drupal\Core\Link;
use Drupal\Core\Routing\RouteMatchInterface;
use Drupal\Core\StringTranslation\StringTranslationTrait;

class MyModuleBreadcrumbBuilder implements BreadcrumbBuilderInterface {

  use StringTranslationTrait;

  public function applies(RouteMatchInterface $route_match): bool {
    return str_starts_with($route_match->getRouteName() ?? '', 'my_module.');
  }

  public function build(RouteMatchInterface $route_match): Breadcrumb {
    $breadcrumb = new Breadcrumb();
    $breadcrumb->addLink(Link::createFromRoute($this->t('Home'), '<front>'));
    $breadcrumb->addLink(Link::createFromRoute($this->t('My Module'), 'my_module.overview'));
    $breadcrumb->addCacheContexts(['route']);
    return $breadcrumb;
  }

}
```

Register in `my_module.services.yml`:

```yaml
services:
  my_module.breadcrumb_builder:
    class: Drupal\my_module\MyModuleBreadcrumbBuilder
    tags:
      - { name: breadcrumb_builder, priority: 100 }
```

## Common Patterns

### Injecting the Current Request

The `Request` object is available as a controller method argument without any
special configuration:

```php
use Symfony\Component\HttpFoundation\Request;

public function search(Request $request): array {
  $keyword = $request->query->get('q', '');
  // ...
}
```

### Returning a 404 or 403

Throw a Symfony exception to trigger an error response:

```php
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;
use Symfony\Component\HttpKernel\Exception\AccessDeniedHttpException;

public function view(string $slug): array {
  $item = $this->loadBySlug($slug);
  if (!$item) {
    throw new NotFoundHttpException();
  }
  if (!$item->isPublished()) {
    throw new AccessDeniedHttpException('This item is not published.');
  }
  return ['#markup' => $item->label()];
}
```

### Cache Metadata on Render Arrays

Always declare cache metadata so that Drupal can invalidate correctly:

```php
public function list(): array {
  $items = $this->entityTypeManager()->getStorage('node')->loadMultiple();
  $build = [
    '#theme' => 'item_list',
    '#items' => [],
    '#cache' => [
      'tags' => ['node_list'],
      'contexts' => ['user.permissions', 'url.query_args.pagers:0'],
      'max-age' => 600,
    ],
  ];
  foreach ($items as $node) {
    $build['#items'][] = $node->label();
  }
  return $build;
}
```
