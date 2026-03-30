---
name: ddev-setup
description: Use when setting up a DDEV environment for Drupal, configuring DDEV services, enabling xdebug, adding DDEV add-ons, or managing DDEV project configuration.
version: 1.0.0
---

# DDEV Setup for Drupal

## Overview

DDEV is a Docker-based local development tool purpose-built for PHP projects including Drupal. It provides a consistent, reproducible development environment with zero-configuration defaults for Drupal, while remaining fully customizable for complex project needs. DDEV handles web server, database, PHP runtime, and additional services like Redis or Solr through a simple CLI interface.

This skill covers project initialization, configuration, day-to-day commands, database management, debugging, and integration with Drush and Composer inside the DDEV container environment.

## Prerequisites

Before using DDEV for Drupal development, ensure the following are installed on the host machine:

- **Docker Desktop** (macOS/Windows) or **Docker Engine** (Linux) -- DDEV requires a Docker provider
- **DDEV v1.23+** -- install via `brew install ddev/ddev/ddev` on macOS or see https://ddev.readthedocs.io/en/stable/users/install/
- **mkcert** -- typically installed automatically with DDEV for local HTTPS certificates

Verify the installation:

```bash
ddev version
docker version
```

## New Drupal Project Setup

Follow these steps in order to create a brand-new Drupal project from scratch.

### Step 1: Create the project directory

```bash
mkdir my-project && cd my-project
```

### Step 2: Configure DDEV for Drupal

```bash
ddev config --project-type=drupal --php-version=8.3 --docroot=web
```

This creates the `.ddev/` directory with a `config.yaml` tailored for Drupal. The `--docroot=web` flag matches the Drupal recommended project structure where the web root is the `web/` subdirectory.

### Step 3: Start the DDEV environment

```bash
ddev start
```

This pulls and starts all required Docker containers (web server, database, etc.), configures networking, and sets up HTTPS via mkcert.

### Step 4: Create the Drupal project via Composer

```bash
ddev composer create drupal/recommended-project
```

This downloads Drupal core and its dependencies into the project directory using the `drupal/recommended-project` Composer template. DDEV runs Composer inside the container, so no local PHP is needed.

### Step 5: Install Drupal

```bash
ddev drush site:install --account-name=admin --account-pass=admin -y
```

This runs the Drupal installation profile, creates the database tables, and sets up an admin account with username `admin` and password `admin`.

### Step 6: Get a login link

```bash
ddev drush uli
```

This generates a one-time login URL. DDEV automatically rewrites it to use the correct local HTTPS hostname.

### Step 7: Launch the site in a browser

```bash
ddev launch
```

Opens the project URL in the default browser.

## Existing Drupal Project Setup

When cloning an existing Drupal project that does not already have DDEV configuration:

```bash
git clone <repo-url> my-project
cd my-project
ddev config --project-type=drupal --php-version=8.3 --docroot=web
ddev start
ddev composer install
```

If the project already has a `.ddev/` directory committed to the repository:

```bash
git clone <repo-url> my-project
cd my-project
ddev start
ddev composer install
```

### Importing an existing database

```bash
ddev import-db --file=path/to/database-dump.sql.gz
```

DDEV supports `.sql`, `.sql.gz`, `.sql.bz2`, `.sql.xz`, `.mysql`, `.mysql.gz`, and `.tar.gz` (containing a `.sql` file) formats.

After importing, run pending updates:

```bash
ddev drush updatedb -y
ddev drush cache:rebuild
```

## Key .ddev/config.yaml Settings

The `.ddev/config.yaml` file controls all project-level DDEV configuration. Key settings for Drupal:

```yaml
name: my-project
type: drupal
docroot: web
php_version: "8.3"
webserver_type: nginx-fpm
database:
  type: mariadb
  version: "10.11"
nodejs_version: "20"
additional_hostnames:
  - my-project-alt
composer_version: "2"
disable_settings_management: false
```

- **php_version**: Supported values include "8.1", "8.2", "8.3", "8.4"
- **webserver_type**: Options are "nginx-fpm" (default) or "apache-fpm"
- **database**: Type can be "mariadb", "mysql", or "postgres" with corresponding version
- **nodejs_version**: Set to "18", "20", "22", etc. for front-end tooling
- **disable_settings_management**: Set to `true` if you manage settings.ddev.php manually

For a complete reference of all config.yaml options, see `references/ddev-config.md`.

## Common DDEV Commands

### Project lifecycle

| Command | Description |
|---|---|
| `ddev start` | Start the project containers |
| `ddev stop` | Stop the project containers |
| `ddev restart` | Restart the project containers |
| `ddev delete -O` | Remove the project containers and volumes (keeps files) |
| `ddev poweroff` | Stop all DDEV projects and the router |

