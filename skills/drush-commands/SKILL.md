---
name: drush-commands
description: Use when running Drush CLI commands for Drupal site operations — cache clearing, configuration import/export, module management, database operations, user management, deployment, cron, watchdog, state, maintenance mode, locale, or any drush command that is not code generation.
version: 1.0.0
---

# Drush Commands Reference

Drush is the command-line shell for Drupal. All commands below use the `ddev drush` prefix (DDEV environment). Drop the `ddev` prefix if running Drush directly.

---

## Cache Management

```bash
# Clear ALL caches (most common command)
ddev drush cr
# Alias: ddev drush cache:rebuild

# Clear a specific cache bin
ddev drush cc bin         # Then select: render, page, dynamic_page_cache, discovery, etc.

# Clear render cache only
ddev drush ev '\Drupal::service("cache.render")->invalidateAll();'

# Clear Twig template cache
ddev drush ev '\Drupal::service("twig")->invalidate();'

# Clear plugin discovery caches (after adding a new plugin)
ddev drush cr

# Clear container cache (after changing services.yml)
ddev drush cr
```

**When to clear caches:**
- After changing `*.services.yml` → `ddev drush cr`
- After adding/changing plugins → `ddev drush cr`
- After changing `*.routing.yml` → `ddev drush cr`
- After changing `*.libraries.yml` → `ddev drush cr`
- After changing Twig templates → Usually auto-detected in dev, `ddev drush cr` if not
- After changing `.module` hooks → `ddev drush cr`

---

## Configuration Management

```bash
# Export all active config to the sync directory
ddev drush config:export -y
# Alias: ddev drush cex -y

# Import config from sync directory to active
ddev drush config:import -y
# Alias: ddev drush cim -y

# Show status — differences between active and sync
ddev drush config:status
# Alias: ddev drush cst

# Get a single config object (view current values)
ddev drush config:get system.site
# Alias: ddev drush cget system.site

# Get a specific config key
ddev drush config:get system.site name

# Set a config value
ddev drush config:set system.site name "New Site Name" -y
# Alias: ddev drush cset system.site name "New Site Name" -y

# Delete a config object
ddev drush config:delete mymodule.settings -y

# Partial import from a directory (not the sync dir)
ddev drush config:import --partial --source=/path/to/config/dir -y

# Import a single config file
ddev drush config:import --partial --source=./config/overrides -y

# List all config names matching a pattern
ddev drush config:list | grep mymodule

# Edit config interactively (opens in $EDITOR)
ddev drush config:edit system.site
```

---

## Module Management

```bash
# Enable a module
ddev drush en mymodule -y
# Alias: ddev drush pm:enable mymodule -y

# Enable multiple modules
ddev drush en mymodule1 mymodule2 mymodule3 -y

# Uninstall a module (removes config and schema)
ddev drush pm:uninstall mymodule -y
# Alias: ddev drush pmu mymodule -y

# List all modules with status
ddev drush pm:list
# Alias: ddev drush pml

# Filter installed modules
ddev drush pml --status=enabled
ddev drush pml --type=module --status=enabled

# Show info about a specific module
ddev drush pm:list --filter=mymodule

# Check for available security updates
ddev drush pm:security
```

---

## Theme Management

```bash
# Enable a theme
ddev drush theme:enable mytheme

# Set the default theme
ddev drush config:set system.theme default mytheme -y

# Set the admin theme
ddev drush config:set system.theme admin claro -y

# Uninstall a theme
ddev drush theme:uninstall mytheme
```

---

## Database Operations

```bash
# Open a database CLI (mysql/mariadb prompt)
ddev drush sql:cli
# Alias: ddev drush sqlc

# Run a SQL query
ddev drush sql:query "SELECT nid, title FROM node_field_data LIMIT 10"
# Alias: ddev drush sqlq "..."

# Dump the database to a file
ddev drush sql:dump > dump.sql
ddev drush sql:dump --gzip > dump.sql.gz
ddev drush sql:dump --result-file=/tmp/dump.sql

# Import a database dump
ddev drush sql:cli < dump.sql
# Or with DDEV:
ddev import-db --file=dump.sql.gz

# Drop all tables (dangerous!)
ddev drush sql:drop -y

# Create a database snapshot (DDEV)
ddev snapshot --name=before-update
ddev snapshot restore --latest
ddev snapshot restore --name=before-update

# Show database connection info
ddev drush sql:conf

# Sanitize the database (remove sensitive data for dev use)
ddev drush sql:sanitize -y
# Resets all passwords to "password", randomizes emails
```

