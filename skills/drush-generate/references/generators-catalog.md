# Drush Generators — Complete Catalog

All commands use the `ddev drush gen` prefix (DDEV context). Generator names are valid for Drush 12/13 (Drupal 10.2+/11+). Run `ddev drush gen --list` to confirm available generators for your project.

---

## Module Generators

### `module`

Generates a complete module scaffold.

```bash
ddev drush gen module
```

**Key answers:** name, machine_name, description, package, dependencies, install_file, libraries_file, permissions_file, event_subscriber, navigation
**Output files:**
- `my_module.info.yml`
- `my_module.module` (optional)
- `composer.json` (optional)
- `my_module.install` (optional)
- `my_module.libraries.yml` (optional)
- `my_module.permissions.yml` (optional)

### `module-file`

Generates a `.module` file for an existing module.

```bash
ddev drush gen module-file --answers='{"module":"my_module"}'
```

**Output files:**
- `my_module.module`

### `install-file`

Generates a `.install` file with install/uninstall hooks.

```bash
ddev drush gen install-file --answers='{"module":"my_module"}'
```

**Output files:**
- `my_module.install`

---

## Controller & Route Generators

### `controller`

Generates a controller class and routing entry.

```bash
ddev drush gen controller
```

**Key answers:** module, class, route_name, route_path, route_title, services
**Output files:**
- `src/Controller/{Class}.php`
- `my_module.routing.yml` (appended)

---

## Form Generators

### `form:simple`

Generates a form extending `FormBase`.

```bash
ddev drush gen form:simple
```

**Key answers:** module, class, form_id, route_name, route_path, route_title, route_permission
**Output files:**
- `src/Form/{Class}.php`
- `my_module.routing.yml` (appended)

### `form:config`

Generates a configuration form extending `ConfigFormBase`.

```bash
ddev drush gen form:config
```

**Key answers:** module, class, form_id, route_name, route_path, route_title, route_permission
**Output files:**
- `src/Form/{Class}.php`
- `my_module.routing.yml` (appended)

### `form:confirm`

Generates a confirmation form extending `ConfirmFormBase`.

```bash
ddev drush gen form:confirm
```

**Key answers:** module, class, form_id, route_name, route_path, route_title
**Output files:**
- `src/Form/{Class}.php`
- `my_module.routing.yml` (appended)

---

## Plugin Generators

### `plugin:block`

Generates a Block plugin.

```bash
ddev drush gen plugin:block
```

**Key answers:** module, plugin_id, plugin_label, category, class, configurable, services
**Output files:**
- `src/Plugin/Block/{Class}.php`

### `plugin:field:type`

Generates a FieldType plugin (defines how a field stores data).

```bash
ddev drush gen plugin:field:type
```

**Key answers:** module, plugin_id, plugin_label, class
**Output files:**
- `src/Plugin/Field/FieldType/{Class}.php`

### `plugin:field:widget`

Generates a FieldWidget plugin (defines how a field is edited).

```bash
ddev drush gen plugin:field:widget
```

**Key answers:** module, plugin_id, plugin_label, class
**Output files:**
- `src/Plugin/Field/FieldWidget/{Class}.php`

### `plugin:field:formatter`

Generates a FieldFormatter plugin (defines how a field is displayed).

```bash
ddev drush gen plugin:field:formatter
```

**Key answers:** module, plugin_id, plugin_label, class
**Output files:**
- `src/Plugin/Field/FieldFormatter/{Class}.php`

### `plugin:queue-worker`

Generates a QueueWorker plugin for processing queue items during cron.

```bash
ddev drush gen plugin:queue-worker
```

**Key answers:** module, plugin_id, plugin_label, class
**Output files:**
- `src/Plugin/QueueWorker/{Class}.php`

### `plugin:action`

Generates an Action plugin for entity bulk operations.

```bash
ddev drush gen plugin:action
```

**Key answers:** module, plugin_id, plugin_label, class, category, configurable
**Output files:**
- `src/Plugin/Action/{Class}.php`

### `plugin:condition`

Generates a Condition plugin for block visibility and context rules.

```bash
ddev drush gen plugin:condition
```

**Key answers:** module, plugin_id, plugin_label, class
**Output files:**
- `src/Plugin/Condition/{Class}.php`

### `plugin:filter`

Generates a text Filter plugin for the filter system.

```bash
ddev drush gen plugin:filter
```

**Key answers:** module, plugin_id, plugin_label, class, filter_type
**Output files:**
- `src/Plugin/Filter/{Class}.php`

### `plugin:menu-link`

Generates a deriver-based menu link plugin.

```bash
ddev drush gen plugin:menu-link
```

**Key answers:** module, class
**Output files:**
- `src/Plugin/Menu/{Class}.php`

### `plugin:rest-resource`

Generates a REST resource plugin.

```bash
ddev drush gen plugin:rest-resource
```

**Key answers:** module, plugin_id, plugin_label, class
**Output files:**
- `src/Plugin/rest/resource/{Class}.php`

### `plugin:views:field`

Generates a Views field handler plugin.

```bash
ddev drush gen plugin:views:field
```

