# Relaticle CRM - Deployment Guide Summary

Quick reference for all deployment methods. For detailed instructions, see the specific guides.

---

## Deployment Options

### 1. Docker Compose (Production)

**Best for**: Production servers, VPS, dedicated hosting

**File**: `docker-compose.prod.yml`

**Quick start**:
```bash
# Copy and configure .env
cp .env.example .env
# Edit .env with your settings

# Generate APP_KEY
docker run --rm ghcr.io/relaticle/relaticle:latest php artisan key:generate --show
# Add to .env

# Deploy
docker compose -f docker-compose.prod.yml up -d
```

**Services included**:
- App (web + API)
- Horizon (queue worker)
- Scheduler (cron jobs)
- PostgreSQL 16
- Redis 7

**Docs**: See `docker-compose.prod.yml` comments

---

### 2. Unraid

**Best for**: Unraid servers, home lab

**Methods**:
- Docker Compose Stack (recommended)
- Community Applications Template
- Manual container setup

**Quick start**:
```bash
mkdir -p /mnt/user/appdata/relaticle-compose
cd /mnt/user/appdata/relaticle-compose
wget https://raw.githubusercontent.com/Relaticle/relaticle/main/unraid/docker-compose-unraid.yml
wget https://raw.githubusercontent.com/Relaticle/relaticle/main/unraid/.env.unraid -O .env
# Edit .env
docker compose -f docker-compose-unraid.yml up -d
```

**Docs**: `unraid/UNRAID-DEPLOYMENT.md`

---

### 3. Single Docker Container

**Best for**: Quick testing, minimal setup

**Quick start**:
```bash
# Generate key
APP_KEY=$(docker run --rm ghcr.io/relaticle/relaticle:latest php artisan key:generate --show)

# Run (requires external PostgreSQL and Redis)
docker run -d \
  --name relaticle \
  -p 8080:8080 \
  -e APP_KEY="$APP_KEY" \
  -e DB_HOST=postgres \
  -e DB_PASSWORD=secret \
  -e REDIS_HOST=redis \
  -v /path/to/storage:/var/www/html/storage/app \
  ghcr.io/relaticle/relaticle:latest
```

**Note**: Requires PostgreSQL and Redis running separately

---

### 4. Kubernetes (Advanced)

**Best for**: Enterprise, high availability, auto-scaling

**Requirements**:
- Kubernetes cluster
- Helm (recommended)
- Persistent volumes
- PostgreSQL operator or external DB
- Redis operator or external cache

**Basic deployment** (without Helm):
```yaml
# Create namespace
kubectl create namespace relaticle

# Deploy PostgreSQL (example using operator)
# Deploy Redis
# Deploy Relaticle app
# Deploy Horizon
# Deploy Scheduler
# Configure Ingress
```

**Note**: Full Helm chart coming soon

---

### 5. Laravel Sail (Development)

**Best for**: Local development, testing

**File**: `docker-compose.yml`

**Quick start**:
```bash
# Install dependencies
composer install

# Copy .env
cp .env.example .env

# Generate key
php artisan key:generate

# Start Sail
./vendor/bin/sail up -d

# Run migrations
./vendor/bin/sail artisan migrate --seed
```

**Services included**:
- Laravel app
- MySQL 8
- Redis
- Meilisearch
- Mailpit
- Selenium

**Access**: http://localhost

---

## Architecture Overview

### Components

```
┌─────────────────┐
│   Web Browser   │
└────────┬────────┘
         │ HTTP/HTTPS
         ▼
┌─────────────────┐
│  Nginx + PHP    │ ◄── Main App Container
│  (Port 8080)    │
└────────┬────────┘
         │
    ┌────┴────┬─────────┬─────────┐
    ▼         ▼         ▼         ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│ Postgres│ │ Redis  │ │Horizon │ │Scheduler│
│  (DB)  │ │(Cache) │ │(Queue) │ │ (Cron)  │
└────────┘ └────────┘ └────────┘ └────────┘
```

### Container Roles

1. **App**: Main web server
   - Nginx + PHP-FPM
   - Serves web UI and API
   - Handles HTTP requests
   - Runs migrations on startup (if AUTORUN_ENABLED=true)

2. **Horizon**: Queue worker
   - Processes background jobs
   - Handles async tasks
   - Email sending, notifications
   - Web UI at `/sysadmin/horizon`