---

## User Management

```bash
# Generate a one-time login link for admin (uid=1)
ddev drush uli
# Alias: ddev drush user:login

# Login link for a specific user
ddev drush uli --name=editor
ddev drush uli --uid=5

# Create a user
ddev drush user:create johndoe --mail="john@example.com" --password="secret"
# Alias: ddev drush ucrt johndoe ...

# Add a role to a user
ddev drush user:role:add administrator johndoe
# Alias: ddev drush urol administrator johndoe

# Remove a role from a user
ddev drush user:role:remove administrator johndoe

# Block/unblock a user
ddev drush user:block johndoe
ddev drush user:unblock johndoe

# Reset a user's password
ddev drush user:password johndoe "newpassword"
# Alias: ddev drush upwd johndoe "newpassword"

# Show user info
ddev drush user:information johndoe
# Alias: ddev drush uinf johndoe

# Cancel (delete) a user
ddev drush user:cancel johndoe -y

# List users with a specific role
ddev drush sqlq "SELECT u.uid, ufd.name FROM users u JOIN users_field_data ufd ON u.uid = ufd.uid JOIN user__roles ur ON u.uid = ur.entity_id WHERE ur.roles_target_id = 'administrator'"
```

---

## Deployment

```bash
# Full deployment sequence (recommended for CI/CD)
ddev drush deploy

# deploy runs these steps in order:
# 1. updatedb (run pending database updates)
# 2. config:import (import configuration)
# 3. cache:rebuild (clear caches)
# 4. deploy:hook (run deploy hooks)

# Run individual deployment steps:
ddev drush updatedb -y          # Run pending update hooks
# Alias: ddev drush updb -y

ddev drush config:import -y     # Import config changes
ddev drush cache:rebuild        # Clear caches

# Deploy hooks (hook_deploy_NAME in .deploy.php)
ddev drush deploy:hook -y

# Maintenance mode during deployment
ddev drush state:set system.maintenance_mode 1 -y
ddev drush deploy
ddev drush state:set system.maintenance_mode 0 -y
ddev drush cr
```

### Deployment Best Practice Script

```bash
#!/bin/bash
# deploy.sh — standard Drupal deployment
set -e

echo "Enabling maintenance mode..."
ddev drush state:set system.maintenance_mode 1 -y

echo "Running database updates..."
ddev drush updatedb -y

echo "Importing configuration..."
ddev drush config:import -y

echo "Running deploy hooks..."
ddev drush deploy:hook -y

echo "Rebuilding cache..."
ddev drush cache:rebuild

echo "Disabling maintenance mode..."
ddev drush state:set system.maintenance_mode 0 -y

echo "Deployment complete!"
ddev drush status
```

---

## Cron

```bash
# Run cron manually
ddev drush cron

# Show when cron last ran
ddev drush sget system.cron_last --format=string
# Convert timestamp: ddev drush ev 'echo date("Y-m-d H:i:s", \Drupal::state()->get("system.cron_last"));'
```

---

## Queue Management

```bash
# List all queues and their item counts
ddev drush queue:list

# Process a specific queue
ddev drush queue:run mymodule_queue

# Delete all items from a queue
ddev drush queue:delete mymodule_queue

# Process queues with a time limit (seconds)
ddev drush queue:run mymodule_queue --time-limit=60
```

---

## Watchdog / Logging

```bash
# Show recent log entries
ddev drush watchdog:show
# Alias: ddev drush ws

# Show last 50 entries
ddev drush ws --count=50

# Filter by severity
ddev drush ws --severity=error
ddev drush ws --severity=warning

# Filter by type (module name)
ddev drush ws --type=mymodule

# Show a specific log entry
ddev drush watchdog:show 12345

# Tail the log (watch in real-time)
ddev drush ws --tail

# Delete all log entries
ddev drush watchdog:delete all -y
```

