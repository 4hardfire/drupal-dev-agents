# Drush Commands Complete Catalog

Comprehensive catalog of all Drush commands available in Drupal 10/11+. Organized by namespace. All commands use the `ddev drush` prefix.

---

## cache (Cache management)

| Command | Alias | Description |
|---------|-------|-------------|
| `cache:rebuild` | `cr` | Rebuild all caches. This is the most common Drush command. |
| `cache:get` | `cg` | Fetch a cached object and display it. |
| `cache:set` | `cs` | Cache an object expressed in JSON or var_export format. |
| `cache:clear` | `cc` | Clear a specific cache bin. Prompts for which bin. |
| `cache:tag:invalidate` | | Invalidate by cache tag. |

### Cache Bins Reference

| Bin | What It Caches |
|-----|---------------|
| `bootstrap` | Bootstrap-level data (module list, active theme) |
| `config` | Configuration objects |
| `data` | General data cache |
| `default` | Default cache bin |
| `discovery` | Plugin and service discovery |
| `dynamic_page_cache` | Dynamic Page Cache (per-user page variations) |
| `entity` | Entity data |
| `menu` | Menu trees and links |
| `page` | Full page cache (Internal Page Cache module) |
| `render` | Render arrays and rendered HTML |
| `static` | Static cache (in-process, not persisted) |
| `toolbar` | Toolbar rendering |

---

## config (Configuration management)

| Command | Alias | Description |
|---------|-------|-------------|
| `config:export` | `cex` | Export configuration to the sync directory |
| `config:import` | `cim` | Import configuration from the sync directory |
| `config:status` | `cst` | Show differences between active and sync |
| `config:get` | `cget` | Display a config object or a specific key |
| `config:set` | `cset` | Set a config value |
| `config:delete` | `cdel` | Delete a config object |
| `config:edit` | `cedit` | Open a config object in $EDITOR |
| `config:list` | `cli` | List all config names |
| `config:pull` | | Export and transfer config from one env to another |

### Config Import Options

```bash
# Standard import
ddev drush cim -y

# Partial import (from a non-sync directory)
ddev drush cim --partial --source=/path/to/dir -y

# Import specific config type only
ddev drush cim --diff     # Show diff before importing

# Dry run (preview what would change)
ddev drush cst
```

### Config Export Options

```bash
# Standard export
ddev drush cex -y

# Export to a specific directory (not sync)
ddev drush cex --destination=/tmp/config -y
```

---

## core (Core commands)

| Command | Alias | Description |
|---------|-------|-------------|
| `core:cron` | `cron` | Run all cron hooks |
| `core:requirements` | `rq` | Information about things that may be wrong (like admin/reports/status) |
| `core:status` | `st` | Show Drupal site status overview |
| `core:edit` | `conf` | Edit drushrc, site alias, and Drupal settings.php files |
| `core:topic` | `topic` | Read detailed documentation on a given topic |
| `core:rsync` | `rsync` | Rsync Drupal code or files to/from another server |

---

## deploy (Deployment)

| Command | Alias | Description |
|---------|-------|-------------|
| `deploy` | | Run all deployment steps (updatedb, config:import, cache:rebuild, deploy:hook) |
| `deploy:hook` | | Run deploy hook implementations (hook_deploy_NAME) |
| `deploy:mark-complete` | | Mark all deploy hooks as having been run |

### Deploy Hook File

Create a `{module}.deploy.php` file in your module root:

```php
<?php

/**
 * @file
 * Deploy hooks for mymodule.
 */

/**
 * Migrate legacy data after deployment.
 */
function mymodule_deploy_migrate_legacy_data(): string {
  // This runs after updatedb and config:import.
  $count = 0;
  // ... migration logic ...
  return "Migrated $count records.";
}

/**
 * Update taxonomy terms after config import.
 */
function mymodule_deploy_update_taxonomy(): string {
  // Deploy hooks run in order of their function names.
  return "Updated taxonomy terms.";
}
```

Deploy hooks vs update hooks:
- **Update hooks** (`hook_update_N`): Run during `drush updatedb`, for schema changes and data migrations tied to code changes
- **Post-update hooks** (`hook_post_update_NAME`): Run after all update hooks, for entity/config updates that depend on schema being current
- **Deploy hooks** (`hook_deploy_NAME`): Run during `drush deploy:hook`, for data operations that should happen after config import (e.g., creating content, setting state)

---

## entity (Entity operations)

| Command | Alias | Description |
|---------|-------|-------------|
| `entity:delete` | `edel` | Delete content entities |
| `entity:save` | `esav` | Re-save entities (triggers hooks) |
| `entity:updates` | `entup` | Apply pending entity schema updates |

```bash
# Delete all articles
ddev drush entity:delete node --bundle=article -y

# Delete specific nodes
ddev drush entity:delete node 1,2,3 -y

# Re-save all articles (triggers presave/save hooks)
ddev drush entity:save node --bundle=article

# Apply pending entity definition updates
ddev drush entity:updates -y
```

