# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Docker-based WordPress deployment stack designed for Coolify deployment platform. The stack consists of:
- WordPress with PHP-FPM (Alpine-based)
- Nginx web server
- MariaDB database
- Redis object cache

## Architecture

### Service Stack

The [docker-compose.yaml](docker-compose.yaml) defines a 7-service WordPress stack:

1. **nginx** - Custom Alpine-based web server
   - Built from [nginx/Dockerfile](nginx/Dockerfile)
   - Exposes port 80 internally (Coolify proxy handles external routing)
   - Uses `coolify.port=80` label for Coolify integration
   - Proxies PHP requests to WordPress PHP-FPM (port 9000)
   - Serves static files directly
   - Includes custom [nginx/default.conf](nginx/default.conf) baked into image
   - Read-only access to WordPress files via shared volume

2. **wordpress** - Custom PHP-FPM container
   - Built from [wordpress/Dockerfile](wordpress/Dockerfile)
   - Processes PHP requests from Nginx
   - Connects to MariaDB and Redis
   - Contains WordPress core files and plugins

3. **plugin-installer** - One-time initialization job
   - Built from [plugin-installer/Dockerfile](plugin-installer/Dockerfile)
   - Runs once after WordPress is ready (`restart: "no"`)
   - Downloads and installs Redis Object Cache plugin
   - Shares WordPress volume to install plugin files
   - Automatically terminates after completion

4. **db** - MariaDB 10.6 database
   - Stores WordPress content and configuration
   - Health-checked before WordPress starts

5. **redis** - Alpine-based Redis cache
   - Object caching for improved performance
   - 256MB memory limit with LRU eviction

6. **filebrowser** - Web-based file manager
   - Provides web UI for managing WordPress files
   - Mounts `wordpress_data` volume at `/srv`
   - Default credentials: admin/admin (change after first login)
   - Accessible via subdomain (e.g., files.yourdomain.com)

7. **phpmyadmin** - Database management interface
   - Web UI for MariaDB administration
   - Auto-configured to connect to `db` service
   - Supports 256MB upload limit for SQL imports
   - Accessible via subdomain (e.g., pma.yourdomain.com)

### Custom WordPress Image

The [wordpress/Dockerfile](wordpress/Dockerfile) extends `wordpress:php8.3-fpm` with:
- PHP Redis extension via PECL
- Additional image processing libraries (GD, JPEG, PNG)
- OPcache optimization
- Custom PHP settings:
  - `memory_limit`: 256M
  - `upload_max_filesize`: 256M
  - `post_max_size`: 256M
  - `max_execution_time`: 300s
  - `max_input_time`: 300s
- Pre-configured file permissions for www-data

**Important**: The docker-compose.yaml uses `build: ./wordpress` to build this custom image rather than pulling the standard WordPress image from Docker Hub.

### Plugin Installer Service

The [plugin-installer](plugin-installer/) directory contains a one-shot Alpine-based container that:
- Waits for WordPress to be healthy and wp-config.php to exist
- Downloads Redis Object Cache plugin from wordpress.org
- Extracts plugin to wp-content/plugins directory
- Runs once and terminates (restart policy: "no")
- Plugin requires manual activation via WordPress admin after installation

### Request Flow

1. HTTP request arrives at **Nginx** (port 80)
2. For PHP files: Nginx forwards to **WordPress PHP-FPM** (port 9000)
3. WordPress queries **MariaDB** for content
4. WordPress checks **Redis** for cached objects
5. Response flows back: WordPress → Nginx → Client

### Nginx Configuration

[nginx/default.conf](nginx/default.conf) provides:
- PHP-FPM connection to wordpress:9000
- WordPress permalink rewriting
- Static asset caching (JS, CSS, images)
- Health check endpoint at `/healthz`
- Upload size limit: 256M (`client_max_body_size`)
- FastCGI timeouts: 300s for read/send operations

## Deployment

### Coolify Deployment

Deploy via the Coolify UI:

1. Create a new application in Coolify
2. Select "Docker Compose" as build pack
3. Point to your Git repository
4. Configure domains:
   - **Domains for nginx**: Your domain (e.g., https://yourdomain.com)
   - **Domains for filebrowser**: File manager (e.g., https://files.yourdomain.com)
   - **Domains for phpmyadmin**: Database admin (e.g., https://pma.yourdomain.com)
   - **Domains for wordpress**: Leave empty
5. Save and deploy

Coolify will automatically:
- Clone your repository
- Build custom Docker images
- Start all services with health checks
- Route traffic through its proxy to nginx

### Local Development

For local testing, you may want to add port mapping to the nginx service:

```yaml
# Add this to nginx service in docker-compose.yaml for local development only
ports:
  - "8080:80"  # Access at http://localhost:8080
```

Then run:

```bash
# Build and start the stack
docker-compose up -d --build

# View logs
docker-compose logs -f

# Rebuild after Dockerfile changes
docker-compose build --no-cache wordpress nginx
docker-compose up -d
```

**Note**: Remove the `ports:` section before deploying to Coolify. Coolify uses `expose` and labels instead.

## WordPress Configuration

Redis integration is pre-configured via `WORDPRESS_CONFIG_EXTRA`:
- `WP_REDIS_HOST`: redis
- `WP_REDIS_PORT`: 6379
- `WP_CACHE`: true
- Redis timeout settings: 1 second

After deployment, activate the Redis Object Cache plugin via WordPress admin to enable caching.

## Git Workflow

Standard git workflow:
```bash
git add .
git commit -m "Your commit message"
git push
```

Coolify will automatically detect the push and redeploy if instant deploy is enabled.

## Important Notes

- Production uses Coolify-injected environment variables (`$SERVICE_USER_WORDPRESS`, `$SERVICE_PASSWORD_ROOT`)
- The Redis maxmemory policy is set to `allkeys-lru` (256MB limit)
- All services have health checks with 10-second intervals
- WordPress requires database and Redis to be healthy before starting
- Custom images are built from [wordpress/Dockerfile](wordpress/Dockerfile) and [nginx/Dockerfile](nginx/Dockerfile) - changes require rebuild
- PHP and Nginx are configured to support 256MB uploads and 300-second execution times
- Configuration files are baked into Docker images (not mounted) for Coolify compatibility
- Port mapping uses `expose` and `coolify.port` label instead of `ports` to avoid conflicts with Coolify's proxy
