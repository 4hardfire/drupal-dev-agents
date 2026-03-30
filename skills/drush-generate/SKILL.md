---
name: drush-generate
description: Use when scaffolding Drupal code using drush generate (drush gen), generating modules, controllers, forms, plugins, services, hooks, event subscribers, or any drush code generation task.
version: 1.0.0
---

# Drush Code Generation

## Overview

Drush provides a powerful code generation system via `ddev drush gen` (alias for `ddev drush generate`). Generators scaffold boilerplate code for modules, plugins, services, forms, controllers, entities, themes, tests, and YAML configuration files. Each generator creates properly structured files following Drupal coding standards.

Key principles:
- All commands run inside DDEV: `ddev drush gen <generator>`.
- By default, generators run interactively, prompting for values (module name, class name, etc.).
- For automation, use `--answer=` flags or `--answers='{...}'` JSON to supply values non-interactively.
- Generated files are placed relative to the Drupal root (typically `web/modules/custom/`).
- Always run `ddev drush cr` after generating code to rebuild the container and clear caches.

---

## Non-Interactive Mode

Generators accept answers via command-line flags, which is essential for automation and scripting.

### Using `--answers` (JSON)

Pass all answers as a JSON object:

```bash
ddev drush gen module --answers='{"name":"My Module","machine_name":"my_module","description":"A custom module.","package":"Custom","dependencies":"","install_file":false,"libraries_file":false,"permissions_file":false,"event_subscriber":false,"navigation":false}'
```

### Using Individual `--answer` Flags

Pass answers one at a time:

```bash
ddev drush gen controller --answer="Module machine name:my_module" --answer="Class:MyController" --answer="Route name:my_module.example" --answer="Route path:/my-module/example" --answer="Route title:Example Page"
```

### The `--dry-run` Flag

Preview what files will be created without writing anything:

```bash
ddev drush gen module --dry-run --answers='{"name":"My Module","machine_name":"my_module"}'
```

### Listing Available Generators

```bash
ddev drush gen --list
```

### Getting Help for a Generator

```bash
ddev drush gen controller --help
```

---

## Module Generators

### `ddev drush gen module`

Generates a full module scaffold including the `.info.yml` file and optionally `.module`, `composer.json`, install file, libraries file, and permissions file.

```bash
ddev drush gen module --answers='{"name":"Example Module","machine_name":"example_module","description":"Provides example functionality.","package":"Custom","dependencies":"","install_file":false,"libraries_file":false,"permissions_file":false,"event_subscriber":false,"navigation":false}'
```

**Files created:**
- `web/modules/custom/example_module/example_module.info.yml`
- `web/modules/custom/example_module/example_module.module` (if selected)
- `web/modules/custom/example_module/composer.json` (if selected)

---

## Controller & Route Generators

### `ddev drush gen controller`

Generates a controller class with a route entry in the module's routing file.

```bash
ddev drush gen controller --answers='{"module":"my_module","class":"DashboardController","route_name":"my_module.dashboard","route_path":"/my-module/dashboard","route_title":"Dashboard","services":""}'
```

**Files created/modified:**
- `src/Controller/DashboardController.php`
- `my_module.routing.yml` (appended)

---

## Form Generators

### `ddev drush gen form:simple`

Generates a form class extending `FormBase` with a route.

```bash
ddev drush gen form:simple --answers='{"module":"my_module","class":"ContactForm","form_id":"my_module_contact","route_name":"my_module.contact_form","route_path":"/my-module/contact","route_title":"Contact","route_permission":"access content"}'
```

**Files created/modified:**
- `src/Form/ContactForm.php`
- `my_module.routing.yml` (appended)

### `ddev drush gen form:config`

Generates a configuration form extending `ConfigFormBase` with a route.

```bash
ddev drush gen form:config --answers='{"module":"my_module","class":"SettingsForm","form_id":"my_module_settings","route_name":"my_module.settings","route_path":"/admin/config/my-module/settings","route_title":"My Module Settings","route_permission":"administer site configuration"}'
```

**Files created/modified:**
- `src/Form/SettingsForm.php`
- `my_module.routing.yml` (appended)

