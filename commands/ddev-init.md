---
name: ddev-init
description: Interactive DDEV + Drupal project setup wizard. Usage: /ddev-init [drupal_version]
argument-hint: "[drupal_version]"
allowed-tools: [Bash, Read, Write, Edit]
---

# DDEV Drupal Setup Command

Guide the user through setting up a Drupal project with DDEV.

## Arguments

The user provided: $ARGUMENTS

If a Drupal version is specified (10, 11), use it. Otherwise default to Drupal 11.

## Workflow

### Step 1: Check Prerequisites

Verify required tools are installed:

```bash
ddev version
```

```bash
docker info --format '{{.ServerVersion}}'
```

If DDEV is not installed, tell the user:
> DDEV is not installed. Install it with:
> - macOS: `brew install ddev/ddev/ddev`
> - Linux: `curl -fsSL https://ddev.com/install.sh | bash`
> - See: https://ddev.readthedocs.io/en/stable/users/install/

If Docker is not running, tell the user to start Docker Desktop or Colima.

### Step 2: Determine Setup Type

Check the current directory state:

```bash
ls -la composer.json .ddev/config.yaml 2>/dev/null
```

- **Has `composer.json` with `drupal/core`** → Existing Drupal project
- **Has `.ddev/config.yaml`** → DDEV already configured
- **Empty or no Drupal** → New project

### Step 3a: New Drupal Project

Set PHP version based on Drupal version:
- Drupal 11: PHP 8.3
- Drupal 10: PHP 8.2 or 8.3

```bash
# Configure DDEV
ddev config --project-type=drupal --php-version=8.3 --docroot=web --database=mariadb:10.11

# Start DDEV
ddev start

# Create Drupal project
ddev composer create drupal/recommended-project

# Install Drupal
ddev drush site:install standard --account-name=admin --account-pass=admin --site-name="My Drupal Site" -y

# Get login link
ddev drush uli
```

### Step 3b: Existing Drupal Project

```bash
# Configure DDEV (auto-detect settings)
ddev config --project-type=drupal --docroot=web

# Start DDEV
ddev start

# Install dependencies
ddev composer install
```

Ask the user about the database:
1. **Import existing database**: `ddev import-db --file=path/to/db.sql.gz`
2. **Fresh install**: `ddev drush site:install standard -y`
3. **Pull from remote**: Configure `ddev pull` provider

### Step 4: Post-Setup Configuration

#### Config sync directory
```bash
mkdir -p config/sync
```

Check if `settings.php` has the config sync directory configured. If not, add:
```php
$settings['config_sync_directory'] = '../config/sync';
```

#### Development modules (ask user first)
```bash
ddev composer require --dev drupal/devel drupal/admin_toolbar drupal/config_split drupal/stage_file_proxy
ddev drush en admin_toolbar admin_toolbar_tools -y
```

#### Development settings
Check if `web/sites/default/settings.local.php` exists. If not, copy the example:
```bash
cp web/sites/example.settings.local.php web/sites/default/settings.local.php
```

Ensure `settings.php` includes it:
```php
if (file_exists($app_root . '/' . $site_path . '/settings.local.php')) {
  include $app_root . '/' . $site_path . '/settings.local.php';
}
```

#### Drush configuration
Create `drush/drush.yml` if it doesn't exist:
```yaml
options:
  uri: ${DDEV_PRIMARY_URL}
```

### Step 5: Report

```
## DDEV Drupal Setup Complete!

### Your site:
- **URL:** {output of ddev describe primary URL}
- **Admin login:** {output of ddev drush uli}
- **Database:** MariaDB 10.11
- **PHP:** 8.3

### Useful commands:
| Command | Description |
|---------|-------------|
| `ddev start` / `ddev stop` | Start/stop the environment |
| `ddev drush <command>` | Run Drush commands |
| `ddev composer <command>` | Run Composer commands |
| `ddev ssh` | SSH into the web container |
| `ddev logs` | View container logs |
| `ddev xdebug on` | Enable Xdebug |
| `ddev snapshot` | Create a database snapshot |
| `ddev describe` | Show project info |
| `ddev drush cr` | Clear all caches |
| `ddev drush uli` | Get admin login link |
| `ddev drush cex -y` | Export configuration |
| `ddev drush cim -y` | Import configuration |

### Next steps:
- Install contrib modules: `ddev composer require drupal/{module}`
- Enable modules: `ddev drush en {module} -y`
- Learn a module's API: `/learn-module {module_name}`
- Start building: `/drupal-feature "your feature description"`
```