---

## State API

```bash
# Get a state value
ddev drush state:get system.maintenance_mode
# Alias: ddev drush sget system.maintenance_mode

# Set a state value
ddev drush state:set system.maintenance_mode 1 -y
# Alias: ddev drush sset system.maintenance_mode 1 -y

# Delete a state value
ddev drush state:delete mymodule.last_sync -y
```

---

## Maintenance Mode

```bash
# Enable maintenance mode
ddev drush state:set system.maintenance_mode 1 -y
ddev drush cr

# Disable maintenance mode
ddev drush state:set system.maintenance_mode 0 -y
ddev drush cr
```

---

## Site Install

```bash
# Install Drupal with standard profile
ddev drush site:install standard -y
# Alias: ddev drush si standard -y

# Install with specific settings
ddev drush si standard \
  --account-name=admin \
  --account-pass=admin \
  --account-mail=admin@example.com \
  --site-name="My Site" \
  --site-mail=site@example.com \
  -y

# Install with a specific installation profile
ddev drush si minimal -y

# Install from existing config
ddev drush si --existing-config -y
```

---

## Entity Operations

```bash
# Delete all nodes of a type
ddev drush entity:delete node --bundle=article -y

# Delete specific entities by ID
ddev drush entity:delete node 1,2,3

# Save all entities of a type (triggers presave/save hooks — useful after field changes)
ddev drush entity:save node --bundle=article
```

---

## Locale / Translation

```bash
# Check for translation updates
ddev drush locale:check

# Import available translations
ddev drush locale:update

# Import a PO translation file
ddev drush locale:import es /path/to/es.po

# Export translations
ddev drush locale:export es > es.po
```

---

## PHP Evaluation

```bash
# Run arbitrary PHP
ddev drush ev 'echo \Drupal::VERSION;'
# Alias: ddev drush php:eval '...'

# Open an interactive PHP shell with Drupal bootstrapped
ddev drush php:cli
# Alias: ddev drush core:cli

# Run a PHP script file
ddev drush php:script path/to/script.php
# Alias: ddev drush scr path/to/script.php
```

### Useful One-Liners

```bash
# Get Drupal version
ddev drush ev 'echo \Drupal::VERSION;'

# Get site UUID
ddev drush ev 'echo \Drupal::config("system.site")->get("uuid");'

# Count nodes by type
ddev drush sqlq "SELECT type, COUNT(*) as count FROM node_field_data GROUP BY type"

# Clear a specific entity from cache
ddev drush ev '\Drupal::entityTypeManager()->getStorage("node")->resetCache([1]);'

# Check if a module is enabled
ddev drush ev 'echo \Drupal::moduleHandler()->moduleExists("mymodule") ? "yes" : "no";'

# Get a service and call a method
ddev drush ev 'echo \Drupal::service("entity_type.manager")->getDefinition("node")->getLabel();'
```

---

## Site Status & Information

```bash
# Full site status report
ddev drush status
# Alias: ddev drush st

# Show specific status field
ddev drush status --field=drupal-version
ddev drush status --field=db-name
ddev drush status --field=php-version

# Show the bootstrap status
ddev drush core:status

# Show the requirements report (same as /admin/reports/status)
ddev drush core:requirements
```

---

## Update Hooks

```bash
# Run pending database update hooks
ddev drush updatedb -y
# Alias: ddev drush updb -y

# Show pending updates without running them
ddev drush updatedb:status
# Alias: ddev drush updbst

# Run post-update hooks
# (these run as part of updatedb automatically)
```

---

## Views

```bash
# List all views and their status
ddev drush views:list

# Enable a view
ddev drush views:enable viewname

# Disable a view
ddev drush views:disable viewname

# Execute a view and show results
ddev drush views:execute viewname
ddev drush views:execute viewname --count=5
```

---

## Scaffolding with Drush

For code generation (creating modules, plugins, forms, services, etc.), see the **drush-generate** skill which covers `ddev drush gen` commands.

## Related Skills

- **drush-generate** — Code scaffolding with `ddev drush gen`
- **ddev-setup** — DDEV project setup and environment management
- **drupal-patterns** — Decision framework for choosing Drupal patterns

