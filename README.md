# WordPress Stack with Nginx, MariaDB, and Redis

A production-ready WordPress deployment stack optimized for Coolify, featuring:

- **WordPress PHP-FPM 8.3** - Custom-built with Redis extension and optimized PHP settings
- **Nginx Alpine** - Custom web server with WordPress-optimized configuration
- **MariaDB 10.6** - Reliable database with health checks
- **Redis Alpine** - Object caching for improved performance
- **FileBrowser** - Web-based file manager for WordPress files
- **phpMyAdmin** - Database management interface
- **Redis Object Cache Plugin** - Automatically installed (requires activation)

## Quick Start

### Coolify Deployment

1. **Create a new application** in Coolify
2. **Select "Docker Compose"** as the build pack
3. **Point to this Git repository**: `https://github.com/itsmereal/coolify-wordpress-maria`
4. **Configure domains**:
   - **Domains for nginx**: Your main domain (e.g., `https://yourdomain.com`)
   - **Domains for filebrowser**: File manager subdomain (e.g., `https://files.yourdomain.com`)
   - **Domains for phpmyadmin**: Database admin subdomain (e.g., `https://pma.yourdomain.com`)
   - **Domains for wordpress**: Leave empty (internal service only)
5. **Save and Deploy**

The deployment will:
- Build custom Docker images for WordPress, Nginx, and plugin installer
- Start all services with health checks
- Automatically install Redis Object Cache plugin
- Route your domain traffic through Coolify's proxy to Nginx

### Service Architecture

```
                          ┌─→ Nginx ─→ WordPress PHP-FPM
                          │              ↓
Internet → Coolify Proxy ─┼─→ FileBrowser ─→ wordpress_data volume
                          │
                          └─→ phpMyAdmin ─→ MariaDB

                             Redis (internal)
```

## Features

### Performance Optimizations
- **PHP-FPM** (more efficient than Apache)
- **OPcache** configured for optimal performance
- **Redis object caching** with LRU eviction (256MB limit)
- **Static asset caching** (JS, CSS, images)
- **Nginx** serving static files directly

### PHP Settings
- Upload limit: **256MB**
- POST size limit: **256MB**
- Execution time: **300 seconds**
- Memory limit: **256MB**

### Custom Images
- Custom WordPress image with Redis extension and optimized PHP settings
- Custom Nginx image with WordPress-optimized configuration baked in
- All configurations are version-controlled and reproducible

### High Availability
- Health checks for all services (10-second intervals)
- Service dependencies ensure correct startup order
- Automatic restarts on failure
- Resource limits prevent memory/CPU exhaustion

## Post-Deployment

After successful deployment:

1. **Access WordPress**: Visit your configured domain
2. **Complete WordPress setup**: Follow the installation wizard
3. **Activate Redis Object Cache**:
   - Login to WordPress admin
   - Go to Plugins → Installed Plugins
   - Find "Redis Object Cache" (automatically installed)
   - Click "Activate"
   - Go to Settings → Redis
   - Click "Enable Object Cache"
   - Verify connection shows "Connected"

### Admin Tools

- **FileBrowser** (`files.yourdomain.com`):
  - Default login: `admin` / `admin`
  - ⚠️ **Change password immediately after first login**
  - Browse, upload, edit WordPress files

- **phpMyAdmin** (`pma.yourdomain.com`):
  - Auto-logged in as root
  - Manage WordPress database
  - Import/export SQL files

## Local Development

To test locally before deploying:

```bash
# Add port mapping to docker-compose.yaml nginx service
ports:
  - "8080:80"

# Build and start
docker-compose up -d --build

# Access at http://localhost:8080

# View logs
docker-compose logs -f

# Stop
docker-compose down
```

**Note**: Remove the `ports:` section before pushing to Coolify.

## Configuration Files

- [docker-compose.yaml](docker-compose.yaml) - Main orchestration file
- [wordpress/Dockerfile](wordpress/Dockerfile) - Custom WordPress image with PHP 8.3 FPM
- [nginx/Dockerfile](nginx/Dockerfile) - Custom Nginx image with optimized config
- [nginx/default.conf](nginx/default.conf) - Nginx server configuration
- [CLAUDE.md](CLAUDE.md) - Detailed technical documentation

## Technical Details

See [CLAUDE.md](CLAUDE.md) for comprehensive architecture documentation, including:
- Detailed service descriptions
- Request flow
- Build instructions
- Troubleshooting
- Development workflow