---

## field (Field operations)

| Command | Alias | Description |
|---------|-------|-------------|
| `field:info` | | Show information about a field |
| `field:create` | `fc` | Create a field (interactive) |
| `field:delete` | `fd` | Delete a field |
| `field:base-info` | | List all base fields for an entity type |
| `field:base-override-create` | | Create a base field override |

```bash
# List all fields on article nodes
ddev drush field:info --entity-type=node --bundle=article

# List all base fields for nodes
ddev drush field:base-info node
```

---

## locale (Translation)

| Command | Alias | Description |
|---------|-------|-------------|
| `locale:check` | | Check for pending translation updates |
| `locale:update` | | Import pending translation updates |
| `locale:import` | | Import a PO file |
| `locale:export` | | Export translations to a PO file |
| `locale:import-all` | | Import all translations from a directory |

```bash
# Full translation workflow
ddev drush locale:check            # Check for updates
ddev drush locale:update           # Download and import updates

# Import a custom translation
ddev drush locale:import es /path/to/es.po --type=customized
ddev drush locale:import fr /path/to/fr.po --type=not-customized

# Export translations for a language
ddev drush locale:export es > translations/es.po
```

---

## php (PHP operations)

| Command | Alias | Description |
|---------|-------|-------------|
| `php:eval` | `ev` | Evaluate arbitrary PHP with Drupal bootstrapped |
| `php:script` | `scr` | Run a PHP script with Drupal bootstrapped |
| `php:cli` | `core:cli` | Open an interactive PHP shell |

### Useful Evaluation Scripts

```bash
# Check Drupal version
ddev drush ev 'echo \Drupal::VERSION;'

# Get site name
ddev drush ev 'echo \Drupal::config("system.site")->get("name");'

# Count entities
ddev drush ev 'echo \Drupal::entityQuery("node")->accessCheck(FALSE)->count()->execute();'

# List enabled modules
ddev drush ev 'print_r(array_keys(\Drupal::moduleHandler()->getModuleList()));'

# Check a service exists
ddev drush ev 'var_dump(\Drupal::hasService("mymodule.my_service"));'

# Run a service method
ddev drush ev '\Drupal::service("mymodule.processor")->processAll();'

# Load and inspect an entity
ddev drush ev '$n = \Drupal\node\Entity\Node::load(1); echo $n->label() . " (" . $n->bundle() . ")";'

# Rebuild entity definitions
ddev drush ev '\Drupal::entityDefinitionUpdateManager()->applyUpdates();'

# Rebuild router
ddev drush ev '\Drupal::service("router.builder")->rebuild();'

# Rebuild theme registry
ddev drush ev '\Drupal::service("theme.registry")->reset();'

# Get current user permissions
ddev drush ev 'print_r(\Drupal\user\Entity\User::load(1)->getRoles());'
```

---

## pm (Package manager)

| Command | Alias | Description |
|---------|-------|-------------|
| `pm:enable` | `en` | Enable one or more modules |
| `pm:uninstall` | `pmu` | Uninstall one or more modules |
| `pm:list` | `pml` | Show a list of available extensions |
| `pm:security` | `sec` | Check for security updates |

```bash
# Enable modules
ddev drush en admin_toolbar devel -y

# Uninstall modules
ddev drush pmu devel -y

# List all enabled modules
ddev drush pml --status=enabled --type=module --no-core

# List enabled contrib modules
ddev drush pml --status=enabled --type=module --no-core

# Check security advisories
ddev drush pm:security
```

---

## queue (Queue management)

| Command | Alias | Description |
|---------|-------|-------------|
| `queue:list` | | List all queues with item counts |
| `queue:run` | | Run a specific queue worker |
| `queue:delete` | | Delete all items from a queue |

```bash
# List all queues
ddev drush queue:list

# Process items in a queue
ddev drush queue:run mymodule_queue

# Process with time limit
ddev drush queue:run mymodule_queue --time-limit=120

# Process with item limit
ddev drush queue:run mymodule_queue --items-limit=50

# Clear a queue
ddev drush queue:delete mymodule_queue
```

---

## role (Role management)

| Command | Alias | Description |
|---------|-------|-------------|
| `role:create` | `rcrt` | Create a role |
| `role:delete` | `rdel` | Delete a role |
| `role:list` | `rls` | List all roles |
| `role:perm:add` | `rap` | Grant permission(s) to a role |
| `role:perm:remove` | `rmp` | Remove permission(s) from a role |
| `role:perm:list` | `rp` | List permissions for a role |

```bash
# Create a role
ddev drush role:create editor "Editor"

# Grant permissions
ddev drush role:perm:add editor "create article content,edit any article content,delete any article content"

# Remove permissions
ddev drush role:perm:remove editor "delete any article content"

# List permissions for a role
ddev drush role:perm:list editor

# List all roles
ddev drush role:list
```