---

## Plugin Generators

### `ddev drush gen plugin:block`

Generates a Block plugin class.

```bash
ddev drush gen plugin:block --answers='{"module":"my_module","plugin_id":"my_module_featured","plugin_label":"Featured Content","category":"Custom","class":"FeaturedContentBlock","configurable":true,"services":""}'
```

**Files created:**
- `src/Plugin/Block/FeaturedContentBlock.php`

### `ddev drush gen plugin:field:type`

Generates a FieldType plugin.

```bash
ddev drush gen plugin:field:type --answers='{"module":"my_module","plugin_id":"color_field","plugin_label":"Color","class":"ColorItem"}'
```

**Files created:**
- `src/Plugin/Field/FieldType/ColorItem.php`

### `ddev drush gen plugin:field:widget`

Generates a FieldWidget plugin.

```bash
ddev drush gen plugin:field:widget --answers='{"module":"my_module","plugin_id":"color_default","plugin_label":"Color default","class":"ColorDefaultWidget"}'
```

**Files created:**
- `src/Plugin/Field/FieldWidget/ColorDefaultWidget.php`

### `ddev drush gen plugin:field:formatter`

Generates a FieldFormatter plugin.

```bash
ddev drush gen plugin:field:formatter --answers='{"module":"my_module","plugin_id":"color_swatch","plugin_label":"Color swatch","class":"ColorSwatchFormatter"}'
```

**Files created:**
- `src/Plugin/Field/FieldFormatter/ColorSwatchFormatter.php`

### `ddev drush gen plugin:queue-worker`

Generates a QueueWorker plugin for cron-based queue processing.

```bash
ddev drush gen plugin:queue-worker --answers='{"module":"my_module","plugin_id":"my_module_email_queue","plugin_label":"Email queue","class":"EmailQueueWorker"}'
```

**Files created:**
- `src/Plugin/QueueWorker/EmailQueueWorker.php`

### `ddev drush gen plugin:action`

Generates an Action plugin for bulk operations.

```bash
ddev drush gen plugin:action --answers='{"module":"my_module","plugin_id":"my_module_send_email","plugin_label":"Send email","class":"SendEmailAction","category":"Custom","configurable":false}'
```

**Files created:**
- `src/Plugin/Action/SendEmailAction.php`

### `ddev drush gen plugin:condition`

Generates a Condition plugin for visibility rules.

```bash
ddev drush gen plugin:condition --answers='{"module":"my_module","plugin_id":"my_module_business_hours","plugin_label":"Business hours","class":"BusinessHoursCondition"}'
```

**Files created:**
- `src/Plugin/Condition/BusinessHoursCondition.php`

---

## Service Generators

### `ddev drush gen service`

Generates a service class and registers it in the module's `services.yml`.

```bash
ddev drush gen service --answers='{"module":"my_module","service_name":"my_module.data_processor","class":"DataProcessor"}'
```

**Files created/modified:**
- `src/DataProcessor.php`
- `my_module.services.yml` (appended or created)

---

## Event Subscriber Generator

### `ddev drush gen event-subscriber`

Generates an EventSubscriber class and registers it as a tagged service.

```bash
ddev drush gen event-subscriber --answers='{"module":"my_module","class":"RequestSubscriber","event":"Symfony\\Component\\HttpKernel\\KernelEvents::REQUEST"}'
```

**Files created/modified:**
- `src/EventSubscriber/RequestSubscriber.php`
- `my_module.services.yml` (appended with event_subscriber tag)

---

## Hook Generator

### `ddev drush gen hook`

Generates a hook implementation in the module file or as an OOP hook class (Drupal 11+).

```bash
ddev drush gen hook --answers='{"module":"my_module","hook_name":"node_presave"}'
```

**Files modified:**
- `my_module.module` (hook function appended)

---

## Entity Generators

### `ddev drush gen entity:content`

Generates a complete content entity type with base fields, list builder, forms, access handler, and routing.

```bash
ddev drush gen entity:content --answers='{"module":"my_module","entity_type_label":"Task","entity_type_id":"task","class":"Task","base_table":"task","admin_permission":"administer tasks"}'
```

