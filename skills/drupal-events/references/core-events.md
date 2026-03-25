# Drupal Core Events Reference

A comprehensive catalog of events provided by Drupal core, organized by category.

---

## Kernel Events

Provided by Symfony's HttpKernel component. These fire during every HTTP request lifecycle.

**Constant class**: `Symfony\Component\HttpKernel\KernelEvents`

### KernelEvents::REQUEST

- **Event class**: `Symfony\Component\HttpKernel\Event\RequestEvent`
- **When it fires**: Very early in the request, after the container is booted but before routing.
- **Common use cases**:
  - Access checking and authentication
  - Redirecting users based on conditions
  - Setting request attributes
  - Implementing custom caching layers
- **Key methods**: `getRequest()`, `setResponse()`, `isMainRequest()`
- **Note**: Calling `setResponse()` short-circuits the request -- no controller is invoked.

### KernelEvents::CONTROLLER

- **Event class**: `Symfony\Component\HttpKernel\Event\ControllerEvent`
- **When it fires**: After the controller is resolved but before it is executed.
- **Common use cases**:
  - Modifying controller arguments
  - Replacing the controller entirely
  - Logging or profiling controller invocations
- **Key methods**: `getController()`, `setController()`, `getRequest()`

### KernelEvents::CONTROLLER_ARGUMENTS

- **Event class**: `Symfony\Component\HttpKernel\Event\ControllerArgumentsEvent`
- **When it fires**: After controller arguments are resolved, right before the controller is called.
- **Common use cases**:
  - Modifying resolved arguments before controller execution
  - Adding computed arguments
- **Key methods**: `getArguments()`, `setArguments()`, `getController()`

### KernelEvents::VIEW

