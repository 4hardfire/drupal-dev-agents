# Dependency Injection Patterns in Drupal

## Principles

1. **Constructor injection** is the only recommended injection method in Drupal.
2. Setter injection and property injection are not used in Drupal core.
3. Every class that receives services via `create()` must declare the corresponding constructor parameters.
4. The `create()` method is a static factory — it pulls services from the container and passes them to `__construct()`.

---

## Pattern by Context

### 1. Services (Defined in `*.services.yml`)

Services are the simplest case. The container injects arguments directly — no `create()` method needed.

```php
<?php

namespace Drupal\mymodule;

use Drupal\Core\Config\ConfigFactoryInterface;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Psr\Log\LoggerInterface;

class DataExporter {

  public function __construct(
    protected readonly EntityTypeManagerInterface $entityTypeManager,
    protected readonly ConfigFactoryInterface $configFactory,
    protected readonly LoggerInterface $logger,
  ) {}

  public function export(string $entityType): array {
    $config = $this->configFactory->get('mymodule.settings');
    $storage = $this->entityTypeManager->getStorage($entityType);
    $limit = $config->get('export_limit') ?? 100;
    $entities = $storage->loadMultiple();
    $this->logger->info('Exported @count @type entities.', [
      '@count' => count($entities),
      '@type' => $entityType,
    ]);
    return $entities;
  }

}
```

```yaml
services:
  mymodule.data_exporter:
    class: Drupal\mymodule\DataExporter
    arguments:
      - '@entity_type.manager'
      - '@config.factory'
      - '@logger.channel.mymodule'

  logger.channel.mymodule:
    parent: logger.channel_base
    arguments: ['mymodule']
```

### 2. Controllers

Controllers extend `ControllerBase` which provides helper methods. Override `create()` to inject additional services.

```php
<?php

namespace Drupal\mymodule\Controller;

use Drupal\Core\Controller\ControllerBase;
use Drupal\mymodule\DataExporter;
use Symfony\Component\DependencyInjection\ContainerInterface;
use Symfony\Component\HttpFoundation\JsonResponse;

class ExportController extends ControllerBase {

  public function __construct(
    protected readonly DataExporter $exporter,
  ) {}

  public static function create(ContainerInterface $container): static {
    return new static(
      $container->get('mymodule.data_exporter'),
    );
  }

  public function exportJson(string $entity_type): JsonResponse {
    $data = $this->exporter->export($entity_type);
    return new JsonResponse($data);
  }

}
```

Note: `ControllerBase` already gives you `$this->entityTypeManager()`, `$this->config()`, `$this->currentUser()`, etc. Only inject services that are not already available through those helpers.

### 3. Forms

Forms follow the same pattern as controllers. `FormBase` and `ConfigFormBase` both support `create()`.

```php
<?php

namespace Drupal\mymodule\Form;

use Drupal\Core\Form\FormBase;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\Mail\MailManagerInterface;
use Drupal\Core\Session\AccountProxyInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

class ContactForm extends FormBase {

  public function __construct(
    protected readonly MailManagerInterface $mailManager,
    protected readonly AccountProxyInterface $currentUser,
  ) {}

  public static function create(ContainerInterface $container): static {
    return new static(
      $container->get('plugin.manager.mail'),
      $container->get('current_user'),
    );
  }

  public function getFormId(): string {
    return 'mymodule_contact_form';
  }

  public function buildForm(array $form, FormStateInterface $form_state): array {
    $form['message'] = [
      '#type' => 'textarea',
      '#title' => $this->t('Message'),
      '#required' => TRUE,
    ];
    $form['submit'] = [
      '#type' => 'submit',
      '#value' => $this->t('Send'),
    ];
    return $form;
  }

  public function submitForm(array &$form, FormStateInterface $form_state): void {
    $this->mailManager->mail(
      'mymodule',
      'contact',
      'admin@example.com',
      'en',
      ['message' => $form_state->getValue('message')],
      $this->currentUser->getEmail(),
    );
  }

}
```

### 4. Block Plugins

Plugins use `ContainerFactoryPluginInterface`. The `create()` signature differs from controllers — it includes `$configuration`, `$plugin_id`, and `$plugin_definition`.

```php
<?php

namespace Drupal\mymodule\Plugin\Block;

use Drupal\Core\Block\Attribute\Block;
use Drupal\Core\Block\BlockBase;
use Drupal\Core\Plugin\ContainerFactoryPluginInterface;
use Drupal\Core\Session\AccountProxyInterface;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use Drupal\mymodule\DataExporter;
use Symfony\Component\DependencyInjection\ContainerInterface;

#[Block(
  id: 'mymodule_export_summary',
  admin_label: new TranslatableMarkup('Export Summary'),
)]
class ExportSummaryBlock extends BlockBase implements ContainerFactoryPluginInterface {

  public function __construct(
    array $configuration,
    $plugin_id,
    $plugin_definition,
    protected readonly DataExporter $exporter,
    protected readonly AccountProxyInterface $currentUser,
  ) {
    parent::__construct($configuration, $plugin_id, $plugin_definition);
  }

  public static function create(
    ContainerInterface $container,
    array $configuration,
    $plugin_id,
    $plugin_definition,
  ): static {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition,
      $container->get('mymodule.data_exporter'),
      $container->get('current_user'),
    );
  }

  public function build(): array {
    return [
      '#markup' => $this->t('Exporter ready for user @uid.', [
        '@uid' => $this->currentUser->id(),
      ]),
    ];
  }

}
```

### 5. Entity Classes (Content Entities)

