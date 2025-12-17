# Relaticle CRM - Project Context for AI Assistants

This document provides comprehensive context about the Relaticle CRM project for AI assistants working on this codebase.

**Last Updated**: December 2025
**Project Version**: Laravel 12.x, PHP 8.4, Filament 4.x

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Technology Stack](#technology-stack)
3. [Project Structure](#project-structure)
4. [Docker & Deployment](#docker--deployment)
5. [GitHub Actions & CI/CD](#github-actions--cicd)
6. [Development Workflow](#development-workflow)
7. [Testing Strategy](#testing-strategy)
8. [Common Commands](#common-commands)
9. [Troubleshooting](#troubleshooting)
10. [Important Patterns & Conventions](#important-patterns--conventions)

---

## Project Overview

**Relaticle** is a modern, self-hosted, open-source CRM platform designed for teams who've outgrown spreadsheets but find Salesforce overkill.

### Key Features
- Fully customizable with no-code custom fields
- Multi-team support with isolated workspaces
- Contact, company, and opportunity management
- Task and note tracking
- Built on Laravel 12 with Filament 4 admin panel
- Self-hosted with complete data ownership
- AGPL-3.0 licensed

### Repository
- **GitHub**: https://github.com/Relaticle/relaticle
- **Website**: https://relaticle.com
- **Main Branch**: `main`
- **License**: AGPL-3.0

---

## Technology Stack

### Backend
- **Framework**: Laravel 12.x
- **PHP Version**: 8.4+
- **Database**: PostgreSQL 15+ (NOT MySQL - this is important!)
- **Cache/Queue**: Redis 7+
- **Queue Management**: Laravel Horizon
- **Admin Panel**: Filament 4.x
- **API**: Laravel API resources

### Frontend
- **Build Tool**: Vite
- **Framework**: Livewire 3
- **CSS**: Tailwind CSS
- **UI Components**: Filament components

### DevOps
- **Containerization**: Docker (multi-stage builds)
- **Base Image**: serversideup/php:8.4-fpm-nginx
- **CI/CD**: GitHub Actions
- **Container Registry**: GHCR (ghcr.io) and Docker Hub
- **Multi-arch**: amd64 and arm64 support

---

## Project Structure

```
relaticle/
├── app/                          # Laravel application code
│   ├── Filament/                # Filament admin panels
│   │   ├── App/                # Main app panel (team-specific)
│   │   └── SysAdmin/           # System admin panel
│   ├── Http/                   # Controllers, middleware, requests
│   ├── Models/                 # Eloquent models
│   └── Providers/              # Service providers
├── app-modules/                 # Modular application structure
│   ├── OnboardSeed/            # Onboarding data seeders
│   └── [other modules]/        # Feature modules
├── database/                    # Migrations, seeders, factories
│   ├── factories/              # Model factories for testing
│   ├── migrations/             # Database migrations
│   └── seeders/                # Database seeders
├── resources/                   # Frontend resources
│   ├── css/                    # Stylesheets
│   ├── js/                     # JavaScript files
│   └── views/                  # Blade templates
├── routes/                      # Route definitions
│   ├── api.php                 # API routes
│   ├── console.php             # Console commands
│   └── web.php                 # Web routes
├── tests/                       # Test suite
│   ├── Feature/                # Feature tests
│   └── Unit/                   # Unit tests
├── .github/                     # GitHub configuration
│   └── workflows/              # GitHub Actions workflows
├── unraid/                      # Unraid deployment files
│   ├── UNRAID-DEPLOYMENT.md    # Unraid deployment guide
│   ├── docker-compose-unraid.yml # Docker Compose for Unraid
│   ├── relaticle.xml           # Unraid template
│   └── .env.unraid             # Unraid environment example
├── Dockerfile                   # Multi-stage Docker build
├── docker-compose.yml          # Development compose file (Laravel Sail)
├── docker-compose.prod.yml     # Production compose file
├── composer.json               # PHP dependencies
├── package.json                # Node.js dependencies
├── phpunit.xml                 # PHPUnit configuration
└── vite.config.js              # Vite build configuration
```

### Important Directories

- **`app/Filament/`**: Contains all Filament panel configurations. The app has two panels:
  - `App/`: Main CRM panel (team-specific, multi-tenant)
  - `SysAdmin/`: System administration panel (accessed via `/sysadmin`)

- **`app-modules/`**: Modular architecture for features. Each module can have its own routes, views, migrations, etc.

- **`unraid/`**: Complete Unraid deployment package including templates, documentation, and compose files

---

## Docker & Deployment

### Dockerfile Architecture

The Dockerfile uses a **multi-stage build** with three stages:

1. **Stage 1: Composer** (`composer:2`)
   - Installs PHP dependencies
   - Uses `--no-dev` for production
   - Uses `--ignore-platform-reqs` to avoid platform conflicts

2. **Stage 2: Frontend** (`node:22-alpine`)
   - Installs NPM dependencies
   - Builds frontend assets with Vite
   - Requires vendor directory from Stage 1 for Filament CSS compilation

3. **Stage 3: Production** (`serversideup/php:8.4-fpm-nginx`)
   - Final production image
   - Copies application code
   - Copies vendor from Stage 1
   - Copies built assets from Stage 2
   - Installs PostgreSQL client for health checks
   - Sets up Laravel automations

### Environment Variables

**Critical for Deployment:**

```bash
# Application
APP_KEY=base64:...          # REQUIRED - Generate with php artisan key:generate
APP_ENV=production          # Environment
APP_DEBUG=false            # Debug mode (false for prod)
APP_URL=http://localhost   # Full application URL

# Database (PostgreSQL only!)
DB_CONNECTION=pgsql        # Must be pgsql
DB_HOST=postgres           # Database host
DB_PORT=5432              # Database port
DB_DATABASE=relaticle     # Database name
DB_USERNAME=relaticle     # Database user
DB_PASSWORD=secret        # Database password

# Redis
REDIS_HOST=redis          # Redis host
REDIS_PORT=6379           # Redis port
REDIS_PASSWORD=null       # Redis password (optional)
CACHE_STORE=redis         # Use Redis for cache
SESSION_DRIVER=redis      # Use Redis for sessions
QUEUE_CONNECTION=redis    # Use Redis for queues

# Laravel Automations (serversideup/php features)
AUTORUN_ENABLED=true                      # Enable automations
AUTORUN_LARAVEL_STORAGE_LINK=true        # Create storage link
AUTORUN_LARAVEL_MIGRATION=true           # Run migrations
AUTORUN_LARAVEL_MIGRATION_ISOLATION=true # Use migration locks
AUTORUN_LARAVEL_CONFIG_CACHE=true        # Cache config
AUTORUN_LARAVEL_ROUTE_CACHE=true         # Cache routes
AUTORUN_LARAVEL_VIEW_CACHE=true          # Cache views
```

### Docker Compose Configurations

1. **`docker-compose.yml`**: Development environment using Laravel Sail
   - MySQL for compatibility with Sail
   - Includes dev tools (Redis, Meilisearch, Mailpit, Selenium)
   - Hot module reloading with Vite

2. **`docker-compose.prod.yml`**: Production deployment
   - PostgreSQL database
   - Redis for cache/sessions/queues
   - Separate containers for:
     - `app`: Main web application
     - `horizon`: Queue worker
     - `scheduler`: Laravel scheduler
   - Health checks on all services
   - Persistent volumes for data

3. **`unraid/docker-compose-unraid.yml`**: Unraid-optimized stack
   - Same as prod but with Unraid-specific paths
   - Uses `/mnt/user/appdata/relaticle` for data
   - Environment variable driven configuration
   - Includes comprehensive comments

### Health Checks

- **App**: `curl -f http://localhost:8080/up`
- **Horizon**: `php artisan horizon:status`
- **PostgreSQL**: `pg_isready -U relaticle -d relaticle`
- **Redis**: `redis-cli ping`

### Volumes

- `/var/www/html/storage/app`: Persistent storage for uploads and files
- Owned by `www-data` user (UID 33, GID 33)

---

## GitHub Actions & CI/CD

### Workflows

Located in `.github/workflows/`:

#### 1. **`tests.yml`** - Test Suite
- **Triggers**: Every push
- **Jobs**:
  - `lint-and-static-analysis`: Runs linting, refactor checks, type coverage, and static analysis
  - `tests`: Runs PHPUnit/Pest tests across 7 shards for parallel execution
- **Services**: PostgreSQL for testing
- **PHP Version**: 8.4
- **Node Version**: 22.x
- **Test Sharding**: 7 parallel shards for faster execution
- **Caching**: Composer and NPM caches

**Recent Fix (December 2025):**
- Fixed cache reference bug on line 130: Changed `steps.cache.outputs.cache-hit` to `steps.npm-cache.outputs.cache-hit`

#### 2. **`docker-publish.yml`** - Docker Image Build & Publish
- **Triggers**:
  - Push to `main` branch
  - Git tags matching `v*`
  - Pull requests to `main`
- **Multi-architecture**: Builds for linux/amd64 and linux/arm64
- **Strategy**: Matrix build with native runners
  - amd64: `ubuntu-latest`
  - arm64: `ubuntu-24.04-arm`
- **Registries**:
  - GHCR: `ghcr.io/relaticle/relaticle`
  - Docker Hub: `manukminasyan/relaticle`
- **Build Process**:
  1. Parallel builds for each architecture
  2. Push by digest
  3. Merge stage creates multi-arch manifest
  4. Push to both registries
- **Tags**:
  - `main`: Branch name
  - `v*`: Semantic versions (e.g., `v1.0.0`, `1.0`)
  - `sha-<hash>`: Git commit SHA
  - `latest`: Only on version tags
- **Caching**: GitHub Actions cache for build layers

#### 3. **`filament-view-monitor-simple.yml`** - Filament View Monitoring
- **Triggers**:
  - Daily at 9 AM UTC
  - Manual trigger
- **Purpose**: Monitors published Filament views for upstream changes
- **Files Monitored**:
  - `components/layout/index.blade.php`
  - `livewire/topbar.blade.php`
- **Action**: Creates GitHub issue when changes detected

### Secrets Required

Configure these in GitHub repository settings:

- `GITHUB_TOKEN`: Automatically provided by GitHub
- `DOCKERHUB_USERNAME`: Docker Hub username
- `DOCKERHUB_TOKEN`: Docker Hub access token

### Build Caching

Both workflows use aggressive caching:
- Composer dependencies: `~/.composer/cache/files`
- NPM dependencies: NPM cache directory
- Docker layers: GitHub Actions cache with scope per platform

---

## Development Workflow

### Initial Setup

```bash
# Clone repository
git clone https://github.com/Relaticle/relaticle.git
cd relaticle

# Install dependencies and setup
composer app-install
```

The `app-install` script (defined in `composer.json`) runs:
1. `composer install`
2. Copies `.env.example` to `.env`
3. Generates `APP_KEY`
4. Runs migrations
5. Installs NPM dependencies
6. Seeds database with demo data

### Development Commands

```bash
# Start development server (includes queue worker and Vite)
composer dev

# Run tests
composer test

# Run specific test suites
composer test:pest              # Run all Pest tests
composer test:pest:coverage    # Run with coverage
composer test:pest:ci          # CI mode
composer test:pest:ci:shard    # Sharded tests (use SHARD env var)

# Code quality
composer test:lint             # PHP CS Fixer (fix code style)
composer test:refactor         # Rector (check refactoring opportunities)
composer test:types            # PHPStan (static analysis)
composer test:type-coverage    # Type coverage analysis

# Laravel commands
php artisan migrate            # Run migrations
php artisan db:seed            # Seed database
php artisan horizon            # Start Horizon
php artisan schedule:work      # Start scheduler
php artisan tinker             # Laravel REPL
```

### Git Workflow

1. **Feature branches**: Create from `main`
2. **Naming**: `feature/description` or `fix/description`
3. **Commits**: Clear, descriptive commit messages
4. **Pull requests**:
   - Target `main` branch
   - CI must pass (tests + linting + Docker build)
   - Require review

---

## Testing Strategy

### Test Framework

- **Primary**: Pest PHP (modern PHPUnit wrapper)
- **Location**: `tests/` directory
- **Structure**:
  - `Feature/`: Integration and feature tests
  - `Unit/`: Unit tests

### Database Testing

- Uses separate PostgreSQL database for testing
- Configuration in `.env.testing`
- Database name: `relaticle_testing`
- Migrations run before tests
- Uses `RefreshDatabase` trait

### Test Execution

**Local:**
```bash
composer test
```

**CI (Sharded):**
```bash
SHARD=1/7 composer test:pest:ci:shard
```

### Coverage

```bash
composer test:pest:coverage
```

Generates HTML coverage report in `coverage/` directory.

---

## Common Commands

### Artisan Commands

```bash
# Application
php artisan key:generate              # Generate APP_KEY
php artisan optimize                  # Optimize application
php artisan optimize:clear           # Clear optimization caches

# Database
php artisan migrate                   # Run migrations
php artisan migrate:fresh --seed     # Fresh database with seeds
php artisan db:seed                  # Run seeders
php artisan migrate:rollback         # Rollback last migration

# Cache
php artisan cache:clear              # Clear application cache
php artisan config:clear             # Clear config cache
php artisan route:clear              # Clear route cache
php artisan view:clear               # Clear view cache

# Queue & Horizon
php artisan horizon                  # Start Horizon
php artisan horizon:pause            # Pause Horizon
php artisan horizon:continue         # Resume Horizon
php artisan queue:work               # Start queue worker
php artisan queue:restart            # Restart queue workers

# Storage
php artisan storage:link             # Create storage symlink

# Filament
php artisan filament:upgrade         # Upgrade Filament assets
php artisan make:filament-user       # Create Filament user
```

### Docker Commands

```bash
# Build image
docker build -t relaticle:latest .

# Run container
docker run -p 8080:8080 \
  -e APP_KEY=base64:... \
  -e DB_HOST=postgres \
  -e DB_PASSWORD=secret \
  ghcr.io/relaticle/relaticle:latest

# Development with compose
docker compose up -d

# Production with compose
docker compose -f docker-compose.prod.yml up -d

# View logs
docker compose logs -f app
docker compose logs -f horizon

# Execute command in container
docker exec -it relaticle-app bash
docker exec relaticle-app php artisan migrate
```

### Composer Scripts

All available in `composer.json`:

```bash
composer app-install           # Initial setup
composer dev                   # Start dev server
composer test                  # Run all tests
composer test:lint             # Fix code style
composer test:refactor         # Check refactoring
composer test:types            # Static analysis
composer test:type-coverage    # Type coverage
```

---

## Troubleshooting

### Common Issues

#### 1. "APP_KEY is not set"

**Solution:**
```bash
php artisan key:generate
```

Or for Docker:
```bash
docker run --rm ghcr.io/relaticle/relaticle:latest php artisan key:generate --show
```

#### 2. Database Connection Failed

**Check:**
- PostgreSQL is running
- Credentials are correct in `.env`
- Host is accessible (use `postgres` for Docker Compose)
- Database exists

**Test connection:**
```bash
php artisan tinker
DB::connection()->getPdo();
```

#### 3. Redis Connection Failed

**Check:**
- Redis is running
- Host is correct (use `redis` for Docker Compose)
- Password matches (if set)

**Test connection:**
```bash
php artisan tinker
Redis::ping();
```

#### 4. Frontend Assets Not Loading

**Solution:**
```bash
npm run build                    # Production
npm run dev                      # Development with HMR
```

In Docker, ensure frontend stage completed:
```bash
docker build --target frontend -t relaticle:frontend .
```

#### 5. Permissions Errors (Docker)

**Fix storage permissions:**
```bash
docker exec relaticle-app chmod -R 775 storage bootstrap/cache
docker exec relaticle-app chown -R www-data:www-data storage bootstrap/cache
```

#### 6. Queue Jobs Not Processing

**Check Horizon:**
```bash
php artisan horizon:status
```

**Restart Horizon:**
```bash
php artisan horizon:terminate
php artisan horizon
```

In Docker:
```bash
docker restart relaticle-horizon
```

#### 7. GitHub Actions Failing

**Common causes:**
- Cache corruption: Clear GitHub Actions cache
- Dependency version conflicts: Check `composer.lock` and `package-lock.json`
- Test failures: Run locally first
- Docker build issues: Test build locally

**Check workflow logs:**
1. Go to Actions tab on GitHub
2. Click failed workflow
3. Expand failed step
4. Check error messages

---

## Important Patterns & Conventions

### Code Style

- **PSR-12**: PHP coding standard
- **PHP CS Fixer**: Automatic code formatting
- **Rector**: PHP refactoring tool
- **PHPStan**: Static analysis (Level 5)

Run before committing:
```bash
composer test:lint
composer test:refactor
composer test:types
```

### Database Conventions

- **PostgreSQL only**: Do not use MySQL-specific features
- **Migrations**: Use Laravel schema builder, avoid raw SQL when possible
- **Naming**:
  - Tables: plural, snake_case (e.g., `companies`, `contact_person`)
  - Columns: snake_case
  - Foreign keys: `model_id` (e.g., `company_id`)
  - Pivot tables: alphabetical order (e.g., `company_person`)

### Filament Patterns

- **Resources**: Located in `app/Filament/App/Resources/` or `app/Filament/SysAdmin/Resources/`
- **Pages**: Custom pages in Resources or standalone
- **Widgets**: Dashboard widgets
- **Multi-tenancy**: Uses team-based tenancy
- **Authorization**: Policies for resource access

### Model Conventions

- **Traits**: Use appropriate traits (`SoftDeletes`, `HasFactory`, etc.)
- **Relationships**: Define all relationships
- **Scopes**: Global and local scopes for common queries
- **Attributes**: Use Laravel 9+ attributes for casts and accessors

### Testing Patterns

- **Pest syntax**: Use Pest's modern syntax
- **Factories**: Create factories for all models
- **Seeders**: Separate seeders for different data sets
- **Assertions**: Use Pest's expressive assertions

Example:
```php
it('creates a company', function () {
    $company = Company::factory()->create();

    expect($company)
        ->toBeInstanceOf(Company::class)
        ->name->not->toBeEmpty();
});
```

### Environment Configuration

- **Never commit** `.env` files
- **Document** all env vars in `.env.example`
- **Use defaults**: Provide sensible defaults in `config/` files
- **Validate**: Use `config()` helper, not `env()` in application code

### Security Best Practices

- **Input validation**: Always validate user input
- **Authorization**: Use policies and gates
- **CSRF protection**: Enabled by default
- **SQL injection**: Use query builder or Eloquent (never raw SQL with user input)
- **XSS protection**: Blade auto-escapes, use `{!! !!}` carefully
- **Mass assignment**: Define `$fillable` or `$guarded` on models

---

## For AI Assistants: Quick Reference

### When Working on Features

1. **Check conventions** in this document
2. **Run tests** before committing: `composer test`
3. **Fix code style**: `composer test:lint`
4. **Check types**: `composer test:types`
5. **Update docs** if adding new features

### When Troubleshooting

1. **Check logs**: `storage/logs/laravel.log`
2. **Check Docker logs**: `docker compose logs`
3. **Check database**: Use `tinker` or database client
4. **Check Redis**: `php artisan tinker` → `Redis::ping()`
5. **Clear caches**: `php artisan optimize:clear`

### When Modifying Docker

1. **Test build locally** before pushing
2. **Use multi-stage build** pattern
3. **Keep image size small**
4. **Test both architectures** (amd64/arm64)
5. **Update docker-compose files** if needed

### When Modifying GitHub Actions

1. **Test changes** on a feature branch first
2. **Check syntax**: Use GitHub Actions validator
3. **Monitor costs**: Be aware of runner minutes
4. **Use caching**: Leverage caching for dependencies
5. **Document changes** in workflow comments

### Key Files to Remember

- `composer.json`: PHP deps & scripts
- `package.json`: Node deps & scripts
- `Dockerfile`: Multi-stage build
- `docker-compose.prod.yml`: Production stack
- `.github/workflows/`: CI/CD pipelines
- `.env.example`: Environment documentation
- `phpunit.xml`: Test configuration
- `config/`: Application configuration

---

## Maintenance Notes

### Regular Tasks

- **Update dependencies**: Monthly security updates
- **Database backups**: Automated daily backups
- **Log rotation**: Ensure logs don't fill disk
- **Monitor Horizon**: Check for failed jobs
- **Cache clearing**: Clear old cache periodically

### Upgrade Checklist

When upgrading major versions:

1. Read Laravel upgrade guide
2. Read Filament upgrade guide
3. Update PHP version if needed
4. Update dependencies
5. Run tests
6. Test migrations
7. Update Docker base image
8. Update documentation
9. Tag release

---

## Additional Resources

- **Laravel Docs**: https://laravel.com/docs/12.x
- **Filament Docs**: https://filamentphp.com/docs/4.x
- **Livewire Docs**: https://livewire.laravel.com/docs
- **Pest Docs**: https://pestphp.com/docs
- **Docker Best Practices**: https://docs.docker.com/develop/dev-best-practices/

---

**Last Updated**: December 17, 2025
**Maintainer**: Relaticle Team
**For Issues**: https://github.com/Relaticle/relaticle/issues