**Files created:**
- `src/Entity/Task.php`
- `src/TaskInterface.php`
- `src/TaskListBuilder.php`
- `src/Form/TaskForm.php`
- `src/Form/TaskDeleteForm.php`
- `src/TaskAccessControlHandler.php`
- `my_module.routing.yml` (appended)
- `my_module.links.menu.yml` (created/appended)
- `my_module.links.task.yml` (created/appended)
- `my_module.links.action.yml` (created/appended)
- `my_module.permissions.yml` (created/appended)

### `ddev drush gen entity:config`

Generates a configuration entity type with form, list builder, and schema.

```bash
ddev drush gen entity:config --answers='{"module":"my_module","entity_type_label":"Workflow","entity_type_id":"workflow","class":"Workflow","admin_permission":"administer workflows"}'
```

**Files created:**
- `src/Entity/Workflow.php`
- `src/WorkflowInterface.php`
- `src/WorkflowListBuilder.php`
- `src/Form/WorkflowForm.php`
- `config/schema/my_module.schema.yml`
- `my_module.routing.yml` (appended)
- `my_module.links.menu.yml`
- `my_module.links.task.yml`
- `my_module.links.action.yml`

---

## Theme Generator

### `ddev drush gen theme`

Generates a theme scaffold with `.info.yml`, `libraries.yml`, and template directories.

```bash
ddev drush gen theme --answers='{"name":"My Theme","machine_name":"my_theme","base_theme":"stable9","description":"Custom theme.","package":"Custom"}'
```

**Files created:**
- `themes/custom/my_theme/my_theme.info.yml`
- `themes/custom/my_theme/my_theme.libraries.yml`
- `themes/custom/my_theme/templates/` directory

---

## Test Generator

### `ddev drush gen test`

Generates a test class (Kernel, Functional, or Unit test).

```bash
ddev drush gen test --answers='{"module":"my_module","class":"DataProcessorTest","type":"kernel"}'
```

**Files created:**
- `tests/src/Kernel/DataProcessorTest.php` (for kernel type)
- `tests/src/Unit/DataProcessorTest.php` (for unit type)
- `tests/src/Functional/DataProcessorTest.php` (for functional type)

---

## YAML File Generators

These generators create or scaffold common module YAML files.

### `ddev drush gen yml:module:services`

Generates a `services.yml` skeleton.

```bash
ddev drush gen yml:module:services --answers='{"module":"my_module"}'
```

### `ddev drush gen yml:module:routing`

Generates a `routing.yml` skeleton.

```bash
ddev drush gen yml:module:routing --answers='{"module":"my_module"}'
```

### `ddev drush gen yml:module:permissions`

Generates a `permissions.yml` skeleton.

```bash
ddev drush gen yml:module:permissions --answers='{"module":"my_module"}'
```

### `ddev drush gen yml:module:links:menu`

Generates a `links.menu.yml` file.

```bash
ddev drush gen yml:module:links:menu --answers='{"module":"my_module"}'
```

### `ddev drush gen yml:module:links:task`

Generates a `links.task.yml` file.

```bash
ddev drush gen yml:module:links:task --answers='{"module":"my_module"}'
```

### `ddev drush gen yml:module:links:action`

Generates a `links.action.yml` file.

```bash
ddev drush gen yml:module:links:action --answers='{"module":"my_module"}'
```

---

## Tips & Best Practices

- **Always clear cache** after generating code: `ddev drush cr`.
- **Review generated code** before committing. Generators produce good scaffolds but may need customization (e.g., adding dependency injection, adjusting permissions, refining form elements).
- **Use `--dry-run`** first to preview what files will be created.
- **Combine generators**: For a full feature, you might use `module` + `controller` + `form:config` + `service` + `plugin:block` in sequence.
- **Check generator availability**: `ddev drush gen --list` shows all installed generators. Contrib modules may add custom generators.
- **Generator names may vary** slightly between Drush versions. Use `ddev drush gen --list` to confirm exact names for your project's Drush version.

---

## Related Skills

- **drupal-patterns** -- Decision framework for choosing which Drupal pattern to use.