- **Event class**: `Symfony\Component\HttpKernel\Event\ViewEvent`
- **When it fires**: When a controller returns something other than a `Response` object (e.g., a render array).
- **Common use cases**:
  - Converting render arrays to Response objects (Drupal's MainContentViewSubscriber does this)
  - Handling custom return types from controllers
- **Key methods**: `getControllerResult()`, `setResponse()`, `getRequest()`

### KernelEvents::RESPONSE

- **Event class**: `Symfony\Component\HttpKernel\Event\ResponseEvent`
- **When it fires**: After the controller returns a Response, before it is sent to the client.
- **Common use cases**:
  - Adding or modifying HTTP headers
  - Modifying the response body
  - Setting cache headers
  - Adding security headers (CSP, HSTS)
- **Key methods**: `getResponse()`, `setResponse()`, `getRequest()`, `isMainRequest()`

### KernelEvents::FINISH_REQUEST

- **Event class**: `Symfony\Component\HttpKernel\Event\FinishRequestEvent`
- **When it fires**: After the response for a request (main or subrequest) is complete.
- **Common use cases**:
  - Resetting state that was set for a subrequest
  - Cleanup of request-scoped resources
- **Key methods**: `getRequest()`

### KernelEvents::TERMINATE

- **Event class**: `Symfony\Component\HttpKernel\Event\TerminateEvent`
- **When it fires**: After the response has been sent to the client, before PHP shuts down.
- **Common use cases**:
  - Sending emails (deferred to avoid slowing the response)
  - Logging and analytics
  - Heavy processing that does not affect the response
  - Queue item creation
- **Key methods**: `getRequest()`, `getResponse()`
- **Note**: Not all SAPI configurations support terminate. FastCGI and FPM do.

### KernelEvents::EXCEPTION

- **Event class**: `Symfony\Component\HttpKernel\Event\ExceptionEvent`
- **When it fires**: When an uncaught exception occurs during request handling.
- **Common use cases**:
  - Custom error pages
  - Logging exceptions
  - Converting exceptions to specific HTTP responses
  - Suppressing or wrapping exceptions
- **Key methods**: `getThrowable()`, `setResponse()`, `getRequest()`, `allowCustomResponseCode()`

---

## Entity Events

Events related to entity type and field storage definition changes.

### EntityTypeEvents

**Constant class**: `Drupal\Core\Entity\EntityTypeEvents`

#### EntityTypeEvents::CREATE

- **Event class**: `Drupal\Core\Entity\Event\EntityTypeEvent` (Drupal 10.3+)
- **When it fires**: When a new entity type definition is created.
- **Common use cases**:
  - Reacting to new entity types being installed
  - Setting up related storage or configuration

#### EntityTypeEvents::UPDATE

- **Event class**: `Drupal\Core\Entity\Event\EntityTypeEvent`
- **When it fires**: When an entity type definition is updated.
- **Common use cases**:
  - Reacting to schema changes on entity types
  - Updating dependent data structures

#### EntityTypeEvents::DELETE

- **Event class**: `Drupal\Core\Entity\Event\EntityTypeEvent`
- **When it fires**: When an entity type definition is deleted.
- **Common use cases**:
  - Cleaning up related data when an entity type is removed

### FieldStorageDefinitionEvents

**Constant class**: `Drupal\Core\Field\FieldStorageDefinitionEvents`

#### FieldStorageDefinitionEvents::CREATE

- **Event class**: `Drupal\Core\Field\Event\FieldStorageDefinitionEvent`
- **When it fires**: When a new field storage definition is created.
- **Common use cases**: Reacting to new fields being added to entity types.

#### FieldStorageDefinitionEvents::UPDATE

- **Event class**: `Drupal\Core\Field\Event\FieldStorageDefinitionEvent`
- **When it fires**: When a field storage definition is updated.
- **Common use cases**: Updating indexes or derived data when field schema changes.

#### FieldStorageDefinitionEvents::DELETE

- **Event class**: `Drupal\Core\Field\Event\FieldStorageDefinitionEvent`
- **When it fires**: When a field storage definition is deleted.
- **Common use cases**: Cleaning up data associated with a removed field.

---

## Configuration Events

Events dispatched by the configuration system during import, export, and runtime changes.

**Constant class**: `Drupal\Core\Config\ConfigEvents`

### ConfigEvents::SAVE

- **Event class**: `Drupal\Core\Config\ConfigCrudEvent`
- **When it fires**: After a configuration object is saved.
- **Common use cases**:
  - Invalidating caches when settings change
  - Triggering side effects based on config changes
  - Synchronizing config with external systems
- **Key methods**: `getConfig()`, `isChanged($key)`

### ConfigEvents::DELETE

- **Event class**: `Drupal\Core\Config\ConfigCrudEvent`
- **When it fires**: After a configuration object is deleted.
- **Common use cases**:
  - Cleaning up related data when config is removed
  - Logging configuration deletions

### ConfigEvents::RENAME

- **Event class**: `Drupal\Core\Config\ConfigRenameEvent`
- **When it fires**: When a configuration object is renamed.
- **Common use cases**:
  - Updating references to renamed configuration
- **Key methods**: `getOldName()`, `getConfig()`

### ConfigEvents::COLLECTION_INFO

- **Event class**: `Drupal\Core\Config\ConfigCollectionEvent` (via `ConfigCollectionInfo`)
- **When it fires**: When the system gathers information about available config collections.
- **Common use cases**:
  - Registering custom configuration collections (e.g., language overrides)

### ConfigEvents::IMPORT_VALIDATE

- **Event class**: `Drupal\Core\Config\ConfigImporterEvent`
- **When it fires**: Before a configuration import is executed, during validation.
- **Common use cases**:
  - Preventing unsafe configuration imports
  - Validating config dependencies before import
- **Key methods**: `getConfigImporter()`
- **Note**: Throw a `ConfigImporterException` to block the import.

### ConfigEvents::IMPORT

- **Event class**: `Drupal\Core\Config\ConfigImporterEvent`
- **When it fires**: After configuration import completes successfully.
- **Common use cases**:
  - Running post-import tasks
  - Rebuilding caches or derived data after config sync
  - Notifying external systems of configuration changes

### ConfigEvents::IMPORT_MISSING_CONTENT

- **Event class**: `Drupal\Core\Config\ConfigImporterMissingContentEvent`
- **When it fires**: During import when referenced content entities are missing.
- **Common use cases**:
  - Creating placeholder content for missing references
  - Logging missing content dependencies

---

## Routing Events

Events dispatched during route building and alteration.

**Constant class**: `Drupal\Core\Routing\RoutingEvents`

### RoutingEvents::ALTER

- **Event class**: `Drupal\Core\Routing\RouteBuildEvent`
- **When it fires**: After all routes are collected, before they are compiled.
- **Common use cases**:
  - Altering existing routes (changing access requirements, controllers, paths)
  - Removing routes provided by other modules
  - Adding custom route options or defaults
- **Key methods**: `getRouteCollection()`

### RoutingEvents::DYNAMIC

- **Event class**: `Drupal\Core\Routing\RouteBuildEvent`
- **When it fires**: When dynamic routes are being collected.
- **Common use cases**:
  - Adding routes that are computed at runtime (e.g., entity routes, Views page routes)
  - Creating routes based on configuration or database state
- **Key methods**: `getRouteCollection()`

### RoutingEvents::FINISHED

- **Event class**: `Drupal\Component\EventDispatcher\Event`
- **When it fires**: After the route collection is fully built and compiled.
- **Common use cases**:
  - Rebuilding caches that depend on the complete route table
  - Logging route build completion

---

## User and Authentication Events

### AccountEvents (Drupal 11.1+)

**Constant class**: `Drupal\Core\Session\AccountEvents`

#### AccountEvents::SET_USER

- **Event class**: `Drupal\Core\Session\Event\AccountSetEvent`
- **When it fires**: When the current session account is set.
- **Common use cases**:
  - Reacting to user session initialization
  - Loading user-specific settings when a session starts

### User Login/Logout

These are still primarily handled via hooks (`hook_user_login`, `hook_user_logout`) in most Drupal versions. Check your Drupal version for event-based alternatives.

---

## Migration Events

Events dispatched during content migration operations.

**Constant class**: `Drupal\migrate\Event\MigrateEvents`

### MigrateEvents::MAP_SAVE

- **Event class**: `Drupal\migrate\Event\MigrateMapSaveEvent`
- **When it fires**: After a row mapping is saved in the migration map table.
- **Common use cases**: Tracking migrated items, building cross-references.

### MigrateEvents::MAP_DELETE

- **Event class**: `Drupal\migrate\Event\MigrateMapDeleteEvent`
- **When it fires**: After a row mapping is deleted from the migration map table.
- **Common use cases**: Cleaning up related data when migration mappings are removed.

### MigrateEvents::PRE_IMPORT

- **Event class**: `Drupal\migrate\Event\MigrateImportEvent`
- **When it fires**: Before a migration import begins.
- **Common use cases**:
  - Preparing the environment for migration (disabling hooks, setting up state)
  - Logging migration start times

### MigrateEvents::POST_IMPORT

- **Event class**: `Drupal\migrate\Event\MigrateImportEvent`
- **When it fires**: After a migration import completes.
- **Common use cases**:
  - Running post-migration cleanup
  - Rebuilding caches after bulk imports
  - Sending notifications about migration completion

### MigrateEvents::PRE_ROW_SAVE

- **Event class**: `Drupal\migrate\Event\MigratePreRowSaveEvent`
- **When it fires**: Before an individual row is saved during migration.
- **Common use cases**:
  - Modifying row data before save
  - Skipping rows based on custom logic
- **Key methods**: `getRow()`, `getMigration()`

### MigrateEvents::POST_ROW_SAVE

- **Event class**: `Drupal\migrate\Event\MigratePostRowSaveEvent`
- **When it fires**: After an individual row is saved during migration.
- **Common use cases**:
  - Creating related entities for the migrated item
  - Logging per-row migration results
- **Key methods**: `getRow()`, `getDestinationIdValues()`

### MigrateEvents::PRE_ROLLBACK

- **Event class**: `Drupal\migrate\Event\MigrateRollbackEvent`
- **When it fires**: Before a migration rollback begins.
- **Common use cases**: Preparing for rollback operations.

### MigrateEvents::POST_ROLLBACK

- **Event class**: `Drupal\migrate\Event\MigrateRollbackEvent`
- **When it fires**: After a migration rollback completes.
- **Common use cases**: Post-rollback cleanup tasks.

### MigrateEvents::IDMAP_MESSAGE

- **Event class**: `Drupal\migrate\Event\MigrateIdMapMessageEvent`
- **When it fires**: When a message is logged for a specific source ID during migration.
- **Common use cases**: Custom error handling and reporting for migration issues.

---

## Layout Builder Events

Events dispatched by the Layout Builder module.

**Constant class**: `Drupal\layout_builder\Event\SectionComponentBuildRenderArrayEvent` (event object, not constant class)

### layout_builder.prepare_layout

- **Event class**: `Drupal\layout_builder\Event\PrepareLayoutEvent`
- **When it fires**: Before a layout is rendered, after sections are loaded.
- **Common use cases**:
  - Adding default sections to empty layouts
  - Modifying layout sections before rendering

### layout_builder.section_component.build_render_array

- **Event class**: `Drupal\layout_builder\Event\SectionComponentBuildRenderArrayEvent`
- **When it fires**: When a section component (block in a layout) builds its render array.
- **Common use cases**:
  - Altering block output within Layout Builder
  - Adding wrappers or modifying render arrays for layout components
- **Key methods**: `getBuild()`, `setBuild()`, `getComponent()`, `getContexts()`

---

## Subscribing to Core Events -- Quick Example

```php
<?php

declare(strict_types=1);

namespace Drupal\my_module\EventSubscriber;

use Drupal\Core\Config\ConfigCrudEvent;
use Drupal\Core\Config\ConfigEvents;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\ExceptionEvent;
use Symfony\Component\HttpKernel\KernelEvents;

/**
 * Subscribes to multiple core events.
 */
class CoreEventsSubscriber implements EventSubscriberInterface {

  /**
   * {@inheritdoc}
   */
  public static function getSubscribedEvents(): array {
    return [
      KernelEvents::EXCEPTION => ['onException', 50],
      ConfigEvents::SAVE => ['onConfigSave', 0],
    ];
  }

  /**
   * Handles uncaught exceptions.
   */
  public function onException(ExceptionEvent $event): void {
    $exception = $event->getThrowable();
    // Custom exception handling logic.
  }

  /**
   * Reacts to configuration saves.
   */
  public function onConfigSave(ConfigCrudEvent $event): void {
    $config = $event->getConfig();
    if ($config->getName() === 'my_module.settings') {
      // React to this module's settings being changed.
    }
  }

}
```

```yaml
services:
  my_module.core_events_subscriber:
    class: Drupal\my_module\EventSubscriber\CoreEventsSubscriber
    tags:
      - { name: event_subscriber }
```
