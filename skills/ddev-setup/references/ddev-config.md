# DDEV Configuration Reference

## .ddev/config.yaml -- Complete Options

The `.ddev/config.yaml` file is the primary configuration file for a DDEV project. It is created by `ddev config` and can be edited manually. After editing, run `ddev restart` to apply changes.

### Core Project Settings

```yaml
name: my-project
```

The project name determines the container names and the default hostname (`my-project.ddev.site`). Must be lowercase alphanumeric with hyphens allowed.

```yaml
type: drupal
```

The project type. For Drupal, use `drupal` (covers Drupal 10/11). DDEV auto-configures settings file management based on this value.

```yaml
docroot: web
```

The document root relative to the project root. For `drupal/recommended-project`, this is `web`. For legacy Drupal installs, it may be empty (project root is the docroot).

### PHP Configuration

```yaml
php_version: "8.3"
```

Supported versions: "8.1", "8.2", "8.3", "8.4". Always quote the version string.

```yaml
xdebug_enabled: false
```

Set to `true` to enable Xdebug at startup. Prefer using `ddev xdebug on/off` at runtime instead.

```yaml
php_memory_limit: "256M"
```

Override the default PHP memory limit. Use "-1" for unlimited.

```yaml
composer_version: "2"
```

Set the major Composer version. Options: "2" (default), "1", or a specific version like "2.7.1".

### Web Server Configuration

```yaml
webserver_type: nginx-fpm
```

Options: `nginx-fpm` (default) or `apache-fpm`. Both use PHP-FPM.

```yaml
additional_hostnames:
  - alt-name
  - another-name
```

Additional hostnames beyond `<project-name>.ddev.site`. Each gets a `.ddev.site` suffix automatically.

```yaml
additional_fqdns:
  - mysite.local
  - dev.mysite.com
```

Fully qualified domain names. Unlike `additional_hostnames`, these are used as-is without appending `.ddev.site`.

```yaml
webserver_https_port: "443"
webserver_http_port: "80"
```

Override the internal container ports. Rarely needed since the DDEV router handles external port mapping.

### Database Configuration

```yaml
database:
  type: mariadb
  version: "10.11"
```

Supported database types and versions:

| Type | Supported Versions |
|---|---|
| mariadb | 10.4, 10.5, 10.6, 10.7, 10.8, 10.11, 11.4 |
| mysql | 5.7, 8.0 |
| postgres | 14, 15, 16 |

Changing the database type or version after initial setup requires exporting and reimporting data. Use `ddev export-db` before making changes.

### Node.js Configuration

```yaml
nodejs_version: "20"
```

Supported versions: "18", "20", "22", or any valid Node.js major version. Node.js and npm/corepack are available inside the web container.

```yaml
corepack_enable: true
```

Enable Corepack for managing Yarn and pnpm versions via the project `package.json`.

### Performance and Resource Settings

```yaml
mutagen_enabled: false
```

Enable Mutagen file synchronization for improved performance on macOS. This is managed globally in most setups via `ddev config global --performance-mode=mutagen`.

```yaml
fail_on_hook_fail: true
```

If set to `true`, DDEV will stop and report an error if any hook command fails. Default is `false`.

```yaml
disable_settings_management: false
```

When `false` (default), DDEV creates and manages `settings.ddev.php` inside the Drupal `sites/default/` directory. Set to `true` if you handle database credentials and other DDEV-specific settings manually.

### Upload Directory

```yaml
upload_dirs:
  - sites/default/files
```

Directories that contain user-uploaded files. Used by `ddev import-files` and for Mutagen excludes.

### Working Directory Defaults

```yaml
working_dir:
  web: /var/www/html
```

Set the default working directory when using `ddev ssh` or `ddev exec`.

### Environment Variables

```yaml
web_environment:
  - SIMPLETEST_BASE_URL=https://my-project.ddev.site
  - SIMPLETEST_DB=mysql://db:db@db/db
  - MINK_DRIVER_ARGS_WEBDRIVER=chrome
  - MY_CUSTOM_VAR=some_value
```

Environment variables injected into the web container. Useful for test runners, custom application config, and CI parity.

## Custom DDEV Commands

Custom commands are shell scripts placed in the `.ddev/commands/` directory. They become available as `ddev <command-name>`.

### Directory structure

```
.ddev/commands/
  host/              # Commands that run on the host machine
    setup.sh
  web/               # Commands that run inside the web container
    compile-theme.sh
  db/                # Commands that run inside the database container
    optimize.sh
```

### Example: Custom host command

File: `.ddev/commands/host/setup.sh`

```bash
#!/bin/bash

## Description: Full project setup from scratch
## Usage: setup
## Example: ddev setup

ddev start
ddev composer install
ddev drush deploy -y
ddev drush uli
ddev launch
```

### Example: Custom web command

File: `.ddev/commands/web/compile-theme.sh`

```bash
#!/bin/bash

## Description: Compile the custom theme assets
## Usage: compile-theme
## Example: ddev compile-theme

cd /var/www/html/web/themes/custom/my_theme
npm install
npm run build
```