**Key answers:** module, plugin_id, class
**Output files:**
- `src/Plugin/views/field/{Class}.php`

### `plugin:views:filter`

Generates a Views filter handler plugin.

```bash
ddev drush gen plugin:views:filter
```

**Key answers:** module, plugin_id, class
**Output files:**
- `src/Plugin/views/filter/{Class}.php`

### `plugin:views:argument`

Generates a Views argument (contextual filter) handler plugin.

```bash
ddev drush gen plugin:views:argument
```

**Key answers:** module, plugin_id, class
**Output files:**
- `src/Plugin/views/argument/{Class}.php`

### `plugin:views:sort`

Generates a Views sort handler plugin.

```bash
ddev drush gen plugin:views:sort
```

**Key answers:** module, plugin_id, class
**Output files:**
- `src/Plugin/views/sort/{Class}.php`

### `plugin:views:style`

Generates a Views style plugin (controls overall output format).

```bash
ddev drush gen plugin:views:style
```

**Key answers:** module, plugin_id, plugin_label, class
**Output files:**
- `src/Plugin/views/style/{Class}.php`

### `plugin:ckeditor`

Generates a CKEditor 5 plugin.

```bash
ddev drush gen plugin:ckeditor
```

**Key answers:** module, plugin_id, plugin_label, class
**Output files:**
- `src/Plugin/CKEditor5Plugin/{Class}.php`

### `plugin-manager`

Generates a custom plugin type with a plugin manager, interface, and attribute class.

```bash
ddev drush gen plugin-manager
```

**Key answers:** module, plugin_type, class
**Output files:**
- `src/Plugin/{PluginType}Manager.php`
- `src/Plugin/{PluginType}Interface.php`
- `src/Attribute/{PluginType}.php`
- `my_module.services.yml` (appended)

---

## Service Generators

### `service`

Generates a service class and registers it in `services.yml`.

```bash
ddev drush gen service
```

**Key answers:** module, service_name, class
**Output files:**
- `src/{Class}.php`
- `my_module.services.yml` (appended or created)

### `service-provider`

Generates a ServiceProvider class for modifying the container at build time.

```bash
ddev drush gen service-provider
```

**Key answers:** module
**Output files:**
- `src/{Module}ServiceProvider.php`

### `middleware`

Generates an HTTP middleware class and service definition.

```bash
ddev drush gen middleware
```

**Key answers:** module, class
**Output files:**
- `src/{Class}.php`
- `my_module.services.yml` (appended with http_middleware tag)

---

## Event Subscriber Generator

### `event-subscriber`

Generates an EventSubscriber class with service registration.

```bash
ddev drush gen event-subscriber
```

**Key answers:** module, class, event
**Output files:**
- `src/EventSubscriber/{Class}.php`
- `my_module.services.yml` (appended with event_subscriber tag)

---

## Hook Generator

### `hook`

Generates a hook implementation stub.

```bash
ddev drush gen hook
```

**Key answers:** module, hook_name
**Output files:**
- `my_module.module` (hook function appended)

---

## Entity Generators

### `entity:content`

Generates a full content entity type with all supporting classes and configuration.

```bash
ddev drush gen entity:content
```

**Key answers:** module, entity_type_label, entity_type_id, class, base_table, admin_permission
**Output files:**
- `src/Entity/{Class}.php`
- `src/{Class}Interface.php`
- `src/{Class}ListBuilder.php`
- `src/Form/{Class}Form.php`
- `src/Form/{Class}DeleteForm.php`
- `src/{Class}AccessControlHandler.php`
- `my_module.routing.yml`
- `my_module.links.menu.yml`
- `my_module.links.task.yml`
- `my_module.links.action.yml`
- `my_module.permissions.yml`
- `templates/{entity-type-id}.html.twig`

### `entity:config`

Generates a configuration entity type with form, list builder, and config schema.

```bash
ddev drush gen entity:config
```

**Key answers:** module, entity_type_label, entity_type_id, class, admin_permission
**Output files:**
- `src/Entity/{Class}.php`
- `src/{Class}Interface.php`
- `src/{Class}ListBuilder.php`
- `src/Form/{Class}Form.php`
- `config/schema/my_module.schema.yml`
- `my_module.routing.yml`
- `my_module.links.menu.yml`
- `my_module.links.task.yml`
- `my_module.links.action.yml`
- `my_module.permissions.yml`

### `entity:bundle`

Generates a bundle class for an existing entity type.

```bash
ddev drush gen entity:bundle
```

**Key answers:** module, entity_type, class
**Output files:**
- `src/Entity/{Class}.php`

---

## Theme Generators

### `theme`

Generates a complete theme scaffold.

```bash
ddev drush gen theme
```

**Key answers:** name, machine_name, base_theme, description, package
**Output files:**
- `themes/custom/{machine_name}/{machine_name}.info.yml`
- `themes/custom/{machine_name}/{machine_name}.libraries.yml`
- `themes/custom/{machine_name}/{machine_name}.theme` (optional)
- `themes/custom/{machine_name}/templates/` directory
- `themes/custom/{machine_name}/css/` directory
- `themes/custom/{machine_name}/js/` directory

### `theme-file`