---

## sql (Database operations)

| Command | Alias | Description |
|---------|-------|-------------|
| `sql:cli` | `sqlc` | Open a SQL CLI (mysql prompt) |
| `sql:conf` | | Show database connection details |
| `sql:connect` | | Show the db connection string |
| `sql:create` | | Create the database |
| `sql:drop` | | Drop all tables in the database |
| `sql:dump` | | Export the database |
| `sql:query` | `sqlq` | Execute a SQL query |
| `sql:sanitize` | `sqlsan` | Sanitize database (remove sensitive data) |
| `sql:sync` | | Sync the database between environments |

```bash
# Dump database
ddev drush sql:dump --gzip > db-backup.sql.gz
ddev drush sql:dump --result-file=/tmp/backup.sql

# Dump specific tables only
ddev drush sql:dump --tables-list=users_field_data,node_field_data > partial.sql

# Dump excluding cache tables
ddev drush sql:dump --structure-tables-key=common > dump-no-cache.sql

# Sanitize for development
ddev drush sql:sanitize -y
# Options:
#   --sanitize-password=test     Set all passwords to "test"
#   --sanitize-email=user+%uid@example.com
#   --allowlist-fields=field_name   Skip sanitizing specific fields
```

---

## state (State API)

| Command | Alias | Description |
|---------|-------|-------------|
| `state:get` | `sget` | Get a state value |
| `state:set` | `sset` | Set a state value |
| `state:delete` | `sdel` | Delete a state value |

```bash
# Maintenance mode
ddev drush sset system.maintenance_mode 1 -y
ddev drush sset system.maintenance_mode 0 -y

# Custom state values
ddev drush sset mymodule.last_sync $(date +%s)
ddev drush sget mymodule.last_sync
ddev drush sdel mymodule.last_sync
```

---

## theme (Theme management)

| Command | Alias | Description |
|---------|-------|-------------|
| `theme:enable` | `then` | Enable a theme |
| `theme:uninstall` | `thun` | Uninstall a theme |

---

## updatedb (Database updates)

| Command | Alias | Description |
|---------|-------|-------------|
| `updatedb` | `updb` | Run pending database update hooks |
| `updatedb:status` | `updbst` | List pending database updates without running them |

```bash
# Check for pending updates
ddev drush updbst

# Run pending updates
ddev drush updb -y

# Run with entity updates included
ddev drush updb --entity-updates -y
```

---

## user (User management)

| Command | Alias | Description |
|---------|-------|-------------|
| `user:login` | `uli` | Generate a one-time login URL |
| `user:create` | `ucrt` | Create a user account |
| `user:cancel` | `ucan` | Cancel a user account |
| `user:password` | `upwd` | Set a user's password |
| `user:block` | `ublk` | Block a user |
| `user:unblock` | `uublk` | Unblock a user |
| `user:role:add` | `urol` | Add a role to a user |
| `user:role:remove` | `urrol` | Remove a role from a user |
| `user:information` | `uinf` | Show user information |

---

## views (Views management)

| Command | Alias | Description |
|---------|-------|-------------|
| `views:list` | `vl` | List all views |
| `views:execute` | `vex` | Execute a view and show results |
| `views:enable` | `ven` | Enable a view |
| `views:disable` | `vdis` | Disable a view |
| `views:dev` | | Enable views development mode |
| `views:analyze` | | Get a report about all views |

---

## watchdog (Logging)

| Command | Alias | Description |
|---------|-------|-------------|
| `watchdog:show` | `ws` | Show watchdog log entries |
| `watchdog:list` | `wd-list` | Interactively filter watchdog messages |
| `watchdog:delete` | `wd-del` | Delete watchdog log entries |
| `watchdog:tail` | `wt` | Tail the watchdog log (real-time) |

```bash
# Show recent errors
ddev drush ws --severity=error --count=20

# Tail the log (like tail -f)
ddev drush ws --tail

# Filter by module
ddev drush ws --type=mymodule --count=50

# Delete old entries
ddev drush watchdog:delete all -y
```

---

## Global Options

These options work with any Drush command:

| Option | Description |
|--------|-------------|
| `-y` / `--yes` | Auto-accept confirmations |
| `-v` / `--verbose` | Verbose output |
| `-d` / `--debug` | Debug output (very verbose) |
| `--uri=URL` | Set the Drupal URI |
| `--root=PATH` | Set the Drupal root |
| `--format=json` | Output in JSON format |
| `--format=yaml` | Output in YAML format |
| `--format=table` | Output as a table |
| `--format=csv` | Output as CSV |
| `--field=name` | Output only a specific field |
| `--filter=text` | Filter output rows |
| `--pipe` | Machine-readable output |

```bash
# Get output as JSON
ddev drush status --format=json

# Get a specific field
ddev drush status --field=drupal-version

# Pipe-friendly output
ddev drush pml --status=enabled --pipe
```
