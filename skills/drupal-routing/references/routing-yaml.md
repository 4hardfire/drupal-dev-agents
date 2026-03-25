# Routing YAML Reference

Complete reference for Drupal `*.routing.yml` files.

## File Location

Place the file in the module root as `<module_name>.routing.yml`. Drupal discovers
it automatically when caches are rebuilt.

## Route Structure

```yaml
<route_name>:
  path: '<path pattern>'
  defaults:
    <default key>: <value>
  requirements:
    <requirement key>: <value>
  options:
    <option key>: <value>
  methods: [GET, POST]
```

Every route must have at minimum `path` and one handler in `defaults` (such as
`_controller` or `_form`), plus at least one access requirement.

## Path Patterns

Paths begin with `/` and may contain parameter placeholders in curly braces.

```yaml
path: '/reports/{year}/{month}'
```

Parameters are passed to the controller method by name:

```php
public function view(string $year, string $month): array { ... }
```

### Optional Parameters

Provide a default value to make a parameter optional:

```yaml
my_module.archive:
  path: '/archive/{year}/{month}'
  defaults:
    _controller: '\Drupal\my_module\Controller\ArchiveController::list'
    _title: 'Archive'
    year: NULL
    month: NULL
  requirements:
    _permission: 'access content'
    year: \d{4}
    month: \d{1,2}
```

If `year` or `month` is omitted from the URL, the controller receives `NULL`.

## Defaults

### Handler Defaults

Only one handler should be specified per route.

| Key | Description |
|---|---|
| `_controller` | `\Fully\Qualified\Class::method` returning a render array or Response. |
| `_form` | `\Fully\Qualified\FormClass` implementing `FormInterface`. |
| `_entity_form` | `<entity_type_id>.<form_mode>`, e.g., `node.edit`. |
| `_entity_list` | Entity type ID; uses the entity's list builder. |

### Other Defaults

| Key | Description |
|---|---|
| `_title` | Static page title. |
| `_title_callback` | `\Class::method` returning a translated string. |
| `_title_arguments` | Array of placeholders for `_title`. |
| `_title_context` | Translation context for `_title`. |

## Requirements

Requirements constrain when a route matches.

### Access Requirements

At least one access requirement is mandatory.

| Key | Value | Description |
|---|---|---|
| `_permission` | `'administer nodes'` | User must have this permission. |
| `_role` | `'administrator'` | User must have this role. Multiple roles: `'administrator+editor'` (AND) or `'administrator,editor'` (OR). |
| `_access` | `'TRUE'` | Unconditional access. Use sparingly. |
| `_custom_access` | `'\Class::method'` | Method receives route parameters and returns `AccessResultInterface`. |
| `_entity_access` | `'node.view'` | Checks entity access for the given operation on the parameter. |

### Parameter Constraints

Constrain parameter values with regex patterns:

```yaml
requirements:
  node: \d+
  type: '[a-z_]+'
```

### Request Constraints

| Key | Value | Description |
|---|---|---|
| `_format` | `html` or `json` | Restrict by response format. |
| `_method` | `GET\|POST` | Restrict by HTTP method. Pipe-separated. |
| `_csrf_token` | `'TRUE'` | Require a valid CSRF token query parameter. |
| `_content_type_format` | `json` | Require specific Content-Type on the request body. |

### CSRF Token Usage

For routes that modify data via GET-style links (e.g., "confirm" links):

```yaml
my_module.action:
  path: '/action/execute/{node}'
  defaults:
    _controller: '\Drupal\my_module\Controller\ActionController::execute'
  requirements:
    _csrf_token: 'TRUE'
    _permission: 'administer nodes'
    node: \d+
```

Generate the token in code:

```php
use Drupal\Core\Url;

$url = Url::fromRoute('my_module.action', ['node' => $node->id()]);
// The token query parameter is added automatically when _csrf_token is TRUE.
```

## Options

### `_admin_route`

Use the administration theme for this route:

```yaml
options:
  _admin_route: TRUE
```

### `no_cache`

Disable the page cache for this route:

```yaml
options:
  no_cache: TRUE
```

### `parameters`

Specify parameter types for upcasting and validation.

```yaml
options:
  parameters:
    node:
      type: entity:node
    taxonomy_term:
      type: entity:taxonomy_term
```

#### Revision Parameters

Load a specific revision:

```yaml
my_module.node_revision:
  path: '/node/{node}/revision/{node_revision}'
  defaults:
    _controller: '\Drupal\my_module\Controller\RevisionController::view'
    _title: 'Revision'
  requirements:
    _permission: 'view all revisions'
    node: \d+
    node_revision: \d+
  options:
    parameters:
      node:
        type: entity:node
      node_revision:
        type: entity_revision:node
```

### `_maintenance_access`

Allow access to this route even in maintenance mode:

```yaml
options:
  _maintenance_access: TRUE
```

### `_theme`

Force a specific theme:

```yaml
options:
  _theme: 'claro'
```

## Wildcard / Catch-All Routes

Use the Symfony `{path}` parameter with a regex that matches slashes:

```yaml
my_module.catchall:
  path: '/docs/{path}'
  defaults:
    _controller: '\Drupal\my_module\Controller\DocsController::render'
    _title: 'Documentation'
    path: ''
  requirements:
    _permission: 'access content'
    path: .+
```

The controller receives the full remaining path as a string.

## Multiple Permissions

Combine permissions with `+` (AND) or `,` (OR):

```yaml
# User must have BOTH permissions:
requirements:
  _permission: 'access content+view own unpublished content'

# User must have at least ONE permission:
requirements:
  _permission: 'access content,bypass node access'
```

## Routes for REST Resources

REST resources typically have their routes generated automatically, but manual
definitions look like this:

```yaml
my_module.api.items:
  path: '/api/items'
  defaults:
    _controller: '\Drupal\my_module\Controller\ApiController::list'
  requirements:
    _permission: 'access content'
    _format: json
  methods: [GET]

my_module.api.item:
  path: '/api/items/{item}'
  defaults:
    _controller: '\Drupal\my_module\Controller\ApiController::get'
  requirements:
    _permission: 'access content'
    _format: json
    item: \d+
  methods: [GET]

my_module.api.item_create:
  path: '/api/items'
  defaults:
    _controller: '\Drupal\my_module\Controller\ApiController::create'
  requirements:
    _permission: 'create items'
    _content_type_format: json
  methods: [POST]
```

## Route Name Conventions

Follow Drupal's naming conventions:

| Pattern | Example |
|---|---|
| `<module>.overview` | `my_module.overview` |
| `<module>.<entity>_list` | `my_module.item_list` |
| `<module>.<entity>_add` | `my_module.item_add` |
| `<module>.<entity>_edit` | `my_module.item_edit` |
| `<module>.<entity>_delete` | `my_module.item_delete` |
| `<module>.settings` | `my_module.settings` |
| `<module>.admin` | `my_module.admin` |
| `entity.<entity_type>.canonical` | `entity.node.canonical` |
| `entity.<entity_type>.edit_form` | `entity.node.edit_form` |

## Debugging Routes

Useful Drush commands:

```bash
# List all routes
ddev drush route:list

# Filter routes by path
ddev drush route:list --path=/admin

# Filter routes by name
ddev drush route:list --name=my_module

# Rebuild the router
ddev drush cache:rebuild
```

## Common Pitfalls

1. **Missing access requirement** -- every route must have at least one access
   check or Drupal will deny access by default and log a warning.
2. **Forgetting to rebuild caches** -- routing YAML is compiled; run
   `ddev drush cr` after changes.
3. **Parameter name mismatch** -- the parameter name in the path must match the
   controller method argument name and the `options.parameters` key.
4. **Incorrect regex in requirements** -- requirements values are regex patterns,
   not quoted strings. Write `\d+` not `'/\d+/'`.