Generates a `.theme` file for an existing theme.

```bash
ddev drush gen theme-file --answers='{"theme":"my_theme"}'
```

**Output files:**
- `my_theme.theme`

---

## Test Generators

### `test`

Generates a PHPUnit test class (Unit, Kernel, Functional, or FunctionalJavascript).

```bash
ddev drush gen test
```

**Key answers:** module, class, type (unit/kernel/functional/functional-javascript)
**Output files (by type):**
- Unit: `tests/src/Unit/{Class}.php`
- Kernel: `tests/src/Kernel/{Class}.php`
- Functional: `tests/src/Functional/{Class}.php`
- FunctionalJavascript: `tests/src/FunctionalJavascript/{Class}.php`

---

## YAML File Generators

### `yml:module:services`

Generates a `services.yml` skeleton for a module.

```bash
ddev drush gen yml:module:services
```

**Key answers:** module
**Output files:**
- `my_module.services.yml`

### `yml:module:routing`

Generates a `routing.yml` skeleton for a module.

```bash
ddev drush gen yml:module:routing
```

**Key answers:** module
**Output files:**
- `my_module.routing.yml`

### `yml:module:permissions`

Generates a `permissions.yml` file for a module.

```bash
ddev drush gen yml:module:permissions
```

**Key answers:** module
**Output files:**
- `my_module.permissions.yml`

### `yml:module:links:menu`

Generates a `links.menu.yml` file for a module.

```bash
ddev drush gen yml:module:links:menu
```

**Key answers:** module
**Output files:**
- `my_module.links.menu.yml`

### `yml:module:links:task`

Generates a `links.task.yml` file for a module (local tasks / tabs).

```bash
ddev drush gen yml:module:links:task
```

**Key answers:** module
**Output files:**
- `my_module.links.task.yml`

### `yml:module:links:action`

Generates a `links.action.yml` file for a module (action links like "Add new").

```bash
ddev drush gen yml:module:links:action
```

**Key answers:** module
**Output files:**
- `my_module.links.action.yml`

### `yml:module:links:contextual`

Generates a `links.contextual.yml` file for a module.

```bash
ddev drush gen yml:module:links:contextual
```

**Key answers:** module
**Output files:**
- `my_module.links.contextual.yml`

---

## Miscellaneous Generators

### `breakpoints`

Generates a `breakpoints.yml` file for a theme.

```bash
ddev drush gen breakpoints
```

**Key answers:** theme
**Output files:**
- `my_theme.breakpoints.yml`

### `layout`

Generates a layout plugin definition.

```bash
ddev drush gen layout
```

**Key answers:** module, layout_name, label, category
**Output files:**
- `layouts/{layout_name}/{layout_name}.html.twig`
- `my_module.layouts.yml` (appended)

### `render-element`

Generates a render element plugin.

```bash
ddev drush gen render-element
```

**Key answers:** module, plugin_id, class
**Output files:**
- `src/Element/{Class}.php`

### `configuration`

Generates a default configuration file with schema.

```bash
ddev drush gen configuration
```

**Key answers:** module
**Output files:**
- `config/install/my_module.settings.yml`
- `config/schema/my_module.schema.yml`

### `phpstorm-metadata`

Generates PhpStorm metadata for service autocompletion.

```bash
ddev drush gen phpstorm-metadata
```

**Output files:**
- `.phpstorm.meta.php`

---

## Generator Combination Patterns

### Full Feature Module (manual sequence)

To scaffold a complete feature module with a settings form, a custom service, and a block plugin:

```bash
# 1. Create the module
ddev drush gen module --answers='{"name":"Feature Example","machine_name":"feature_example","description":"Example feature module.","package":"Custom","dependencies":"","install_file":false,"libraries_file":false,"permissions_file":true,"event_subscriber":false,"navigation":false}'

# 2. Add a config form
ddev drush gen form:config --answers='{"module":"feature_example","class":"SettingsForm","form_id":"feature_example_settings","route_name":"feature_example.settings","route_path":"/admin/config/feature-example","route_title":"Feature Example Settings","route_permission":"administer site configuration"}'

# 3. Add a service
ddev drush gen service --answers='{"module":"feature_example","service_name":"feature_example.processor","class":"Processor"}'

# 4. Add a block plugin
ddev drush gen plugin:block --answers='{"module":"feature_example","plugin_id":"feature_example_display","plugin_label":"Feature Display","category":"Custom","class":"FeatureDisplayBlock","configurable":false,"services":""}'

# 5. Clear cache
ddev drush cr
```

### Content Entity Module

```bash
# 1. Create the module
ddev drush gen module --answers='{"name":"Task Manager","machine_name":"task_manager","description":"Manages tasks.","package":"Custom","dependencies":"","install_file":true,"libraries_file":false,"permissions_file":false,"event_subscriber":false,"navigation":false}'

# 2. Generate the content entity
ddev drush gen entity:content --answers='{"module":"task_manager","entity_type_label":"Task","entity_type_id":"task","class":"Task","base_table":"task","admin_permission":"administer tasks"}'

# 3. Clear cache and install
ddev drush cr
ddev drush en task_manager
```
