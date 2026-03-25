---
name: drush-gen
description: Guided Drush code generation — knows all generators and helps with options. Usage: /drush-gen <generator_type>
argument-hint: <generator_type>
allowed-tools: [Bash, Read, Write, Edit]
---

# Drush Generate Command

Help the user generate Drupal code scaffolding using `ddev drush gen`.

## Arguments

The user provided: $ARGUMENTS

This should be a generator type (e.g., "plugin:block", "form:config", "controller", "entity:content") or "list" to show all generators.

## Workflow

### Step 1: Validate Generator

If `$ARGUMENTS` is empty or "list", show available generators organized by category:

```bash
ddev drush gen --list
```

Present them in a formatted table:

**Module Generators:**
| Generator | Description |
|-----------|-------------|
| `module` | Full module scaffold |
| `module-file` | .module file |

**Plugin Generators:**
| Generator | Description |
|-----------|-------------|
| `plugin:block` | Block plugin |
| `plugin:field:type` | Field type plugin |
| `plugin:field:widget` | Field widget plugin |
| `plugin:field:formatter` | Field formatter plugin |
| `plugin:queue-worker` | Queue worker plugin |
| `plugin:action` | Action plugin |
| `plugin:condition` | Condition plugin |
| `plugin:filter` | Text filter plugin |
| `plugin:menu-link` | Menu link plugin |
| `plugin:rest-resource` | REST resource plugin |
| `plugin:entity-reference-selection` | Entity reference selection |

**Form Generators:**
| Generator | Description |
|-----------|-------------|
| `form:simple` | FormBase form |
| `form:config` | ConfigFormBase form |
| `form:confirm` | ConfirmFormBase form |

**Entity Generators:**
| Generator | Description |
|-----------|-------------|
| `entity:content` | Content entity type |
| `entity:config` | Config entity type |
| `entity:bundle-class` | Bundle class for entity |

**Other Generators:**
| Generator | Description |
|-----------|-------------|
| `controller` | Controller + route |
| `service` | Service class + YAML |
| `event-subscriber` | Event subscriber |
| `hook` | Hook implementation |
| `middleware` | HTTP middleware |
| `theme` | Theme scaffold |
| `test` | PHPUnit test |
| `phpstorm-metadata` | PHPStorm metadata |

**YAML Generators:**
| Generator | Description |
|-----------|-------------|
| `yml:module:services` | services.yml |
| `yml:module:routing` | routing.yml |
| `yml:module:permissions` | permissions.yml |
| `yml:module:links:menu` | links.menu.yml |
| `yml:module:links:task` | links.task.yml |
| `yml:module:links:action` | links.action.yml |
| `yml:module:libraries` | libraries.yml |
| `yml:module:info` | info.yml |

Then ask the user which generator they want to use.

### Step 2: Determine Module Context

Find existing custom modules:

```bash
find web/modules/custom -name "*.info.yml" -maxdepth 2 2>/dev/null
```

- If one custom module exists, suggest it as the target
- If multiple exist, ask which one
- If none exist, suggest creating a new module first with `ddev drush gen module`

### Step 3: Gather Required Information

Based on the generator type, ask for necessary inputs. Common parameters:

**All generators:**
- Module name (machine name)
- Module path (usually auto-detected)

**plugin:block:**
- Plugin label (human name)
- Plugin ID (machine name)
- Plugin category
- Whether to create a config form
- Whether to inject dependencies

**form:config / form:simple:**
- Form class name
- Route path (for config form: `/admin/config/{category}/{name}`)
- Permission for route access

**controller:**
- Controller class name
- Route path
- Route title
- Permission

**entity:content:**
- Entity type label
- Entity type ID
- Whether to add base fields
- Whether to add admin UI (routes, forms, list builder)
- Whether to add Views integration
- Whether to support revisions
- Whether to support translations

**entity:config:**
- Entity type label
- Entity type ID
- Admin path prefix

**service:**
- Service name (class name)
- Service ID (module.service_name)

**event-subscriber:**
- Class name
- Events to subscribe to

**hook:**
- Hook name (e.g., form_alter, entity_presave)

### Step 4: Run Generator

Execute the Drush generate command. For automation, use the `--answers` flag:

```bash
ddev drush gen $ARGUMENTS --answers='{"name":"My Module","machine_name":"mymodule","class":"MyClassName"}'
```

If interactive mode is needed (complex generators), run:

```bash
ddev drush gen $ARGUMENTS
```

### Step 5: Post-Generation

After generation:

1. **Show generated files:**
   ```bash
   git status --short
   ```

2. **Display key file contents** — read and show the main generated files

3. **Clear caches:**
   ```bash
   ddev drush cr
   ```

4. **Provide context-specific tips:**

   For **plugins**:
   - Remind about cache metadata (`getCacheTags`, `getCacheContexts`, `getCacheMaxAge`)
   - Remind about access control if applicable

   For **forms**:
   - Remind about validation in `validateForm()`
   - For config forms, check that config schema exists

   For **entities**:
   - Remind about permissions in `.permissions.yml`
   - Remind about Views integration (`views_data` handler)
   - Suggest adding base fields in `baseFieldDefinitions()`

   For **services**:
   - Show how to inject the service in a controller or form
   - Show the service YAML entry

   For **event subscribers**:
   - Show the service YAML with `event_subscriber` tag
   - List common events they might want to subscribe to

5. **Suggest next steps** based on what was generated
