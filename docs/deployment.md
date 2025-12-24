# Deployment Guide

Complete guide for deploying the SEO Platform to production environments.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Environment Configuration](#environment-configuration)
3. [Docker Deployment](#docker-deployment)
4. [Cloud Deployments](#cloud-deployments)
5. [Database Management](#database-management)
6. [SSL/TLS Configuration](#ssltls-configuration)
7. [Monitoring and Logging](#monitoring-and-logging)
8. [Backup and Recovery](#backup-and-recovery)
9. [Scaling](#scaling)
10. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Server Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| **CPU** | 2 cores | 4+ cores |
| **RAM** | 4 GB | 8+ GB |
| **Storage** | 50 GB SSD | 100+ GB SSD |
| **OS** | Ubuntu 22.04 LTS | Ubuntu 22.04 LTS |

### Required Software

```bash
# Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# Docker Compose
sudo apt-get update
sudo apt-get install docker-compose-plugin

# Verify installation
docker --version      # 24.0+
docker compose version  # 2.x+
```

### Domain and DNS

1. Register a domain (e.g., `yourdomain.com`)
2. Configure DNS records:

```
A     yourdomain.com        → <server_ip>
A     api.yourdomain.com    → <server_ip>
A     dashboard.yourdomain.com → <server_ip>
CNAME www.yourdomain.com    → yourdomain.com
```

---

## Environment Configuration

### Production Environment File

Create `.env` with production values:

```bash
# ===========================================
# DOMAIN CONFIGURATION
# ===========================================
DOMAIN=yourdomain.com
FRONTEND_HOST=https://dashboard.yourdomain.com
ENVIRONMENT=production
STACK_NAME=seo-platform

# ===========================================
# SECURITY
# ===========================================
# Generate with: openssl rand -hex 32
SECRET_KEY=your-256-bit-secret-key-here

# Fernet key for token encryption
# Generate with: python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
TOKEN_ENCRYPTION_KEY=your-fernet-key-here

# ===========================================
# DATABASE
# ===========================================
POSTGRES_SERVER=db
POSTGRES_PORT=5432
POSTGRES_DB=seo_platform
POSTGRES_USER=seo_platform
# Generate with: openssl rand -base64 32
POSTGRES_PASSWORD=your-strong-db-password

# ===========================================
# REDIS
# ===========================================
REDIS_URL=redis://redis:6379/0
CELERY_BROKER_URL=redis://redis:6379/0
CELERY_RESULT_BACKEND=redis://redis:6379/0

# ===========================================
# FIRST SUPERUSER
# ===========================================
FIRST_SUPERUSER=admin@yourdomain.com
FIRST_SUPERUSER_PASSWORD=change-this-password

# ===========================================
# EMAIL (SMTP)
# ===========================================
SMTP_HOST=smtp.sendgrid.net
SMTP_PORT=587
SMTP_USER=apikey
SMTP_PASSWORD=your-sendgrid-api-key
SMTP_TLS=true
EMAILS_FROM_EMAIL=noreply@yourdomain.com
EMAILS_FROM_NAME=SEO Platform

# ===========================================
# DOCKER IMAGES
# ===========================================
DOCKER_IMAGE_BACKEND=ghcr.io/yourorg/seo-platform-backend
DOCKER_IMAGE_FRONTEND=ghcr.io/yourorg/seo-platform-frontend
TAG=latest

# ===========================================
# CORS
# ===========================================
BACKEND_CORS_ORIGINS=["https://dashboard.yourdomain.com"]

# ===========================================
# GOOGLE OAUTH (Optional)
# ===========================================
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
GOOGLE_REDIRECT_URI=https://api.yourdomain.com/api/v1/gsc/callback

# Google Ads Developer Token (Optional)
GOOGLE_ADS_DEVELOPER_TOKEN=your-developer-token

# ===========================================
# MONITORING (Optional)
# ===========================================
SENTRY_DSN=https://xxx@sentry.io/xxx
```

### Security Checklist

- [ ] Generate unique `SECRET_KEY` (never reuse)
- [ ] Generate unique `TOKEN_ENCRYPTION_KEY`
- [ ] Use strong `POSTGRES_PASSWORD`
- [ ] Change `FIRST_SUPERUSER_PASSWORD` after first login
- [ ] Set proper `BACKEND_CORS_ORIGINS`
- [ ] Configure SMTP for email delivery
- [ ] Set up Sentry for error tracking

---

## Docker Deployment

### Production Stack

Use `docker-compose.yml` (production-ready):

```bash
# Clone repository
git clone https://github.com/yourorg/seo-platform.git
cd seo-platform

# Copy and configure environment
cp .env.example .env
nano .env  # Edit with production values

# Build images
docker compose build

# Start services
docker compose up -d

# View logs
docker compose logs -f
```

### Service Health Checks

```bash
# Check all services
docker compose ps

# Expected output:
# NAME           STATUS          PORTS
# backend        Up (healthy)    8000/tcp
# db             Up (healthy)    5432/tcp
# frontend       Up              80/tcp
# redis          Up (healthy)    6379/tcp
# worker         Up
# beat           Up
# traefik        Up              80/tcp, 443/tcp
```

### Verify Deployment

```bash
# Backend health
curl https://api.yourdomain.com/api/v1/utils/health-check/
# Expected: {"status":"healthy"}

# Frontend
curl -I https://dashboard.yourdomain.com
# Expected: HTTP/2 200
```

---

## Cloud Deployments

### AWS (EC2 + RDS)

#### 1. Launch EC2 Instance

```bash
# Recommended: t3.medium or larger
# AMI: Ubuntu 22.04 LTS
# Security Group: 22 (SSH), 80 (HTTP), 443 (HTTPS)
```

#### 2. Set Up RDS PostgreSQL

```bash
# Engine: PostgreSQL 15+
# Instance: db.t3.micro (dev) or db.t3.medium (prod)
# Enable: Multi-AZ for production
# Security Group: Allow from EC2
```

#### 3. Deploy

```bash
# SSH to EC2
ssh -i key.pem ubuntu@<ec2-ip>

# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker ubuntu

# Clone and configure
git clone https://github.com/yourorg/seo-platform.git
cd seo-platform

# Update .env with RDS endpoint
POSTGRES_SERVER=your-rds-endpoint.rds.amazonaws.com

# Deploy
docker compose up -d
```

### DigitalOcean (Droplet + Managed DB)

#### 1. Create Droplet

```bash
# Size: 4 GB RAM / 2 vCPUs minimum
# Image: Docker on Ubuntu 22.04
# Region: Choose closest to users
```

#### 2. Create Managed PostgreSQL

```bash
# Node: 1 GB RAM / 1 vCPU (dev) or 2 GB / 1 vCPU (prod)
# Version: PostgreSQL 15+
```

#### 3. Deploy

```bash
# SSH to droplet
ssh root@<droplet-ip>

# Clone and configure
git clone https://github.com/yourorg/seo-platform.git
cd seo-platform

# Update .env with managed DB connection string
# Deploy
docker compose up -d
```

### Google Cloud Platform (Cloud Run)

For serverless deployment:

```yaml
# cloudbuild.yaml
steps:
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'gcr.io/$PROJECT_ID/seo-backend', './backend']

  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'gcr.io/$PROJECT_ID/seo-backend']

  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
      - 'run'
      - 'deploy'
      - 'seo-backend'
      - '--image=gcr.io/$PROJECT_ID/seo-backend'
      - '--region=us-central1'
      - '--allow-unauthenticated'
```

---

## Database Management

### Running Migrations

```bash
# Apply all migrations
docker compose exec backend alembic upgrade head

# Check current revision
docker compose exec backend alembic current

# View migration history
docker compose exec backend alembic history
```

### Database Backup

#### Manual Backup

```bash
# Create backup
docker compose exec db pg_dump -U seo_platform seo_platform > backup_$(date +%Y%m%d).sql

# Compressed backup
docker compose exec db pg_dump -U seo_platform seo_platform | gzip > backup_$(date +%Y%m%d).sql.gz
```

#### Automated Backups (Cron)

```bash
# Add to crontab
crontab -e

# Daily backup at 2 AM
0 2 * * * cd /path/to/seo-platform && docker compose exec -T db pg_dump -U seo_platform seo_platform | gzip > /backups/backup_$(date +\%Y\%m\%d).sql.gz
```

### Database Restore

```bash
# Restore from backup
cat backup_20240115.sql | docker compose exec -T db psql -U seo_platform seo_platform

# From compressed backup
gunzip -c backup_20240115.sql.gz | docker compose exec -T db psql -U seo_platform seo_platform
```

---

## SSL/TLS Configuration

### Traefik with Let's Encrypt

The default `docker-compose.yml` includes Traefik with automatic SSL:

```yaml
# Already configured in docker-compose.yml
labels:
  - traefik.http.routers.backend-https.tls=true
  - traefik.http.routers.backend-https.tls.certresolver=le
```

### Manual Certificate (Optional)

If using custom certificates:

```yaml
# traefik/traefik.yml
tls:
  certificates:
    - certFile: /certs/yourdomain.com.crt
      keyFile: /certs/yourdomain.com.key
```

### Force HTTPS

Traefik automatically redirects HTTP to HTTPS via middleware:

```yaml
labels:
  - traefik.http.routers.backend-http.middlewares=https-redirect
```

---

## Monitoring and Logging

### Application Logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f backend
docker compose logs -f worker

# Last 100 lines
docker compose logs --tail=100 backend
```

### Sentry Error Tracking

Configure in `.env`:

```bash
SENTRY_DSN=https://xxx@sentry.io/xxx
```

Errors are automatically captured and reported.

### Celery Monitoring (Flower)

Access Flower dashboard:

```
http://yourdomain.com:5555
```

Shows:
- Active workers
- Task queues
- Task history
- Success/failure rates

### Health Check Endpoint

```bash
# Automated health monitoring
curl https://api.yourdomain.com/api/v1/utils/health-check/
```

Use with monitoring tools (UptimeRobot, Pingdom, etc.).

### Log Aggregation (Optional)

For centralized logging, add Loki or ELK stack:

```yaml
# docker-compose.override.yml
services:
  loki:
    image: grafana/loki:2.8.0
    ports:
      - "3100:3100"

  promtail:
    image: grafana/promtail:2.8.0
    volumes:
      - /var/log:/var/log
      - ./promtail-config.yml:/etc/promtail/config.yml
```

---

## Backup and Recovery

### Full System Backup

```bash
#!/bin/bash
# backup.sh

BACKUP_DIR=/backups/$(date +%Y%m%d)
mkdir -p $BACKUP_DIR

# Database
docker compose exec -T db pg_dump -U seo_platform seo_platform | gzip > $BACKUP_DIR/db.sql.gz

# Environment
cp .env $BACKUP_DIR/

# Upload to S3 (optional)
aws s3 sync $BACKUP_DIR s3://your-backup-bucket/$(date +%Y%m%d)/
```

### Disaster Recovery

```bash
# 1. Provision new server
# 2. Install Docker
# 3. Clone repository
git clone https://github.com/yourorg/seo-platform.git
cd seo-platform

# 4. Restore environment
cp /backups/20240115/.env .

# 5. Start services
docker compose up -d db redis
sleep 30  # Wait for DB to be ready

# 6. Restore database
gunzip -c /backups/20240115/db.sql.gz | docker compose exec -T db psql -U seo_platform seo_platform

# 7. Start remaining services
docker compose up -d
```

---

## Scaling

### Horizontal Scaling (Workers)

Increase Celery workers:

```yaml
# docker-compose.override.yml
services:
  worker:
    deploy:
      replicas: 4
```

Or scale manually:

```bash
docker compose up -d --scale worker=4
```

### Vertical Scaling (Resources)

Add resource limits:

```yaml
# docker-compose.override.yml
services:
  backend:
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '0.5'
          memory: 512M
```

### Load Balancing

Traefik automatically load balances across replicas:

```bash
docker compose up -d --scale backend=3
```

### Database Scaling

For high traffic:

1. **Read Replicas**: Route read queries to replicas
2. **Connection Pooling**: Use PgBouncer
3. **Managed Database**: Use AWS RDS, DigitalOcean Managed DB

---

## Troubleshooting

### Common Issues

#### 1. Container Won't Start

```bash
# Check logs
docker compose logs backend

# Common causes:
# - Missing environment variables
# - Database not ready
# - Port conflicts
```

#### 2. Database Connection Failed

```bash
# Verify database is running
docker compose exec db pg_isready -U seo_platform

# Check connection string
docker compose exec backend python -c "from app.core.db import engine; print(engine.url)"
```

#### 3. Migrations Failed

```bash
# Check current state
docker compose exec backend alembic current

# Reset to specific revision
docker compose exec backend alembic downgrade <revision>

# Regenerate migrations
docker compose exec backend alembic revision --autogenerate -m "fix"
```

#### 4. Celery Tasks Not Running

```bash
# Check worker status
docker compose logs worker

# Verify Redis connection
docker compose exec redis redis-cli ping

# Check Flower dashboard
open http://localhost:5555
```

#### 5. SSL Certificate Issues

```bash
# Check Traefik logs
docker compose logs traefik

# Verify DNS
nslookup api.yourdomain.com

# Test certificate
openssl s_client -connect api.yourdomain.com:443
```

### Debug Mode

For debugging, set environment:

```bash
ENVIRONMENT=local
```

This enables:
- Detailed error messages
- Auto-reload on code changes
- Debug toolbar

**Never use in production!**

### Getting Help

1. Check logs: `docker compose logs -f`
2. Review [Architecture](./architecture.md) for system design
3. Check [API Reference](./api-reference.md) for endpoint issues
4. Open GitHub issue with:
   - Docker version
   - Relevant logs
   - Steps to reproduce

---

## Maintenance

### Updating

```bash
# Pull latest code
git pull origin main

# Rebuild images
docker compose build

# Apply migrations
docker compose exec backend alembic upgrade head

# Restart services
docker compose up -d
```

### Zero-Downtime Deployment

```bash
# Scale up new version
docker compose up -d --scale backend=2 --no-recreate

# Wait for health checks
sleep 30

# Remove old containers
docker compose up -d --scale backend=1
```

### Cleanup

```bash
# Remove unused images
docker image prune -a

# Remove unused volumes (caution!)
docker volume prune

# Full cleanup
docker system prune -a
```

---

## Security Hardening

### Firewall (UFW)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
sudo ufw enable
```

### Fail2Ban

```bash
sudo apt install fail2ban
sudo systemctl enable fail2ban
```

### Docker Security

```yaml
# docker-compose.override.yml
services:
  backend:
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp
```

---

**Previous**: [API Reference](./api-reference.md) | [Architecture](./architecture.md)