3. **Scheduler**: Cron jobs
   - Runs scheduled tasks
   - Maintenance commands
   - Periodic data updates

4. **PostgreSQL**: Database
   - Primary data store
   - Version 16 recommended
   - Persistent volume required

5. **Redis**: Cache & Queue
   - Session storage
   - Application cache
   - Queue backend
   - Volatile data OK

---

## Required Environment Variables

### Minimum Required

```env
APP_KEY=base64:...          # Generate with artisan key:generate
APP_URL=http://localhost    # Your application URL
DB_HOST=postgres            # Database host
DB_PASSWORD=secret          # Database password
REDIS_HOST=redis            # Redis host
```

### Production Recommended

```env
APP_ENV=production
APP_DEBUG=false
LOG_LEVEL=warning
CACHE_STORE=redis
SESSION_DRIVER=redis
QUEUE_CONNECTION=redis
AUTORUN_ENABLED=true
AUTORUN_LARAVEL_MIGRATION=true
```

### Full Example

See `.env.example` or `unraid/.env.unraid`

---

## Port Mapping

| Service    | Internal Port | External Port (default) | Configurable |
|------------|---------------|-------------------------|--------------|
| App        | 8080          | 8080                    | Yes          |
| PostgreSQL | 5432          | Not exposed             | Optional     |
| Redis      | 6379          | Not exposed             | Optional     |
| Horizon    | N/A (CLI)     | N/A                     | N/A          |
| Scheduler  | N/A (CLI)     | N/A                     | N/A          |

**Note**: For production, use reverse proxy (Nginx, Traefik, Caddy) in front of app container.

---

## Volume Mapping

### Required Volumes

| Container Path               | Purpose                  | Required |
|------------------------------|--------------------------|----------|
| `/var/www/html/storage/app`  | User uploads and files   | Yes      |

### Optional Volumes

| Container Path               | Purpose                  | Required |
|------------------------------|--------------------------|----------|
| `/var/lib/postgresql/data`   | PostgreSQL data          | Yes (DB) |
| `/data`                      | Redis data (persistence) | Optional |

**Important**: Storage volume must be writable by `www-data` (UID 33, GID 33)

---

## Health Checks

All production deployments should include health checks:

```yaml
# App container
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/up"]
  interval: 30s
  timeout: 5s
  retries: 3
  start_period: 60s

# Horizon container
healthcheck:
  test: ["CMD", "php", "/var/www/html/artisan", "horizon:status"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 30s

# PostgreSQL
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U relaticle -d relaticle"]
  interval: 10s
  timeout: 5s
  retries: 5

# Redis
healthcheck:
  test: ["CMD", "redis-cli", "ping"]
  interval: 10s
  timeout: 5s
  retries: 5
```

---

## SSL/TLS Configuration

### Option 1: Reverse Proxy (Recommended)

Use Nginx, Traefik, or Caddy in front:

```nginx
server {
    listen 443 ssl http2;
    server_name crm.example.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Update APP_URL:
```env
APP_URL=https://crm.example.com
```

### Option 2: Let's Encrypt with Certbot

Use with reverse proxy for automatic certificate management.

---

## Scaling

### Horizontal Scaling

**App containers**: Can run multiple replicas
- Use load balancer (HAProxy, Nginx, cloud LB)
- Shared storage required (NFS, S3, etc.)
- Session stored in Redis (stateless)

**Horizon containers**: Can run multiple workers
- Configure in `config/horizon.php`
- Each worker processes different queues

**Scheduler**: Run only ONE instance
- Uses mutex locks to prevent duplicates

### Vertical Scaling

**Resource recommendations**:

| Deployment Size | CPU   | RAM   | Storage |
|-----------------|-------|-------|---------|
| Small (<100)    | 2 core| 2 GB  | 10 GB   |
| Medium (<500)   | 4 core| 4 GB  | 50 GB   |
| Large (<2000)   | 8 core| 8 GB  | 200 GB  |
| XL (>2000)      | 16+   | 16+ GB| 500+ GB |

**Database**: Scale separately based on data size

---

## Monitoring

### Built-in

- **Horizon Dashboard**: `/sysadmin/horizon`
  - Queue metrics
  - Job failures
  - Worker status

- **Laravel Telescope** (if enabled): `/sysadmin/telescope`
  - Request logging
  - Query profiling
  - Exception tracking

### External

Recommended monitoring stack:
- **Prometheus**: Metrics collection
- **Grafana**: Dashboards
- **Loki**: Log aggregation
- **Sentry**: Error tracking (configure SENTRY_LARAVEL_DSN)

---

## Backup Strategy

### What to Backup

1. **PostgreSQL database** (critical)
   ```bash
   docker exec relaticle-postgres pg_dump -U relaticle relaticle > backup.sql
   ```

2. **Storage volume** (user uploads)
   ```bash
   tar -czf storage-backup.tar.gz /path/to/storage
   ```

3. **Environment config** (.env file)

4. **Redis** (optional, can be regenerated)

### Backup Frequency

- Database: Daily full + hourly incremental
- Storage: Daily incremental
- Config: On change

### Restore Process

1. Deploy clean installation
2. Restore database
3. Restore storage files
4. Configure environment
5. Run migrations (if needed)
6. Clear caches

---

## Update Process

### Docker Compose

```bash
# Pull new images
docker compose -f docker-compose.prod.yml pull

