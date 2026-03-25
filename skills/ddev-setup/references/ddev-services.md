# DDEV Services and Add-ons Reference

## Overview

DDEV extends its base environment through add-ons (formerly called "additional services"). Add-ons install via `ddev get` and typically add a Docker Compose file to `.ddev/` along with any required configuration. DDEV maintains an official set of add-ons and also supports third-party and custom add-ons.

### Managing add-ons

```bash
ddev get --list                            # List available official add-ons
ddev get <add-on>                          # Install an add-on
ddev get --installed                       # List installed add-ons
ddev get --remove <add-on>                 # Remove an installed add-on
```

After installing or removing an add-on, restart the project:

```bash
ddev restart
```

## Redis

Redis provides an in-memory data store used as a cache backend for Drupal.

### Installation

```bash
ddev get ddev/ddev-redis
ddev restart
```

This adds a Redis 7 container accessible at `redis:6379` from within the DDEV network.

### Drupal configuration

Install the Redis module:

```bash
ddev composer require drupal/redis
ddev drush pm:enable redis -y
```

Add to `settings.php` or `settings.ddev.php`:

```php
$settings['redis.connection']['interface'] = 'PhpRedis';
$settings['redis.connection']['host'] = 'redis';
$settings['redis.connection']['port'] = '6379';

$settings['cache']['default'] = 'cache.backend.redis';
$settings['container_yamls'][] = 'modules/contrib/redis/example.services.yml';

// Optional: Use Redis for the lock and flood control systems.
$settings['container_yamls'][] = 'modules/contrib/redis/redis.services.yml';
```

### Redis CLI

```bash
ddev redis-cli                              # Open Redis CLI
ddev redis-cli INFO                         # Get Redis server info
ddev redis-cli FLUSHALL                     # Clear all Redis data
ddev redis-cli MONITOR                      # Watch real-time commands
```

### Verifying Redis is working

```bash
ddev drush cache:rebuild
ddev redis-cli DBSIZE                       # Should show non-zero key count
```

## Solr

Apache Solr provides full-text search capabilities for Drupal via the Search API module.

### Installation

```bash
ddev get ddev/ddev-solr
ddev restart
```

This adds a Solr 9 container accessible at `http://solr:8983` from within the DDEV network. The Solr admin UI is available at `https://<project>.ddev.site:8983`.

### Drupal configuration

Install the Search API Solr module:

```bash
ddev composer require drupal/search_api_solr
ddev drush pm:enable search_api_solr -y
```

Configure in the Drupal admin UI:

1. Go to `/admin/config/search/search-api`
2. Add a new server with the Solr backend
3. Configure the connection:
   - **HTTP protocol**: `http`
   - **Solr host**: `solr`
   - **Solr port**: `8983`
   - **Solr path**: `/`
   - **Solr core**: `dev` (default core created by the add-on)

### Generating Solr configuration

The Search API Solr module can generate Solr configuration files tailored to your Drupal setup:

```bash
ddev drush search-api-solr:get-server-config <server_id> solr-config.zip
```

Upload the generated config to Solr or place the files in the Solr configset directory.

### Custom Solr configuration

To use custom Solr config, place files in `.ddev/solr/conf/` before starting:

```
.ddev/solr/conf/
  schema.xml
  solrconfig.xml
  elevate.xml
  stopwords.txt
  synonyms.txt
```

## Elasticsearch

Elasticsearch provides an alternative search backend to Solr.

### Installation

```bash
ddev get ddev/ddev-elasticsearch
ddev restart
```

This adds an Elasticsearch 8 container accessible at `http://elasticsearch:9200` from within the DDEV network.

### Drupal configuration

Install the Elasticsearch Connector and Search API modules:

```bash
ddev composer require drupal/elasticsearch_connector drupal/search_api
ddev drush pm:enable elasticsearch_connector search_api -y
```

Configure the cluster at `/admin/config/search/elasticsearch-connector`:

- **Cluster URL**: `http://elasticsearch:9200`

### Verifying Elasticsearch

```bash
ddev exec curl -s http://elasticsearch:9200 | python3 -m json.tool
ddev exec curl -s http://elasticsearch:9200/_cluster/health | python3 -m json.tool
```

### Memory tuning

If Elasticsearch runs out of memory, override via a docker-compose file:

File: `.ddev/docker-compose.elasticsearch-override.yaml`

```yaml
services:
  elasticsearch:
    environment:
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
```

## Memcached

Memcached is an alternative in-memory cache backend to Redis.

### Installation

```bash
ddev get ddev/ddev-memcached
ddev restart
```

This adds a Memcached container accessible at `memcached:11211` from within the DDEV network.

### Drupal configuration

Install the Memcache module:

```bash
ddev composer require drupal/memcache
ddev drush pm:enable memcache -y
```

Add to `settings.php` or `settings.ddev.php`:

```php
$settings['memcache']['servers'] = [
  'memcached:11211' => 'default',
];
$settings['memcache']['bins'] = [
  'default' => 'default',
];
$settings['memcache']['key_prefix'] = 'my_project_';
$settings['cache']['default'] = 'cache.backend.memcache';
```

### Verifying Memcached

```bash
ddev exec echo "stats" | nc memcached 11211
```

## Mailpit (Built-in Email Testing)

Mailpit is included in DDEV by default -- no add-on installation is needed. It captures all outgoing email from the Drupal application for local testing.

### Accessing Mailpit

```bash
ddev launch -m                              # Open Mailpit UI in browser
```

The Mailpit web UI is also available at `https://<project>.ddev.site:8026`.

### How it works

DDEV configures the web container to route all PHP `mail()` and SMTP traffic through Mailpit. Drupal sends email normally, and Mailpit captures it instead of delivering to real recipients. No Drupal module installation is required for basic email capture.

