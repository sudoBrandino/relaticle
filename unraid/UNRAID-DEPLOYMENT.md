# Relaticle CRM - Unraid Deployment Guide

This guide provides comprehensive instructions for deploying Relaticle CRM on Unraid servers.

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Deployment Methods](#deployment-methods)
4. [Method 1: Docker Compose Stack (Recommended)](#method-1-docker-compose-stack-recommended)
5. [Method 2: Community Applications Template](#method-2-community-applications-template)
6. [Method 3: Manual Container Setup](#method-3-manual-container-setup)
7. [Initial Configuration](#initial-configuration)
8. [Maintenance](#maintenance)
9. [Troubleshooting](#troubleshooting)
10. [Backup and Restore](#backup-and-restore)

## Overview

Relaticle is a modern, self-hosted CRM platform built with Laravel 12, PHP 8.4, and Filament 4. This deployment includes:

- **Main Application**: Web interface and API
- **Horizon**: Queue worker for background jobs
- **Scheduler**: Cron job management
- **PostgreSQL 16**: Primary database
- **Redis 7**: Cache and queue backend

## Prerequisites

- **Unraid Version**: 6.9.0 or later
- **Docker**: Enabled on your Unraid server
- **Disk Space**: Minimum 2GB for application + space for your data
- **RAM**: Minimum 2GB recommended (4GB+ for better performance)
- **CPU**: Any modern x86_64 or ARM64 processor

## Deployment Methods

### Method 1: Docker Compose Stack (Recommended)

The Docker Compose method provides the easiest all-in-one deployment.

#### Step 1: Install Compose Manager Plugin

1. Go to **Apps** tab in Unraid
2. Search for "Compose Manager"
3. Install the Compose Manager plugin

#### Step 2: Create Stack Directory

```bash
mkdir -p /mnt/user/appdata/relaticle-compose
cd /mnt/user/appdata/relaticle-compose
```

#### Step 3: Download Compose File

Download the Unraid-specific compose file:

```bash
wget https://raw.githubusercontent.com/Relaticle/relaticle/main/unraid/docker-compose-unraid.yml
```

Or manually create `docker-compose-unraid.yml` with the contents from the repository.

#### Step 4: Create Environment File

Create a `.env` file with your configuration:

```bash
nano .env
```

Add the following content (customize the values):

```env
# Application Configuration
APP_NAME=Relaticle
APP_ENV=production
APP_DEBUG=false
APP_TIMEZONE=UTC
APP_URL=http://your-unraid-ip:8080
APP_PORT=8080

# Generate this with: docker run --rm ghcr.io/relaticle/relaticle:latest php artisan key:generate --show
APP_KEY=base64:YOUR_GENERATED_KEY_HERE

# Database Configuration
DB_DATABASE=relaticle
DB_USERNAME=relaticle
DB_PASSWORD=your_secure_database_password

# Redis Configuration (optional password)
REDIS_PASSWORD=null

# Mail Configuration (Optional - for notifications)
MAIL_MAILER=log
MAIL_HOST=
MAIL_PORT=587
MAIL_USERNAME=
MAIL_PASSWORD=
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=hello@example.com
MAIL_FROM_NAME=Relaticle

# Storage Path
APPDATA_PATH=/mnt/user/appdata/relaticle

# Logging
LOG_CHANNEL=stderr
LOG_LEVEL=warning
```

#### Step 5: Generate Application Key

Generate a secure APP_KEY:

```bash
docker run --rm ghcr.io/relaticle/relaticle:latest php artisan key:generate --show
```

Copy the output (starts with `base64:`) and paste it into your `.env` file as the `APP_KEY` value.

#### Step 6: Deploy the Stack

Using Compose Manager:

1. Go to **Apps > Compose** in Unraid
2. Click **Add New Stack**
3. Name it "relaticle"
4. Point to `/mnt/user/appdata/relaticle-compose`
5. Click **Compose Up**

Or via command line:

```bash
cd /mnt/user/appdata/relaticle-compose
docker compose -f docker-compose-unraid.yml up -d
```

#### Step 7: Access Relaticle

Wait about 60 seconds for the application to initialize, then access:

```
http://your-unraid-ip:8080
```

The first time you access the application, you'll be prompted to create an admin account.

---

### Method 2: Community Applications Template

*Note: This method requires the Community Applications plugin and the template to be approved.*

#### Step 1: Install from Community Applications

1. Go to **Apps** tab
2. Search for "Relaticle"
3. Click **Install**

#### Step 2: Configure Dependencies

Before starting Relaticle, install these dependencies:

**Install PostgreSQL:**

1. Search for "PostgreSQL" in Community Applications
2. Install the official PostgreSQL container
3. Configure:
   - Database Name: `relaticle`
   - Username: `relaticle`
   - Password: `your_secure_password`
   - Port: `5432`

**Install Redis:**

1. Search for "Redis" in Community Applications
2. Install the official Redis container
3. Use default settings (port 6379)

#### Step 3: Configure Relaticle

After dependencies are running:

1. Click Edit on Relaticle container
2. Required fields:
   - **APP_KEY**: Generate using `docker run --rm ghcr.io/relaticle/relaticle:latest php artisan key:generate --show`
   - **DB_HOST**: Container name or IP of PostgreSQL
   - **DB_PASSWORD**: Same as PostgreSQL password
   - **REDIS_HOST**: Container name or IP of Redis
   - **APP_URL**: Your full URL (e.g., `http://192.168.1.100:8080`)
3. Optional fields:
   - Configure mail settings if you want email notifications
   - Adjust log level if needed
4. Click **Apply**

#### Step 4: Access Application

Access Relaticle at: `http://your-unraid-ip:8080`

---

### Method 3: Manual Container Setup

For advanced users who want full control.

#### Step 1: Create Network

```bash
docker network create relaticle-network
```

#### Step 2: Deploy PostgreSQL

```bash
docker run -d \
  --name relaticle-postgres \
  --network relaticle-network \
  -e POSTGRES_DB=relaticle \
  -e POSTGRES_USER=relaticle \
  -e POSTGRES_PASSWORD=your_secure_password \
  -v /mnt/user/appdata/relaticle/postgres:/var/lib/postgresql/data \
  --restart unless-stopped \
  postgres:16-alpine
```

#### Step 3: Deploy Redis

```bash
docker run -d \
  --name relaticle-redis \
  --network relaticle-network \
  -v /mnt/user/appdata/relaticle/redis:/data \
  --restart unless-stopped \
  redis:7-alpine redis-server --appendonly yes
```

#### Step 4: Generate APP_KEY

```bash
docker run --rm ghcr.io/relaticle/relaticle:latest php artisan key:generate --show
```

Save this key for the next step.

#### Step 5: Deploy Relaticle App

```bash
docker run -d \
  --name relaticle-app \
  --network relaticle-network \
  -p 8080:8080 \
  -e APP_NAME=Relaticle \
  -e APP_ENV=production \
  -e APP_KEY="base64:YOUR_GENERATED_KEY_HERE" \
  -e APP_DEBUG=false \
  -e APP_URL=http://your-unraid-ip:8080 \
  -e DB_CONNECTION=pgsql \
  -e DB_HOST=relaticle-postgres \
  -e DB_PORT=5432 \
  -e DB_DATABASE=relaticle \
  -e DB_USERNAME=relaticle \
  -e DB_PASSWORD=your_secure_password \
  -e REDIS_HOST=relaticle-redis \
  -e REDIS_PORT=6379 \
  -e CACHE_STORE=redis \
  -e SESSION_DRIVER=redis \
  -e QUEUE_CONNECTION=redis \
  -e AUTORUN_ENABLED=true \
  -e AUTORUN_LARAVEL_MIGRATION=true \
  -v /mnt/user/appdata/relaticle/storage:/var/www/html/storage/app \
  --restart unless-stopped \
  ghcr.io/relaticle/relaticle:latest
```

#### Step 6: Deploy Horizon (Queue Worker)

```bash
docker run -d \
  --name relaticle-horizon \
  --network relaticle-network \
  -e APP_NAME=Relaticle \
  -e APP_ENV=production \
  -e APP_KEY="base64:YOUR_GENERATED_KEY_HERE" \
  -e DB_CONNECTION=pgsql \
  -e DB_HOST=relaticle-postgres \
  -e DB_PORT=5432 \
  -e DB_DATABASE=relaticle \
  -e DB_USERNAME=relaticle \
  -e DB_PASSWORD=your_secure_password \
  -e REDIS_HOST=relaticle-redis \
  -e QUEUE_CONNECTION=redis \
  -e AUTORUN_ENABLED=false \
  -v /mnt/user/appdata/relaticle/storage:/var/www/html/storage/app \
  --restart unless-stopped \
  ghcr.io/relaticle/relaticle:latest \
  php /var/www/html/artisan horizon
```

#### Step 7: Deploy Scheduler

```bash
docker run -d \
  --name relaticle-scheduler \
  --network relaticle-network \
  -e APP_NAME=Relaticle \
  -e APP_ENV=production \
  -e APP_KEY="base64:YOUR_GENERATED_KEY_HERE" \
  -e DB_CONNECTION=pgsql \
  -e DB_HOST=relaticle-postgres \
  -e DB_PORT=5432 \
  -e DB_DATABASE=relaticle \
  -e DB_USERNAME=relaticle \
  -e DB_PASSWORD=your_secure_password \
  -e REDIS_HOST=relaticle-redis \
  -e AUTORUN_ENABLED=false \
  -v /mnt/user/appdata/relaticle/storage:/var/www/html/storage/app \
  --restart unless-stopped \
  ghcr.io/relaticle/relaticle:latest \
  php /var/www/html/artisan schedule:work
```

---

## Initial Configuration

### First-Time Setup

1. Access Relaticle at your configured URL
2. You'll be redirected to the registration page
3. Create your admin account:
   - Name
   - Email
   - Password
4. Complete the onboarding wizard

### System Administrator Panel

Access the system admin panel at:
- `http://your-url/sysadmin` (path-based, default)
- Or configure `SYSADMIN_DOMAIN` for subdomain access

The system admin panel allows you to:
- Manage teams
- Configure system settings
- Monitor background jobs (Horizon)
- View logs and diagnostics

### Creating Teams

1. Go to System Admin Panel → Teams
2. Click "Create Team"
3. Configure team settings
4. Invite users to the team

---

## Maintenance

### Updating Relaticle

#### Docker Compose Method:

```bash
cd /mnt/user/appdata/relaticle-compose
docker compose pull
docker compose down
docker compose up -d
```

#### Manual Method:

```bash
docker pull ghcr.io/relaticle/relaticle:latest
docker stop relaticle-app relaticle-horizon relaticle-scheduler
docker rm relaticle-app relaticle-horizon relaticle-scheduler
# Re-run the docker run commands from deployment
```

### Viewing Logs

#### Docker Compose:

```bash
cd /mnt/user/appdata/relaticle-compose
docker compose logs -f app
docker compose logs -f horizon
docker compose logs -f scheduler
```

#### Manual:

```bash
docker logs -f relaticle-app
docker logs -f relaticle-horizon
docker logs -f relaticle-scheduler
```

### Database Maintenance

#### Backup Database:

```bash
docker exec relaticle-postgres pg_dump -U relaticle relaticle > backup.sql
```

#### Restore Database:

```bash
docker exec -i relaticle-postgres psql -U relaticle relaticle < backup.sql
```

### Clearing Cache

```bash
docker exec relaticle-app php artisan cache:clear
docker exec relaticle-app php artisan config:clear
docker exec relaticle-app php artisan route:clear
docker exec relaticle-app php artisan view:clear
```

---

## Troubleshooting

### Container Won't Start

**Check logs:**

```bash
docker logs relaticle-app
```

**Common issues:**

1. **Missing APP_KEY**: Generate and set `APP_KEY` in environment
2. **Database connection failed**: Verify PostgreSQL is running and credentials are correct
3. **Redis connection failed**: Verify Redis is running

### Can't Access Web Interface

1. Verify container is running: `docker ps | grep relaticle`
2. Check port mapping: Ensure port 8080 isn't used by another service
3. Check firewall: Ensure Unraid firewall allows port 8080
4. Wait for startup: First boot can take 60-90 seconds

### Database Migration Errors

Manually run migrations:

```bash
docker exec relaticle-app php artisan migrate --force
```

### Queue Jobs Not Processing

Check Horizon status:

```bash
docker exec relaticle-app php artisan horizon:status
```

Restart Horizon:

```bash
docker restart relaticle-horizon
```

### Performance Issues

1. **Increase container resources**: Allocate more RAM/CPU in Docker settings
2. **Enable Redis caching**: Verify `CACHE_STORE=redis` is set
3. **Optimize Laravel**: Run optimization commands:

```bash
docker exec relaticle-app php artisan optimize
```

### Permission Issues

Fix storage permissions:

```bash
docker exec relaticle-app chmod -R 775 storage bootstrap/cache
docker exec relaticle-app chown -R www-data:www-data storage bootstrap/cache
```

---

## Backup and Restore

### Full Backup Strategy

Back up these directories/data:

1. **Application Storage**: `/mnt/user/appdata/relaticle/storage`
2. **PostgreSQL Data**: Export database (see Database Maintenance)
3. **Redis Data**: `/mnt/user/appdata/relaticle/redis` (optional, can be regenerated)
4. **Configuration**: Your `.env` file or environment variables

### Automated Backup Script

Create `/mnt/user/appdata/relaticle/backup.sh`:

```bash
#!/bin/bash

BACKUP_DIR="/mnt/user/backups/relaticle/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

# Backup database
docker exec relaticle-postgres pg_dump -U relaticle relaticle > "$BACKUP_DIR/database.sql"

# Backup storage
cp -r /mnt/user/appdata/relaticle/storage "$BACKUP_DIR/"

# Backup .env (if using compose)
cp /mnt/user/appdata/relaticle-compose/.env "$BACKUP_DIR/"

echo "Backup completed: $BACKUP_DIR"
```

Make it executable:

```bash
chmod +x /mnt/user/appdata/relaticle/backup.sh
```

Add to Unraid User Scripts plugin to run automatically.

### Restore from Backup

1. Stop containers:
   ```bash
   docker stop relaticle-app relaticle-horizon relaticle-scheduler
   ```

2. Restore database:
   ```bash
   docker exec -i relaticle-postgres psql -U relaticle relaticle < backup/database.sql
   ```

3. Restore storage:
   ```bash
   cp -r backup/storage/* /mnt/user/appdata/relaticle/storage/
   ```

4. Start containers:
   ```bash
   docker start relaticle-app relaticle-horizon relaticle-scheduler
   ```

---

## Support and Resources

- **Documentation**: https://relaticle.com/documentation
- **GitHub Issues**: https://github.com/Relaticle/relaticle/issues
- **Community Discussions**: https://github.com/Relaticle/relaticle/discussions
- **Discord**: https://discord.gg/b9WxzUce4Q

---

## Security Recommendations

1. **Change default passwords**: Use strong, unique passwords for database
2. **Use HTTPS**: Configure reverse proxy (e.g., SWAG, Nginx Proxy Manager)
3. **Regular updates**: Keep Relaticle and dependencies updated
4. **Backup regularly**: Implement automated backup strategy
5. **Limit access**: Use firewall rules or VPN for external access
6. **Enable 2FA**: Configure two-factor authentication for admin accounts

---

## Advanced Configuration

### Using with Reverse Proxy

If using a reverse proxy (SWAG, Nginx Proxy Manager):

1. Set `APP_URL` to your full domain: `https://crm.yourdomain.com`
2. Configure proxy to forward to `http://unraid-ip:8080`
3. Enable WebSocket support for real-time features

### Custom Domain Setup

1. Update `APP_URL` environment variable
2. Configure DNS to point to your server
3. Set up SSL certificate (Let's Encrypt recommended)
4. Update CORS settings if needed

### Email Configuration

For production use, configure SMTP:

```env
MAIL_MAILER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=your-email@gmail.com
MAIL_PASSWORD=your-app-password
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=noreply@yourdomain.com
MAIL_FROM_NAME=Relaticle CRM
```

### OAuth Configuration

To enable Google or GitHub login:

```env
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
GOOGLE_REDIRECT_URI=https://your-domain.com/auth/google/callback

GITHUB_CLIENT_ID=your-github-client-id
GITHUB_CLIENT_SECRET=your-github-client-secret
```

---

## For Future AI Sessions

### Key Technical Details

**Architecture:**
- Laravel 12.x application with Filament 4 admin panel
- Uses serversideup/php:8.4-fpm-nginx as base image
- Multi-stage Docker build: composer dependencies → frontend build → production image
- Separate containers for web, queue worker (Horizon), and scheduler

**Important Environment Variables:**
- `AUTORUN_ENABLED=true`: Enables automatic Laravel setup on container start
- `AUTORUN_LARAVEL_MIGRATION=true`: Runs database migrations automatically
- `AUTORUN_LARAVEL_MIGRATION_ISOLATION=true`: Uses migration isolation
- `AUTORUN_LARAVEL_STORAGE_LINK=true`: Creates storage symlink

**Port Configuration:**
- Application listens on port 8080 internally (nginx)
- Map to any external port via Docker

**Volume Mounts:**
- `/var/www/html/storage/app`: Persistent storage for uploads and files
- Must be writable by `www-data` user (UID 33)

**Health Checks:**
- App: `curl -f http://localhost:8080/up`
- Horizon: `php artisan horizon:status`
- PostgreSQL: `pg_isready`
- Redis: `redis-cli ping`

**Database:**
- PostgreSQL 16+ required (not MySQL)
- Runs migrations automatically on first start
- Uses UTF-8 encoding

**Queue System:**
- Laravel Horizon for queue management
- Redis as queue backend
- Web interface at `/sysadmin/horizon`

**Caching:**
- Redis for session, cache, and queues
- Config/route/view caching enabled in production

**Common Commands:**
- Generate key: `docker run --rm ghcr.io/relaticle/relaticle:latest php artisan key:generate --show`
- Run migrations: `docker exec relaticle-app php artisan migrate`
- Clear cache: `docker exec relaticle-app php artisan cache:clear`
- Access console: `docker exec -it relaticle-app bash`

**Troubleshooting Patterns:**
- If migrations fail: Check database connection and credentials
- If assets missing: Verify frontend build stage completed successfully
- If permissions errors: Ensure volumes owned by www-data (UID 33)
- If slow performance: Check Redis connection and cache configuration

**Update Process:**
1. Pull new image
2. Stop containers
3. Start containers (migrations run automatically)
4. Clear cache if needed

---

*Last Updated: December 2025*
*Relaticle Version: Latest*