Commands must be executable (`chmod +x`). The `## Description`, `## Usage`, and `## Example` comments are required metadata that DDEV parses for the help system.

## Custom Docker Compose Overrides

Place additional Docker Compose files in `.ddev/` with the naming pattern `docker-compose.*.yaml`. DDEV merges them with the main configuration at startup.

### Example: Increase shared memory for the database

File: `.ddev/docker-compose.db-tuning.yaml`

```yaml
services:
  db:
    shm_size: "512m"
    command: >
      --innodb-buffer-pool-size=512M
      --max-allowed-packet=256M
```

### Example: Add a custom container

File: `.ddev/docker-compose.puppeteer.yaml`

```yaml
services:
  puppeteer:
    container_name: ddev-${DDEV_SITENAME}-puppeteer
    image: ghcr.io/puppeteer/puppeteer:latest
    restart: "no"
    labels:
      com.ddev.site-name: ${DDEV_SITENAME}
      com.ddev.approot: ${DDEV_APPROOT}
    volumes:
      - ".:/mnt/ddev_config"
    environment:
      - VIRTUAL_HOST=$DDEV_HOSTNAME
```

## Hooks

Hooks run commands at specific lifecycle points. Define them in `.ddev/config.yaml`:

```yaml
hooks:
  post-start:
    - exec: drush deploy -y
    - exec: drush cache:rebuild

  post-import-db:
    - exec: drush updatedb -y
    - exec: drush cache:rebuild

  pre-stop:
    - exec-host: echo "Stopping project"

  post-stop:
    - exec-host: echo "Project stopped"

  pre-start:
    - exec-host: echo "Starting project"
```

### Hook types

| Hook | When it runs |
|---|---|
| `pre-start` | Before containers start |
| `post-start` | After containers are running and healthy |
| `pre-stop` | Before containers stop |
| `post-stop` | After containers stop |
| `post-import-db` | After a database import completes |
| `post-import-files` | After a file import completes |
| `pre-composer` | Before any `ddev composer` command |
| `post-composer` | After any `ddev composer` command |
| `pre-snapshot` | Before a snapshot is taken |
| `post-snapshot` | After a snapshot is taken |

### Hook command types

- **exec**: Runs inside the web container
- **exec-host**: Runs on the host machine

```yaml
hooks:
  post-start:
    - exec: "command inside web container"
    - exec-host: "command on the host"
```

## Nginx Configuration Customization

Copy the default Nginx config and modify it:

```bash
ddev nginx-config
```

This creates `.ddev/nginx_full/nginx-site.conf`. Edit this file to customize Nginx behavior. Alternatively, add snippet files to `.ddev/nginx/` for partial overrides that get included in the server block:

File: `.ddev/nginx/redirect.conf`

```nginx
if ($host = 'old-domain.ddev.site') {
    return 301 https://my-project.ddev.site$request_uri;
}
```

## Apache Configuration Customization

For Apache projects (`webserver_type: apache-fpm`), customize via `.ddev/apache/apache-site.conf`. Copy the default first:

```bash
ddev exec cp /etc/apache2/apache-site.conf /mnt/ddev_config/apache/apache-site.conf
```

## Custom PHP Configuration

Add PHP ini overrides by placing `.ini` files in `.ddev/php/`:

File: `.ddev/php/my-overrides.ini`

```ini
[PHP]
upload_max_filesize = 128M
post_max_size = 128M
max_execution_time = 300
max_input_vars = 5000
memory_limit = 512M

[opcache]
opcache.memory_consumption = 256
opcache.max_accelerated_files = 20000
```

## Router Configuration

Global DDEV router settings live in `~/.ddev/global_config.yaml`:

```yaml
router_http_port: "80"
router_https_port: "443"
use_hardened_images: false
performance_mode: mutagen
```

To change default ports (useful when port 80/443 are in use):

```bash
ddev config global --router-http-port=8080 --router-https-port=8443
```

## Multiple PHP Versions Across Projects

Each DDEV project has its own `config.yaml` and runs in isolated containers. Different projects can use different PHP versions simultaneously:

```bash
# Project A
cd project-a && ddev config --php-version=8.1

# Project B
cd project-b && ddev config --php-version=8.3
```

Both projects can run at the same time with their respective PHP versions.

## Config Splitting

For team-specific or environment-specific overrides, use `.ddev/config.local.yaml`. This file is typically gitignored and merges with the main `config.yaml`:

```yaml
# .ddev/config.local.yaml -- not committed to git
php_version: "8.4"
web_environment:
  - MY_DEV_KEY=abc123
```

Add to `.gitignore`:

```
.ddev/config.local.yaml
```

DDEV also supports `.ddev/config.*.yaml` files which are loaded in alphabetical order and merged. This allows patterns like:

- `.ddev/config.yaml` -- shared base config (committed)
- `.ddev/config.ci.yaml` -- CI-specific overrides (committed)
- `.ddev/config.local.yaml` -- developer-specific overrides (gitignored)