Content entity classes do **not** support standard DI. They are instantiated by the entity storage, not the container. To access services inside an entity class, use `\Drupal::service()` — this is one of the few acceptable places.

Alternatively, move logic into a dedicated service and inject the entity into that service's methods.

```php
// Preferred: dedicated service instead of logic in the entity.
class NodePublisher {

  public function __construct(
    protected readonly EntityTypeManagerInterface $entityTypeManager,
    protected readonly LoggerInterface $logger,
  ) {}

  public function publishNode(int $nid): void {
    $node = $this->entityTypeManager->getStorage('node')->load($nid);
    if ($node && !$node->isPublished()) {
      $node->setPublished()->save();
      $this->logger->info('Published node @nid.', ['@nid' => $nid]);
    }
  }

}
```

### 6. Drush Commands

Drush 12+ commands are Symfony console commands registered as services.

```php
<?php

namespace Drupal\mymodule\Commands;

use Drupal\mymodule\DataExporter;
use Drush\Attributes as CLI;
use Drush\Commands\DrushCommands;

class ExportCommands extends DrushCommands {

  public function __construct(
    protected readonly DataExporter $exporter,
  ) {
    parent::__construct();
  }

  #[CLI\Command(name: 'mymodule:export', aliases: ['mexp'])]
  #[CLI\Argument(name: 'entity_type', description: 'The entity type to export.')]
  #[CLI\Usage(name: 'mymodule:export node', description: 'Export all nodes.')]
  public function export(string $entity_type): void {
    $results = $this->exporter->export($entity_type);
    $this->io()->success(sprintf('Exported %d entities.', count($results)));
  }

}
```

```yaml
services:
  mymodule.commands.export:
    class: Drupal\mymodule\Commands\ExportCommands
    arguments:
      - '@mymodule.data_exporter'
    tags:
      - { name: drush.command }
```

---

## ContainerInjectionInterface

`ContainerInjectionInterface` defines a single method:

```php
public static function create(ContainerInterface $container): static;
```

Use it for any class that Drupal instantiates through the container factory pattern but that is not a service itself (e.g., route controllers not extending `ControllerBase`, custom access checkers passed by class name).

```php
<?php

namespace Drupal\mymodule\Access;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Access\AccessResultInterface;
use Drupal\Core\DependencyInjection\ContainerInjectionInterface;
use Drupal\Core\Session\AccountInterface;
use Drupal\mymodule\PermissionChecker;
use Symfony\Component\DependencyInjection\ContainerInterface;

class CustomAccessCheck implements ContainerInjectionInterface {

  public function __construct(
    protected readonly PermissionChecker $permissionChecker,
  ) {}

  public static function create(ContainerInterface $container): static {
    return new static(
      $container->get('mymodule.permission_checker'),
    );
  }

  public function access(AccountInterface $account): AccessResultInterface {
    return AccessResult::allowedIf(
      $this->permissionChecker->hasSpecialAccess($account)
    );
  }

}
```

---

## ContainerAwareInterface (Deprecated)

`ContainerAwareInterface` (and `ContainerAwareTrait`) inject the entire container. This is **deprecated** in Drupal 10+ and should never be used. Always prefer constructor injection of specific services.

---

## Service Autowiring (Drupal 11.1+)

Drupal 11.1 added support for Symfony's autowiring feature. When enabled, the container resolves constructor type-hints to service IDs automatically.

### Enabling Autowire

```yaml
services:
  _defaults:
    autowire: true

  mymodule.data_exporter:
    class: Drupal\mymodule\DataExporter
    # No 'arguments' key needed — EntityTypeManagerInterface, ConfigFactoryInterface,
    # and LoggerInterface are resolved automatically.
```

### Limitations

- Autowiring only works for services whose interface maps to a single implementation in the container.
- Ambiguous interfaces (e.g., multiple cache bins implementing `CacheBackendInterface`) require explicit arguments.
- Scalar arguments (`%parameter%`) cannot be autowired and must still be listed.

### Combining Autowire with Explicit Arguments

```yaml
services:
  _defaults:
    autowire: true

  mymodule.report_generator:
    class: Drupal\mymodule\ReportGenerator
    arguments:
      $maxResults: '%mymodule.max_results%'
    # All typed service arguments are autowired; only the scalar is explicit.
```

---

## Common Service IDs Reference

| Service ID | Interface / Class | Purpose |
|------------|-------------------|---------|
| `entity_type.manager` | `EntityTypeManagerInterface` | Entity CRUD, storage, view builders |
| `current_user` | `AccountProxyInterface` | Current logged-in user |
| `config.factory` | `ConfigFactoryInterface` | Read/write configuration |
| `messenger` | `MessengerInterface` | Status/warning/error messages |
| `plugin.manager.mail` | `MailManagerInterface` | Send emails |
| `path.validator` | `PathValidatorInterface` | Validate internal paths |
| `path_alias.manager` | `AliasManagerInterface` | Path alias resolution |
| `database` | `Connection` | Database connection |
| `event_dispatcher` | `EventDispatcherInterface` | Dispatch and listen to events |
| `logger.factory` | `LoggerChannelFactoryInterface` | Logging |
| `cache.default` | `CacheBackendInterface` | Default cache bin |
| `http_client` | `ClientInterface` (Guzzle) | HTTP requests |
| `module_handler` | `ModuleHandlerInterface` | Module management, hook invocation |
| `token` | `Token` | Token replacement |
| `string_translation` | `TranslationInterface` | String translation |
| `request_stack` | `RequestStack` | Current HTTP request |