# Stop services
docker compose -f docker-compose.prod.yml down

# Start with new images
docker compose -f docker-compose.prod.yml up -d

# Migrations run automatically if AUTORUN_ENABLED=true
# Otherwise, run manually:
docker exec relaticle-app php artisan migrate --force
```

### Zero-Downtime Updates

1. Run new app containers alongside old
2. Run migrations (compatible with old version)
3. Switch traffic to new containers
4. Shut down old containers

**Note**: Requires careful migration design for compatibility

---

## Troubleshooting Deployment

### Container Won't Start

1. Check logs: `docker logs container-name`
2. Verify environment variables
3. Check database connectivity
4. Verify Redis connectivity
5. Check volume permissions

### Database Connection Failed

1. Verify PostgreSQL is running
2. Check credentials
3. Test connection: `docker exec app php artisan tinker` → `DB::connection()->getPdo()`
4. Check network connectivity

### Redis Connection Failed

1. Verify Redis is running
2. Check host/port
3. Test: `docker exec app php artisan tinker` → `Redis::ping()`

### 500 Internal Server Error

1. Check logs: `storage/logs/laravel.log`
2. Verify APP_KEY is set
3. Run: `docker exec app php artisan optimize:clear`
4. Check file permissions

### Assets Not Loading

1. Verify frontend was built (check Dockerfile logs)
2. Run: `docker exec app php artisan storage:link`
3. Check nginx config

### Queue Not Processing

1. Check Horizon: `docker logs relaticle-horizon`
2. Verify Redis connection
3. Restart Horizon: `docker restart relaticle-horizon`
4. Check queue configuration

---

## Security Checklist

- [ ] Generate strong APP_KEY
- [ ] Use strong database password
- [ ] Enable HTTPS (SSL/TLS)
- [ ] Set APP_DEBUG=false in production
- [ ] Configure firewall (only expose necessary ports)
- [ ] Regular security updates
- [ ] Database backups configured
- [ ] Use Redis password (optional but recommended)
- [ ] Configure CORS if needed
- [ ] Enable 2FA for admin accounts
- [ ] Review file upload permissions
- [ ] Set up log monitoring
- [ ] Configure rate limiting

---

## Performance Tuning

### App Container

```env
# Cache everything
AUTORUN_LARAVEL_CONFIG_CACHE=true
AUTORUN_LARAVEL_ROUTE_CACHE=true
AUTORUN_LARAVEL_VIEW_CACHE=true

# OPcache enabled (default in serversideup/php)
PHP_OPCACHE_ENABLE=1
```

### Database

- Enable connection pooling (PgBouncer)
- Tune PostgreSQL settings for workload
- Regular VACUUM and ANALYZE
- Index optimization

### Redis

- Enable persistence if needed: `appendonly yes`
- Tune memory limits
- Consider Redis Cluster for large deployments

### CDN

- Serve static assets via CDN
- Configure Laravel Mix/Vite for CDN URLs
- Use S3 or compatible for file storage

---

## Support

- **Issues**: https://github.com/Relaticle/relaticle/issues
- **Discussions**: https://github.com/Relaticle/relaticle/discussions
- **Discord**: https://discord.gg/b9WxzUce4Q
- **Docs**: https://relaticle.com/documentation

---

**Last Updated**: December 2025
