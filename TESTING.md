# Testing Guide for Relaticle Docker Deployment

This guide covers how to test all the Docker deployment changes.

---

## Prerequisites

- Docker 20.10+ and Docker Compose 2.0+
- 4GB RAM minimum
- Linux, macOS, or Windows with WSL2

---

## Test 1: Docker Build (Quick Test - 5 minutes)

Test that the Dockerfile builds correctly:

```bash
# Clone the repository
git clone https://github.com/sudoBrandino/relaticle.git
cd relaticle

# Switch to the feature branch
git checkout claude/docker-deployment-github-actions-GY6uv

# Build the Docker image
docker build -t relaticle-test:latest .

# Verify build succeeded
docker images relaticle-test:latest

# Test PHP version
docker run --rm relaticle-test:latest php --version
# Should output: PHP 8.4.x

# Test Artisan
docker run --rm relaticle-test:latest php artisan --version
# Should output: Laravel Framework 12.x
```

**Expected result:** Build completes without errors, image is ~500MB-800MB

---

## Test 2: Docker Compose Production Stack (Full Test - 15 minutes)

Test the complete production deployment:

```bash
cd relaticle

# Create environment file
cp .env.example .env.prod

# Generate APP_KEY
APP_KEY=$(docker run --rm ghcr.io/relaticle/relaticle:latest php artisan key:generate --show)
echo "Generated APP_KEY: $APP_KEY"

# Edit .env.prod with required values
cat > .env.prod << 'EOF'
APP_NAME=Relaticle
APP_ENV=production
APP_KEY=PASTE_YOUR_KEY_HERE
APP_DEBUG=false
APP_URL=http://localhost:8080

DB_DATABASE=relaticle
DB_USERNAME=relaticle
DB_PASSWORD=super_secret_password_change_me

REDIS_PASSWORD=null

LOG_CHANNEL=stderr
LOG_LEVEL=info
EOF

# Replace PASTE_YOUR_KEY_HERE with the actual key
sed -i "s|PASTE_YOUR_KEY_HERE|$APP_KEY|" .env.prod

# Start the stack
docker compose -f docker-compose.prod.yml --env-file .env.prod up -d

# Wait for services to be healthy (60-90 seconds)
echo "Waiting for services to start..."
sleep 90

# Check service status
docker compose -f docker-compose.prod.yml ps

# Check logs
docker compose -f docker-compose.prod.yml logs app | tail -50

# Test the application
curl -I http://localhost:8080

# Check Horizon status
docker compose -f docker-compose.prod.yml exec app php artisan horizon:status

# View database tables
docker compose -f docker-compose.prod.yml exec postgres psql -U relaticle -d relaticle -c "\dt"
```

**Expected results:**
- All containers running and healthy
- `curl` returns HTTP 200 or 302
- Horizon shows "running"
- Database has ~30+ tables

**Access the application:**
Open browser to: http://localhost:8080

You should see the Relaticle welcome/login page.

**Cleanup:**
```bash
docker compose -f docker-compose.prod.yml down -v
```

---

## Test 3: Unraid Docker Compose Stack (15 minutes)

Test the Unraid-specific deployment:

```bash
cd relaticle/unraid

# Copy environment template
cp .env.unraid .env

# Generate APP_KEY
APP_KEY=$(docker run --rm ghcr.io/relaticle/relaticle:latest php artisan key:generate --show)

# Edit .env and set required values
# At minimum, set:
# - APP_KEY
# - DB_PASSWORD
# - APP_URL (use your machine's IP)

nano .env  # or vim, or any editor

# Start the stack
docker compose -f docker-compose-unraid.yml up -d

# Wait for startup
sleep 90

# Check status
docker compose -f docker-compose-unraid.yml ps

# Test access
curl -I http://localhost:8080

# Check logs
docker compose -f docker-compose-unraid.yml logs -f app
```

**Cleanup:**
```bash
docker compose -f docker-compose-unraid.yml down -v
```

---

## Test 4: Multi-Architecture Build (Advanced - 20 minutes)

Test building for multiple architectures (amd64 and arm64):

```bash
# Set up buildx
docker buildx create --name multiarch --use
docker buildx inspect --bootstrap

# Build for multiple platforms (doesn't push)
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --target production \
  -t relaticle-multiarch:test \
  .

# Check the build completed
docker buildx imagetools inspect relaticle-multiarch:test
```

---

## Test 5: GitHub Actions Workflow Validation (Local - 5 minutes)

Test workflow syntax without running on GitHub:

```bash
# Install act (GitHub Actions local runner)
# macOS: brew install act
# Linux: curl https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash

# Validate workflow syntax
act -l

# Run the tests workflow (requires large runner)
act push -W .github/workflows/tests.yml --container-architecture linux/amd64

# Or just validate without running
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/tests.yml'))" && echo "✓ tests.yml valid"
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/docker-publish.yml'))" && echo "✓ docker-publish.yml valid"
```

---

## Test 6: Unraid XML Template Validation (2 minutes)

Validate the Unraid template:

```bash
# Validate XML syntax
python3 -c "import xml.etree.ElementTree as ET; ET.parse('unraid/relaticle.xml')" && echo "✓ XML valid"

# Check required fields
grep -E "<Name>|<Repository>|<WebUI>|<Icon>" unraid/relaticle.xml

# Validate environment variables are documented
grep -c "<Config Name=" unraid/relaticle.xml
# Should show 30+ configurations
```

---

## Test 7: Health Checks (5 minutes)

With the stack running from Test 2 or 3:

```bash
# Test app health endpoint
curl http://localhost:8080/up
# Should return "OK" or redirect

# Test Horizon health
docker exec relaticle-app php artisan horizon:status
# Should show "running"

# Test database connection
docker exec relaticle-app php artisan tinker --execute="echo DB::connection()->getPdo() ? 'Connected' : 'Failed';"
# Should output "Connected"

# Test Redis connection
docker exec relaticle-app php artisan tinker --execute="echo Redis::ping() ? 'Connected' : 'Failed';"
# Should output "Connected"

# Check PostgreSQL health
docker exec relaticle-postgres pg_isready -U relaticle
# Should output "accepting connections"

# Check Redis health
docker exec relaticle-redis redis-cli ping
# Should output "PONG"
```

---

## Test 8: First-Time Setup Flow (10 minutes)

Test the complete user experience:

```bash
# Start fresh stack (from Test 2 or 3)
docker compose -f docker-compose.prod.yml up -d

# Wait for migrations to complete
sleep 90

# Open browser to http://localhost:8080
# You should be redirected to registration page

# Create admin account through the web UI:
# - Name: Test Admin
# - Email: admin@example.com
# - Password: Choose a strong password

# After registration, verify access to:
# - Dashboard: http://localhost:8080
# - System Admin: http://localhost:8080/sysadmin
# - Horizon: http://localhost:8080/sysadmin/horizon

# Test creating a team:
# - Go to System Admin → Teams
# - Click Create Team
# - Fill in team details
# - Save

# Test CRM features:
# - Switch to your team (top-right dropdown)
# - Create a Company
# - Create a Contact
# - Create an Opportunity
```

---

## Test 9: Update/Migration Test (10 minutes)

Test the update process:

```bash
# With stack running
docker compose -f docker-compose.prod.yml ps

# Pull latest images (simulating an update)
docker compose -f docker-compose.prod.yml pull

# Restart with new images
docker compose -f docker-compose.prod.yml up -d

# Check that migrations run automatically
docker compose -f docker-compose.prod.yml logs app | grep -i migration

# Verify application still works
curl http://localhost:8080/up
```

---

## Test 10: Backup and Restore (15 minutes)

Test backup procedures:

```bash
# Backup database
docker exec relaticle-postgres pg_dump -U relaticle relaticle > backup_$(date +%Y%m%d).sql
echo "Database backup created: backup_$(date +%Y%m%d).sql"

# Backup storage
docker cp relaticle-app:/var/www/html/storage/app ./storage_backup/

# Simulate disaster: destroy everything
docker compose -f docker-compose.prod.yml down -v

# Restore: start fresh stack
docker compose -f docker-compose.prod.yml up -d
sleep 90

# Restore database
cat backup_$(date +%Y%m%d).sql | docker exec -i relaticle-postgres psql -U relaticle relaticle

# Restore storage
docker cp ./storage_backup/. relaticle-app:/var/www/html/storage/app/

# Verify restoration
curl http://localhost:8080
# Should show your data
```

---

## Test 11: Performance Test (Optional - 10 minutes)

Basic load testing:

```bash
# Install Apache Bench (if not installed)
# Ubuntu: sudo apt-get install apache2-utils
# macOS: Already installed

# Run simple load test
ab -n 1000 -c 10 http://localhost:8080/

# Check response times and success rate
# Look for:
# - Requests per second
# - Time per request
# - Failed requests (should be 0)

# Monitor container resources
docker stats --no-stream
```

---

## Troubleshooting Tests

### Issue: Container won't start

```bash
# Check logs
docker compose -f docker-compose.prod.yml logs app

# Check for common issues
docker exec relaticle-app php artisan config:clear
docker exec relaticle-app php artisan cache:clear
```

### Issue: Database connection failed

```bash
# Verify PostgreSQL is running
docker compose -f docker-compose.prod.yml ps postgres

# Check database logs
docker compose -f docker-compose.prod.yml logs postgres

# Test connection
docker exec relaticle-postgres psql -U relaticle -d relaticle -c "SELECT 1;"
```

### Issue: 500 error on web page

```bash
# Check Laravel logs
docker exec relaticle-app cat storage/logs/laravel.log | tail -50

# Verify APP_KEY is set
docker exec relaticle-app php artisan tinker --execute="echo config('app.key') ? 'Set' : 'Not set';"

# Clear caches
docker exec relaticle-app php artisan optimize:clear
```

---

## Success Criteria Checklist

After testing, you should have verified:

- [ ] Docker image builds successfully
- [ ] All containers start and become healthy
- [ ] Web UI is accessible
- [ ] Can create admin account
- [ ] Can create teams
- [ ] Can create CRM records (companies, contacts, opportunities)
- [ ] Horizon is running and processing jobs
- [ ] Scheduler is running
- [ ] Health checks pass
- [ ] Database migrations run automatically
- [ ] Logs are visible and showing no errors
- [ ] Backup and restore works
- [ ] GitHub Actions workflows pass

---

## CI/CD Testing (GitHub Actions)

The GitHub Actions workflows are tested automatically on push:

**Check status:**
- Visit: https://github.com/sudoBrandino/relaticle/actions
- Look for green checkmarks ✅

**What's tested in CI:**
- ✅ PHP 8.4 compatibility
- ✅ Code style (PHP CS Fixer)
- ✅ Static analysis (PHPStan)
- ✅ Refactor checks (Rector)
- ✅ Type coverage
- ✅ 7 parallel test shards
- ✅ Docker multi-arch build (amd64/arm64)
- ✅ GHCR push (on main repo)

---

## Need Help?

If any tests fail:

1. Check the specific error message
2. Review the troubleshooting section above
3. Check the logs: `docker compose -f <compose-file> logs`
4. Open an issue on GitHub with:
   - What test failed
   - Error message
   - Your environment (OS, Docker version)

---

**Last Updated:** December 2025