### SMTP configuration

If your Drupal site uses an SMTP module (e.g., `symfony_mailer`), configure it to use the Mailpit SMTP server:

- **SMTP host**: `localhost`
- **SMTP port**: `1025`
- **Encryption**: None
- **Authentication**: None

### Mailpit API

Mailpit provides a REST API for programmatic access (useful in automated tests):

```bash
# List messages
ddev exec curl -s http://localhost:8025/api/v1/messages | python3 -m json.tool

# Delete all messages
ddev exec curl -s -X DELETE http://localhost:8025/api/v1/messages
```

## phpMyAdmin

phpMyAdmin provides a web-based database management interface.

### Installation

```bash
ddev get ddev/ddev-phpmyadmin
ddev restart
```

### Accessing phpMyAdmin

```bash
ddev phpmyadmin                             # Open phpMyAdmin in browser
```

The interface is available at `https://<project>.ddev.site:8036`. It connects automatically to the project database with no additional credentials needed.

## Adminer

Adminer is a lightweight alternative to phpMyAdmin that supports MySQL, MariaDB, and PostgreSQL.

### Installation

```bash
ddev get ddev/ddev-adminer
ddev restart
```

### Accessing Adminer

```bash
ddev adminer                                # Open Adminer in browser
```

## Custom Services via Docker Compose

You can add any Docker-based service by creating a Docker Compose file in the `.ddev/` directory.

### Naming convention

Files must follow the pattern `.ddev/docker-compose.<service-name>.yaml` to be picked up by DDEV.

### Required labels

Custom service containers should include DDEV labels for proper lifecycle management:

```yaml
services:
  my-service:
    container_name: ddev-${DDEV_SITENAME}-my-service
    image: my-service-image:latest
    restart: "no"
    labels:
      com.ddev.site-name: ${DDEV_SITENAME}
      com.ddev.approot: ${DDEV_APPROOT}
    environment:
      - VIRTUAL_HOST=$DDEV_HOSTNAME
      - HTTP_EXPOSE=8080:8080
      - HTTPS_EXPOSE=9443:8080
    volumes:
      - ".:/mnt/ddev_config"
```

### Environment variables available in Docker Compose

| Variable | Description |
|---|---|
| `${DDEV_SITENAME}` | The project name |
| `${DDEV_APPROOT}` | The project root directory on the host |
| `${DDEV_HOSTNAME}` | The primary hostname |
| `${DDEV_HOST_DB_PORT}` | The host-exposed database port |
| `${DDEV_HOST_WEBSERVER_PORT}` | The host-exposed HTTP port |
| `${DDEV_HOST_HTTPS_PORT}` | The host-exposed HTTPS port |

### Exposing service ports via the DDEV router

Use the `HTTP_EXPOSE` and `HTTPS_EXPOSE` environment variables to route traffic through the DDEV router:

```yaml
environment:
  - HTTP_EXPOSE=8080:8080        # hostPort:containerPort
  - HTTPS_EXPOSE=9443:8080       # hostPort:containerPort (with TLS termination)
```

### Example: Storybook service

File: `.ddev/docker-compose.storybook.yaml`

```yaml
services:
  storybook:
    container_name: ddev-${DDEV_SITENAME}-storybook
    image: node:20-alpine
    restart: "no"
    working_dir: /app/web/themes/custom/my_theme
    command: npx storybook dev -p 6006 --ci
    labels:
      com.ddev.site-name: ${DDEV_SITENAME}
      com.ddev.approot: ${DDEV_APPROOT}
    environment:
      - VIRTUAL_HOST=$DDEV_HOSTNAME
      - HTTP_EXPOSE=6006:6006
      - HTTPS_EXPOSE=6007:6006
    volumes:
      - ../:/app:cached
```

### Example: Varnish cache

File: `.ddev/docker-compose.varnish.yaml`

```yaml
services:
  varnish:
    container_name: ddev-${DDEV_SITENAME}-varnish
    image: varnish:7.4
    restart: "no"
    labels:
      com.ddev.site-name: ${DDEV_SITENAME}
      com.ddev.approot: ${DDEV_APPROOT}
    environment:
      - VIRTUAL_HOST=$DDEV_HOSTNAME
      - HTTP_EXPOSE=8080:80
    volumes:
      - ".:/mnt/ddev_config"
      - "./varnish/default.vcl:/etc/varnish/default.vcl"
    command: varnishd -F -a :80 -b web:80 -f /etc/varnish/default.vcl
```

## Connecting Services to Each Other

All DDEV containers for a project share a Docker network. Services can reference each other by their container service name:

| Service | Internal Hostname | Default Port |
|---|---|---|
| Web (Drupal) | `web` | 80 / 443 |
| Database | `db` | 3306 (MySQL/MariaDB) or 5432 (PostgreSQL) |
| Redis | `redis` | 6379 |
| Solr | `solr` | 8983 |
| Elasticsearch | `elasticsearch` | 9200 |
| Memcached | `memcached` | 11211 |

## Troubleshooting Services

### Check running containers

```bash
ddev describe                              # Shows all services and their status
docker ps --filter "label=com.ddev.site-name=my-project"
```

### View service logs

```bash
ddev logs -s <service-name>                # e.g., ddev logs -s redis
```

### Restart a specific service

Restarting the entire project is the supported approach:

```bash
ddev restart
```

### Service not starting

Check for port conflicts and resource constraints:

```bash
ddev poweroff                              # Stop everything cleanly
docker system prune                        # Clean up unused resources
ddev start                                 # Fresh start
```

### Reset an add-on

Remove and reinstall:

```bash
ddev get --remove ddev/ddev-redis
ddev get ddev/ddev-redis
ddev restart
```