### Interacting with the container

| Command | Description |
|---|---|
| `ddev ssh` | Open an interactive shell inside the web container |
| `ddev exec <command>` | Run a single command inside the web container |
| `ddev logs` | Show web container logs |
| `ddev logs -s db` | Show database container logs |
| `ddev describe` | Show project details: URL, ports, database info, services |

### Project information

```bash
ddev describe          # Detailed project info
ddev list              # List all DDEV projects on the system
ddev status            # Quick status of current project
```

## Database Operations

### Import and export

```bash
ddev import-db --file=dump.sql.gz          # Import a database dump
ddev export-db --file=backup.sql.gz        # Export the current database
ddev export-db --gzip --file=backup.sql.gz # Export compressed
```

### Snapshots

Snapshots are fast, filesystem-level database backups (much faster than SQL dumps):

```bash
ddev snapshot                              # Create a snapshot
ddev snapshot --name=before-update         # Create a named snapshot
ddev snapshot restore --latest             # Restore the most recent snapshot
ddev snapshot restore before-update        # Restore a named snapshot
ddev snapshot --list                       # List all snapshots
ddev snapshot --cleanup                    # Delete all snapshots
```

### Direct database access

```bash
ddev mysql                                 # Open a MySQL/MariaDB CLI
ddev psql                                  # Open a PostgreSQL CLI (if using postgres)
```

## Xdebug

DDEV includes Xdebug but keeps it disabled by default for performance.

### Toggle Xdebug

```bash
ddev xdebug on         # Enable Xdebug step debugging
ddev xdebug off        # Disable Xdebug
ddev xdebug status     # Check current Xdebug state
```

### IDE configuration tips

- **PHPStorm**: Set the path mapping from the project root to `/var/www/html` in the PHP server configuration. PHPStorm typically auto-detects DDEV Xdebug connections.
- **VS Code**: Install the PHP Debug extension. Create a `.vscode/launch.json` with:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Listen for Xdebug",
      "type": "php",
      "request": "launch",
      "port": 9003,
      "pathMappings": {
        "/var/www/html": "${workspaceFolder}"
      }
    }
  ]
}
```

### Xdebug profiling

```bash
ddev xdebug on --mode=profile
```

Profile output goes to `/tmp/` inside the container. Retrieve files with `ddev exec ls /tmp/cachegrind*`.

## Drush in DDEV

All Drush commands must be prefixed with `ddev drush` to run inside the container:

```bash
ddev drush status                          # Drupal status report
ddev drush cache:rebuild                   # Rebuild caches (cr)
ddev drush updatedb -y                     # Run pending database updates
ddev drush config:export -y                # Export configuration
ddev drush config:import -y                # Import configuration
ddev drush uli                             # Generate one-time login link
ddev drush user:password admin "newpass"   # Reset a user password
ddev drush pm:list --type=module           # List all modules
ddev drush pm:enable module_name -y        # Enable a module
ddev drush pm:uninstall module_name -y     # Uninstall a module
ddev drush watchdog:show                   # Show recent log entries
```

## Composer in DDEV

All Composer commands must be prefixed with `ddev composer` to run inside the container:

```bash
ddev composer require drupal/module_name          # Add a module
ddev composer require --dev drupal/devel           # Add a dev dependency
ddev composer install                              # Install from lock file
ddev composer update drupal/core-* -W              # Update Drupal core
ddev composer update drupal/module_name            # Update a single module
ddev composer outdated --direct                    # Check for outdated deps
ddev composer why drupal/some_package              # Show why a package is installed
```

## Troubleshooting

Common issues and solutions:

| Problem | Solution |
|---|---|
| Port conflict on startup | `ddev poweroff` then `ddev start`, or change router ports in `~/.ddev/global_config.yaml` |
| Database connection refused | `ddev restart` or check `ddev describe` for DB credentials |
| Composer memory errors | `ddev config --php-memory-limit=-1` or use `ddev exec COMPOSER_MEMORY_LIMIT=-1 composer ...` |
| Stale containers | `ddev delete -O && ddev start` for a fresh container set |
| Permission issues | `ddev exec chown -R $(id -u):$(id -g) /var/www/html` |
| SSL certificate errors | `mkcert -install` then `ddev restart` |

## Related Skills

- **drush-commands** -- Comprehensive Drush command reference
- **drush-generate** -- Code scaffolding with `ddev drush gen`
- **drupal-patterns** -- Decision framework for choosing Drupal patterns
