# Relaticle CRM - Unraid Deployment Files

This directory contains everything you need to deploy Relaticle CRM on Unraid.

## Quick Start

**Fastest method using Docker Compose:**

1. Install Compose Manager plugin on Unraid
2. Download files:
   ```bash
   mkdir -p /mnt/user/appdata/relaticle-compose
   cd /mnt/user/appdata/relaticle-compose
   wget https://raw.githubusercontent.com/Relaticle/relaticle/main/unraid/docker-compose-unraid.yml
   wget https://raw.githubusercontent.com/Relaticle/relaticle/main/unraid/.env.unraid -O .env
   ```
3. Generate APP_KEY:
   ```bash
   docker run --rm ghcr.io/relaticle/relaticle:latest php artisan key:generate --show
   ```
4. Edit `.env` file and paste your APP_KEY
5. Deploy:
   ```bash
   docker compose -f docker-compose-unraid.yml up -d
   ```
6. Access: `http://your-unraid-ip:8080`

## Files in This Directory

- **`UNRAID-DEPLOYMENT.md`** - Complete deployment guide with all methods
- **`docker-compose-unraid.yml`** - Docker Compose stack (recommended)
- **`relaticle.xml`** - Community Applications template
- **`.env.unraid`** - Example environment configuration
- **`README.md`** - This file

## Deployment Methods

### 1. Docker Compose Stack (Recommended)

Complete all-in-one deployment with PostgreSQL, Redis, and all Relaticle services.

**Pros:**
- Single command deployment
- Automatic dependency management
- Easy updates
- Includes all required services

**See:** [UNRAID-DEPLOYMENT.md](UNRAID-DEPLOYMENT.md#method-1-docker-compose-stack-recommended)

### 2. Community Applications Template

Use the Unraid template to deploy via the Apps tab.

**Pros:**
- Familiar Unraid UI
- No command line needed
- Easy configuration

**Cons:**
- Requires manual dependency setup (PostgreSQL, Redis)
- More steps than Docker Compose

**See:** [UNRAID-DEPLOYMENT.md](UNRAID-DEPLOYMENT.md#method-2-community-applications-template)

### 3. Manual Container Setup

Deploy each container individually for full control.

**Pros:**
- Maximum flexibility
- Custom configuration

**Cons:**
- More complex
- More maintenance

**See:** [UNRAID-DEPLOYMENT.md](UNRAID-DEPLOYMENT.md#method-3-manual-container-setup)

## Requirements

- Unraid 6.9.0+
- 2GB+ RAM (4GB+ recommended)
- 2GB+ disk space
- Docker enabled

## Support

- **Full Documentation**: [UNRAID-DEPLOYMENT.md](UNRAID-DEPLOYMENT.md)
- **Troubleshooting**: [UNRAID-DEPLOYMENT.md#troubleshooting](UNRAID-DEPLOYMENT.md#troubleshooting)
- **GitHub Issues**: https://github.com/Relaticle/relaticle/issues
- **Community**: https://github.com/Relaticle/relaticle/discussions

## Submitting to Unraid Community Applications

This template is ready for submission to the Unraid Community Applications repository.

**Submission Requirements:**
- [x] Template XML file (`relaticle.xml`)
- [x] Icon URL (hosted on relaticle.com)
- [x] Comprehensive documentation
- [x] Docker Compose alternative
- [x] Working Docker image on GHCR

**To Submit:**
1. Fork the Community Applications repository: https://github.com/Squidly271/docker-templates
2. Add `relaticle.xml` to the repository
3. Create pull request with template
4. Reference this documentation

## For Unraid App Market Reviewers

**Template Quality Checklist:**

- ✅ Comprehensive help text in Overview
- ✅ All required environment variables clearly documented
- ✅ Secure defaults (passwords not hardcoded)
- ✅ Health checks implemented
- ✅ Volume mounts for persistence
- ✅ Proper dependency documentation
- ✅ Support and project URLs included
- ✅ Tested on Unraid 6.9+
- ✅ Multi-architecture support (amd64/arm64)
- ✅ Uses official Docker image from GHCR
- ✅ Complete deployment documentation

**Additional Resources:**
- Project Website: https://relaticle.com
- GitHub Repository: https://github.com/Relaticle/relaticle
- Docker Image: ghcr.io/relaticle/relaticle:latest
- License: AGPL-3.0

## License

Relaticle is open-source software licensed under the AGPL-3.0 license.
