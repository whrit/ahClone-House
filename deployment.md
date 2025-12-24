# Production Deployment Guide

Comprehensive guide for deploying the SEO Platform to production environments.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Architecture Overview](#architecture-overview)
3. [Environment Configuration](#environment-configuration)
4. [Docker Deployment](#docker-deployment)
5. [Cloud Deployment Options](#cloud-deployment-options)
6. [Database Setup](#database-setup)
7. [Security Configuration](#security-configuration)
8. [Monitoring & Observability](#monitoring--observability)
9. [Backup & Recovery](#backup--recovery)
10. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required Software

- **Docker Engine**: v24.0+ (not Docker Desktop for production servers)
- **Docker Compose**: v2.x or later
- **PostgreSQL**: 17+ (or managed RDS/Cloud SQL/DigitalOcean Managed Database)
- **Redis**: 7+ (or managed ElastiCache/MemoryStore/DigitalOcean Managed Redis)
- **Domain**: Registered domain with DNS access
- **SSL Certificate**: Let's Encrypt (automatic via Traefik) or custom certificate

### Cloud Provider Account (Choose One)

- **AWS**: Account with IAM permissions for ECS, RDS, ElastiCache, ECR
- **Google Cloud Platform**: Account with permissions for Cloud Run, Cloud SQL, Memorystore, Artifact Registry
- **DigitalOcean**: Account for App Platform, Managed Databases, Container Registry
- **Self-Hosted**: VPS/Dedicated server with minimum 2 CPU cores, 4GB RAM, 40GB storage

### Development Tools

- `git` for version control
- `ssh` for server access
- `openssl` for generating secrets
- `curl` or `wget` for testing endpoints

### Knowledge Requirements

- Basic Linux command line proficiency
- Understanding of Docker and containerization
- Familiarity with environment variables and configuration management
- Basic networking knowledge (DNS, SSL/TLS, HTTP/HTTPS)

---

## Architecture Overview

### System Architecture Diagram

```
                                    Internet
                                       │
                                       ▼
                              ┌────────────────┐
                              │  Load Balancer │
                              │   (Traefik)    │
                              │   SSL/TLS      │
                              └────────┬───────┘
                                       │
                   ┌───────────────────┼───────────────────┐
                   │                   │                   │
                   ▼                   ▼                   ▼
          ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
          │   Frontend     │  │   Backend API  │  │    Adminer     │
          │  (Nginx/SPA)   │  │   (FastAPI)    │  │   (Admin UI)   │
          └────────────────┘  └───────┬────────┘  └────────────────┘
                                      │
                   ┌──────────────────┼──────────────────┐
                   │                  │                  │
                   ▼                  ▼                  ▼
          ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
          │    Worker      │  │     Beat       │  │    Flower      │
          │   (Celery)     │  │  (Scheduler)   │  │  (Monitor UI)  │
          └───────┬────────┘  └───────┬────────┘  └────────────────┘
                  │                   │
                  └──────────┬────────┘
                             │
                   ┌─────────┴─────────┐
                   │                   │
                   ▼                   ▼
          ┌────────────────┐  ┌────────────────┐
          │   PostgreSQL   │  │     Redis      │
          │   (Database)   │  │  (Cache/Queue) │
          └────────────────┘  └────────────────┘
```

### Components Description

#### Frontend Layer
- **Technology**: React SPA built with Vite
- **Web Server**: Nginx (production build served as static files)
- **Purpose**: User interface for the SEO platform
- **Access**: `https://dashboard.yourdomain.com`

#### Backend API
- **Technology**: FastAPI with Uvicorn (4 workers in production)
- **Purpose**: RESTful API, business logic, authentication
- **Access**: `https://api.yourdomain.com`
- **Health Check**: `/api/v1/utils/health-check/`

#### Background Workers
- **Worker (Celery)**: Processes background tasks (SERP refreshes, audits, data scraping)
  - Concurrency: 4 workers
  - Auto-restart on failure
  - Max tasks per child: 1000 (prevents memory leaks)

- **Beat (Celery Beat)**: Scheduled task coordinator
  - Manages periodic tasks (daily SERP updates, retention cleanup)
  - Single instance (do not scale horizontally)

- **Flower**: Web-based monitoring for Celery
  - Access: `https://flower.yourdomain.com`
  - Basic auth protected

#### Data Layer
- **PostgreSQL 17**: Primary database
  - Stores: users, projects, keywords, audit data, SERP snapshots
  - Connection pooling recommended for production
  - Automatic backups required

- **Redis 7**: Cache and message broker
  - Celery task queue
  - Session storage
  - Rate limiting data
  - LRU eviction policy

#### Reverse Proxy & SSL
- **Traefik**: Automatic SSL/TLS with Let's Encrypt
  - HTTP to HTTPS redirect
  - Certificate auto-renewal
  - Load balancing
  - Security headers

---

## Environment Configuration

### Required Environment Variables

Create a `.env` file in the project root:

```bash
# Domain Configuration
DOMAIN=yourdomain.com
FRONTEND_HOST=https://dashboard.yourdomain.com
ENVIRONMENT=production

# Project Identity
PROJECT_NAME="SEO Platform"
STACK_NAME=seo-platform-prod

# Backend Configuration
BACKEND_CORS_ORIGINS="https://dashboard.yourdomain.com"
SECRET_KEY=YOUR_SECURE_SECRET_KEY_HERE
FIRST_SUPERUSER=admin@yourdomain.com
FIRST_SUPERUSER_PASSWORD=SECURE_PASSWORD_HERE

# Email Configuration (SMTP)
SMTP_HOST=smtp.sendgrid.net
SMTP_USER=apikey
SMTP_PASSWORD=YOUR_SMTP_PASSWORD
EMAILS_FROM_EMAIL=noreply@yourdomain.com
SMTP_TLS=True
SMTP_SSL=False
SMTP_PORT=587

# PostgreSQL Database
POSTGRES_SERVER=db
POSTGRES_PORT=5432
POSTGRES_DB=seo_platform
POSTGRES_USER=postgres
POSTGRES_PASSWORD=SECURE_DB_PASSWORD_HERE

# Redis Configuration
REDIS_URL=redis://redis:6379/0
CELERY_BROKER_URL=redis://redis:6379/0
CELERY_RESULT_BACKEND=redis://redis:6379/0

# Monitoring
SENTRY_DSN=https://your-sentry-dsn@sentry.io/project-id

# Docker Images
DOCKER_IMAGE_BACKEND=your-registry/seo-platform-backend
DOCKER_IMAGE_FRONTEND=your-registry/seo-platform-frontend
TAG=latest

# Flower Monitoring
FLOWER_USER=admin
FLOWER_PASSWORD=SECURE_FLOWER_PASSWORD

# Google OAuth & APIs
GOOGLE_CLIENT_ID=your-client-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your-client-secret
GOOGLE_REDIRECT_URI=https://api.yourdomain.com/api/v1/integrations/google/callback
TOKEN_ENCRYPTION_KEY=YOUR_32_BYTE_FERNET_KEY
GOOGLE_ADS_DEVELOPER_TOKEN=your-ads-developer-token

# SERP API
SERP_API_KEY=your-serp-api-key
SERP_API_URL=https://serpapi.com/search

# Application Limits
MAX_KEYWORDS_PER_PROJECT=500
SERP_REFRESH_DAILY_CAP=100

# Data Retention (days)
RETENTION_JOB_RUNS=7
RETENTION_SERP_SNAPSHOTS=90
RETENTION_AUDIT_RUNS=30
RETENTION_LINK_SNAPSHOTS=60
RETENTION_GSC_DAILY=365
RETENTION_ADS_DAILY=365
```

### Generate Secure Secrets

```bash
# Generate SECRET_KEY
python -c "import secrets; print(secrets.token_urlsafe(32))"

# Generate POSTGRES_PASSWORD
openssl rand -base64 32

# Generate Fernet key for TOKEN_ENCRYPTION_KEY
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

### Secrets Management Best Practices

1. **Never commit `.env` files to version control**
   ```bash
   echo ".env" >> .gitignore
   ```

2. **Use a secrets manager** for production:
   - AWS Secrets Manager
   - Google Cloud Secret Manager
   - HashiCorp Vault
   - DigitalOcean App Platform Environment Variables

3. **Set restrictive file permissions**:
   ```bash
   chmod 600 .env
   ```

---

## Docker Deployment

### Production vs Development

This project includes two Docker Compose configurations:

- **`docker-compose.yml`**: Development with hot reload, exposed ports, volume mounts
- **`docker-compose.prod.yml`**: Production with resource limits, logging, security hardening

### Building Production Images

```bash
# Build backend image
docker build -t your-registry/seo-platform-backend:latest ./backend

# Build frontend image
docker build \
  --build-arg VITE_API_URL=https://api.yourdomain.com \
  --build-arg NODE_ENV=production \
  -t your-registry/seo-platform-frontend:latest ./frontend
```

### Deploying with Docker Compose (Self-Hosted)

#### Step 1: Set Up Traefik (One-Time Setup)

1. **Create Traefik directory**:
   ```bash
   mkdir -p /root/code/traefik-public
   ```

2. **Copy Traefik Docker Compose** to server:
   ```bash
   rsync -a docker-compose.traefik.yml root@your-server.com:/root/code/traefik-public/
   ```

3. **Create Traefik public network**:
   ```bash
   ssh root@your-server.com
   docker network create traefik-public
   ```

4. **Set Traefik environment variables**:
   ```bash
   export USERNAME=admin
   export PASSWORD=your-secure-password
   export HASHED_PASSWORD=$(openssl passwd -apr1 $PASSWORD)
   export DOMAIN=yourdomain.com
   export EMAIL=admin@yourdomain.com
   ```

5. **Start Traefik**:
   ```bash
   cd /root/code/traefik-public
   docker compose -f docker-compose.traefik.yml up -d
   ```

#### Step 2: Deploy the Application

1. **Clone repository on server**:
   ```bash
   git clone https://github.com/yourusername/seo-platform.git /root/code/seo-platform
   cd /root/code/seo-platform
   ```

2. **Create production `.env` file** with values from Environment Configuration section

3. **Pull latest images** (if using a registry):
   ```bash
   docker compose -f docker-compose.prod.yml pull
   ```

4. **Start the application**:
   ```bash
   docker compose -f docker-compose.prod.yml up -d
   ```

5. **Verify all services are running**:
   ```bash
   docker compose -f docker-compose.prod.yml ps
   ```

6. **Check logs**:
   ```bash
   docker compose -f docker-compose.prod.yml logs -f
   ```

#### Step 3: Verify Deployment

Test the following URLs:

- **Frontend**: `https://dashboard.yourdomain.com`
- **Backend API**: `https://api.yourdomain.com/docs`
- **Health Check**: `https://api.yourdomain.com/api/v1/utils/health-check/`
- **Adminer**: `https://adminer.yourdomain.com`
- **Flower**: `https://flower.yourdomain.com`
- **Traefik**: `https://traefik.yourdomain.com`

### Updating the Application

```bash
cd /root/code/seo-platform
git pull origin main

# Pull latest images or rebuild
docker compose -f docker-compose.prod.yml pull

# Restart services with zero downtime
docker compose -f docker-compose.prod.yml up -d --no-deps backend
docker compose -f docker-compose.prod.yml up -d --no-deps frontend
docker compose -f docker-compose.prod.yml up -d --no-deps worker
```

### Scaling Services

```bash
# Scale to 3 worker instances
docker compose -f docker-compose.prod.yml up -d --scale worker=3

# Note: Do NOT scale beat (scheduler) - only one instance should run
```

---

## Cloud Deployment Options

### Option 1: AWS (ECS Fargate)

#### Prerequisites
- AWS CLI configured
- ECR repositories created
- VPC with public and private subnets
- RDS PostgreSQL instance
- ElastiCache Redis cluster

#### Create RDS PostgreSQL Database

```bash
aws rds create-db-instance \
  --db-instance-identifier seo-platform-db \
  --db-instance-class db.t4g.medium \
  --engine postgres \
  --engine-version 17.2 \
  --master-username postgres \
  --master-user-password YOUR_SECURE_PASSWORD \
  --allocated-storage 20 \
  --storage-type gp3 \
  --vpc-security-group-ids sg-xxxxxxxxx \
  --db-subnet-group-name seo-platform-subnet-group \
  --backup-retention-period 7 \
  --enable-performance-insights \
  --publicly-accessible false
```

#### Create ElastiCache Redis Cluster

```bash
aws elasticache create-cache-cluster \
  --cache-cluster-id seo-platform-redis \
  --cache-node-type cache.t4g.micro \
  --engine redis \
  --engine-version 7.0 \
  --num-cache-nodes 1 \
  --cache-subnet-group-name seo-platform-subnet-group \
  --security-group-ids sg-xxxxxxxxx
```

#### Push Images to ECR

```bash
# Login to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# Build and push backend
docker build -t seo-platform-backend ./backend
docker tag seo-platform-backend:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/seo-platform-backend:latest
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/seo-platform-backend:latest

# Build and push frontend
docker build \
  --build-arg VITE_API_URL=https://api.yourdomain.com \
  -t seo-platform-frontend ./frontend
docker tag seo-platform-frontend:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/seo-platform-frontend:latest
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/seo-platform-frontend:latest
```

#### Create ECS Cluster

```bash
aws ecs create-cluster \
  --cluster-name seo-platform-cluster \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy capacityProvider=FARGATE,weight=1
```

#### Register Task Definition

Create `backend-task-definition.json` and register:

```bash
aws ecs register-task-definition --cli-input-json file://backend-task-definition.json
```

See full AWS deployment in the expanded documentation sections.

### Option 2: Google Cloud Platform (Cloud Run)

#### Create Cloud SQL PostgreSQL Instance

```bash
gcloud sql instances create seo-platform-db \
  --database-version=POSTGRES_17 \
  --tier=db-custom-2-4096 \
  --region=us-central1 \
  --backup-start-time=03:00 \
  --enable-bin-log \
  --retained-backups-count=7

# Create database
gcloud sql databases create seo_platform --instance=seo-platform-db
```

#### Create Cloud Memorystore Redis

```bash
gcloud redis instances create seo-platform-redis \
  --size=1 \
  --region=us-central1 \
  --redis-version=redis_7_0 \
  --tier=basic
```

#### Push to Artifact Registry

```bash
gcloud auth configure-docker us-central1-docker.pkg.dev

docker build -t us-central1-docker.pkg.dev/your-project/seo-platform/backend:latest ./backend
docker push us-central1-docker.pkg.dev/your-project/seo-platform/backend:latest
```

#### Deploy to Cloud Run

```bash
gcloud run deploy backend \
  --image us-central1-docker.pkg.dev/your-project/seo-platform/backend:latest \
  --region us-central1 \
  --platform managed \
  --allow-unauthenticated \
  --port 8000 \
  --cpu 2 \
  --memory 2Gi \
  --min-instances 1 \
  --max-instances 10 \
  --set-env-vars ENVIRONMENT=production \
  --set-secrets SECRET_KEY=seo-platform-secret-key:latest \
  --add-cloudsql-instances your-project:us-central1:seo-platform-db
```

### Option 3: DigitalOcean App Platform

#### Create App Platform Spec File

Create `.do/app.yaml`:

```yaml
name: seo-platform
region: nyc
services:
  - name: backend
    image:
      registry_type: DOCR
      repository: backend
      tag: latest
    instance_count: 2
    instance_size_slug: professional-xs
    http_port: 8000
    health_check:
      http_path: /api/v1/utils/health-check/
    routes:
      - path: /
    envs:
      - key: ENVIRONMENT
        value: production
      - key: SECRET_KEY
        scope: RUN_AND_BUILD_TIME
        type: SECRET

  - name: frontend
    image:
      registry_type: DOCR
      repository: frontend
      tag: latest
    instance_count: 1
    instance_size_slug: basic-xxs
    http_port: 80

  - name: worker
    image:
      registry_type: DOCR
      repository: backend
      tag: latest
    instance_count: 2
    instance_size_slug: professional-xs
    run_command: celery -A app.core.celery worker --loglevel=info

databases:
  - name: db
    engine: PG
    version: "17"
    production: true
  - name: redis
    engine: REDIS
    version: "7"
    production: true
```

#### Deploy

```bash
doctl apps create --spec .do/app.yaml
```

---

## Database Setup

### Initial Database Setup

The `prestart` service in Docker Compose automatically:
1. Waits for PostgreSQL to be ready
2. Runs migrations (`alembic upgrade head`)
3. Creates initial data (first superuser)

### Manual Database Operations

```bash
# Run migrations manually
docker compose -f docker-compose.prod.yml exec backend alembic upgrade head

# View migration history
docker compose -f docker-compose.prod.yml exec backend alembic history

# Create new migration
docker compose -f docker-compose.prod.yml exec backend alembic revision --autogenerate -m "Description"

# Database shell access
docker compose -f docker-compose.prod.yml exec db psql -U postgres -d seo_platform
```

### Connection Pooling with PgBouncer

Add to `docker-compose.prod.yml`:

```yaml
  pgbouncer:
    image: edoburu/pgbouncer:1.21.0
    restart: always
    environment:
      - DB_HOST=db
      - DB_PORT=5432
      - DB_USER=${POSTGRES_USER}
      - DB_PASSWORD=${POSTGRES_PASSWORD}
      - DB_NAME=${POSTGRES_DB}
      - POOL_MODE=transaction
      - MAX_CLIENT_CONN=1000
      - DEFAULT_POOL_SIZE=20
    depends_on:
      - db
    networks:
      - default
```

Update backend services to use `POSTGRES_SERVER=pgbouncer`.

---

## Security Configuration

### SSL/TLS with Let's Encrypt

Traefik automatically provisions and renews SSL certificates.

Verify certificates:
```bash
openssl s_client -connect yourdomain.com:443 -servername yourdomain.com < /dev/null | openssl x509 -noout -dates
```

### Firewall Configuration

#### UFW (Ubuntu/Debian)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

#### AWS Security Groups

**Load Balancer Security Group**:
- HTTP (80) from 0.0.0.0/0
- HTTPS (443) from 0.0.0.0/0

**Database Security Group**:
- PostgreSQL (5432) from Backend security group only

**Redis Security Group**:
- Custom TCP (6379) from Backend security group only

### Security Headers

Configured via Traefik labels in `docker-compose.prod.yml`:
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `X-XSS-Protection: 1; mode=block`
- `Strict-Transport-Security: max-age=31536000; includeSubDomains`

### Secrets Rotation

#### Rotating SECRET_KEY

```bash
NEW_SECRET=$(python -c "import secrets; print(secrets.token_urlsafe(32))")
sed -i "s/SECRET_KEY=.*/SECRET_KEY=$NEW_SECRET/" .env
docker compose -f docker-compose.prod.yml restart backend worker beat
```

#### Rotating Database Password

```bash
docker compose -f docker-compose.prod.yml exec db psql -U postgres -c "ALTER USER postgres WITH PASSWORD 'NEW_PASSWORD';"
sed -i "s/POSTGRES_PASSWORD=.*/POSTGRES_PASSWORD=NEW_PASSWORD/" .env
docker compose -f docker-compose.prod.yml restart backend worker beat
```

---

## Monitoring & Observability

### Health Checks

```bash
# Backend health check
curl https://api.yourdomain.com/api/v1/utils/health-check/

# Service-level health checks
docker compose -f docker-compose.prod.yml ps
```

### Logging

```bash
# View all logs
docker compose -f docker-compose.prod.yml logs -f

# View specific service logs
docker compose -f docker-compose.prod.yml logs -f backend

# Last 100 lines
docker compose -f docker-compose.prod.yml logs --tail=100 backend
```

### Monitoring with Prometheus & Grafana

Add to `docker-compose.prod.yml`:

```yaml
  prometheus:
    image: prom/prometheus:latest
    restart: always
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    networks:
      - default
    labels:
      - traefik.enable=true
      - traefik.http.routers.${STACK_NAME}-prometheus.rule=Host(`prometheus.${DOMAIN}`)

  grafana:
    image: grafana/grafana:latest
    restart: always
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD:-admin}
    volumes:
      - grafana-data:/var/lib/grafana
    networks:
      - traefik-public
    labels:
      - traefik.enable=true
      - traefik.http.routers.${STACK_NAME}-grafana.rule=Host(`grafana.${DOMAIN}`)

volumes:
  prometheus-data:
  grafana-data:
```

### Alerting

Create `alerts.yml` for Prometheus:

```yaml
groups:
  - name: application
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"

      - alert: ServiceDown
        expr: up == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} is down"
```

---

## Backup & Recovery

### Automated Database Backups

Add to `docker-compose.prod.yml`:

```yaml
  db-backup:
    image: prodrigestivill/postgres-backup-local:17
    restart: always
    volumes:
      - ./backups:/backups
    environment:
      - POSTGRES_HOST=db
      - POSTGRES_DB=${POSTGRES_DB}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - SCHEDULE=@daily
      - BACKUP_KEEP_DAYS=7
      - BACKUP_KEEP_WEEKS=4
      - BACKUP_KEEP_MONTHS=6
    depends_on:
      - db
```

### Manual Backup & Restore

```bash
# Backup
docker compose -f docker-compose.prod.yml exec db pg_dump -U postgres seo_platform | gzip > backup_$(date +%Y%m%d).sql.gz

# Restore
gunzip < backup_20240101.sql.gz | \
  docker compose -f docker-compose.prod.yml exec -T db psql -U postgres seo_platform
```

### Backup to S3

Create `/root/scripts/backup-to-s3.sh`:

```bash
#!/bin/bash
BACKUP_FILE="seo_platform_$(date +%Y%m%d_%H%M%S).sql.gz"
docker compose -f /root/code/seo-platform/docker-compose.prod.yml exec -T db \
  pg_dump -U postgres seo_platform | gzip > /tmp/$BACKUP_FILE
aws s3 cp /tmp/$BACKUP_FILE s3://your-backup-bucket/database/
rm /tmp/$BACKUP_FILE
```

Add to crontab:
```bash
0 3 * * * /root/scripts/backup-to-s3.sh
```

### Disaster Recovery

**Recovery Time Objective (RTO)**: 1 hour
**Recovery Point Objective (RPO)**: 24 hours

Steps:
1. Provision new infrastructure
2. Restore application code
3. Restore environment configuration
4. Restore database from backup
5. Start all services
6. Verify functionality

---

## Troubleshooting

### Common Issues

#### Container Fails to Start

**Diagnosis**:
```bash
docker compose -f docker-compose.prod.yml logs backend
docker compose -f docker-compose.prod.yml ps backend
```

**Solutions**:
- Check for missing environment variables
- Verify database is ready
- Check if port is already in use

#### SSL Certificate Not Provisioning

**Diagnosis**:
```bash
docker logs traefik
nslookup yourdomain.com
```

**Solutions**:
- Verify DNS is pointing to server IP
- Ensure port 80 is open (required for Let's Encrypt)
- Check Let's Encrypt rate limits
- Verify acme.json permissions: `chmod 600 acme.json`

#### Database Connection Errors

**Diagnosis**:
```bash
docker compose -f docker-compose.prod.yml exec backend \
  python -c "from app.core.db import engine; engine.connect(); print('Connected')"
```

**Solutions**:
- Verify credentials in .env match database
- Check database is running
- Verify network connectivity

#### Celery Workers Not Processing Tasks

**Diagnosis**:
```bash
docker compose -f docker-compose.prod.yml logs worker
# Check Flower dashboard: https://flower.yourdomain.com
```

**Solutions**:
- Test Redis connection: `docker compose exec redis redis-cli ping`
- Restart workers: `docker compose restart worker`
- Check memory usage and reduce concurrency if needed

#### High Memory Usage

**Diagnosis**:
```bash
docker stats
free -h
```

**Solutions**:
- Increase server memory
- Reduce resource limits in docker-compose.prod.yml
- Reduce Celery concurrency
- Enable swap (not recommended for production)

#### Slow API Response Times

**Diagnosis**:
```bash
time curl https://api.yourdomain.com/api/v1/utils/health-check/
```

**Solutions**:
- Add database indexes
- Enable connection pooling (PgBouncer)
- Scale backend instances
- Optimize queries

### Log Analysis

```bash
# Find errors in last hour
docker compose -f docker-compose.prod.yml logs --since 1h | grep -i error

# Find specific error pattern
docker compose -f docker-compose.prod.yml logs backend | grep "ConnectionError"

# Count error occurrences
docker compose -f docker-compose.prod.yml logs backend | grep -c "500 Internal Server Error"
```

---

## Appendix

### Environment Variables Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `DOMAIN` | Yes | `localhost` | Primary domain |
| `FRONTEND_HOST` | Yes | - | Full frontend URL |
| `ENVIRONMENT` | Yes | `local` | Environment: local, staging, production |
| `SECRET_KEY` | Yes | - | Secret key for JWT signing |
| `POSTGRES_SERVER` | Yes | - | PostgreSQL host |
| `POSTGRES_PASSWORD` | Yes | - | Database password |
| `REDIS_URL` | No | `redis://localhost:6379/0` | Redis connection URL |
| `SENTRY_DSN` | No | - | Sentry DSN for error tracking |

(See full table in Environment Configuration section)

### Quick Reference Commands

```bash
# Start production stack
docker compose -f docker-compose.prod.yml up -d

# Stop stack
docker compose -f docker-compose.prod.yml down

# View logs
docker compose -f docker-compose.prod.yml logs -f

# Restart service
docker compose -f docker-compose.prod.yml restart backend

# Scale workers
docker compose -f docker-compose.prod.yml up -d --scale worker=3

# Run database migration
docker compose -f docker-compose.prod.yml exec backend alembic upgrade head

# Create database backup
docker compose -f docker-compose.prod.yml exec db pg_dump -U postgres seo_platform | gzip > backup.sql.gz

# Access database shell
docker compose -f docker-compose.prod.yml exec db psql -U postgres seo_platform

# View resource usage
docker stats
```

### DNS Configuration Example (Cloudflare)

```
Type    Name        Content             Proxy   TTL
A       @           123.45.67.89        Yes     Auto
A       www         123.45.67.89        Yes     Auto
CNAME   api         yourdomain.com      Yes     Auto
CNAME   dashboard   yourdomain.com      Yes     Auto
CNAME   traefik     yourdomain.com      Yes     Auto
CNAME   adminer     yourdomain.com      Yes     Auto
CNAME   flower      yourdomain.com      Yes     Auto
```

### Production Readiness Checklist

**Infrastructure**
- [ ] Server provisioned with sufficient resources
- [ ] Docker and Docker Compose installed
- [ ] Firewall configured
- [ ] DNS records configured
- [ ] SSL certificates provisioned

**Application**
- [ ] .env file configured with production values
- [ ] All secrets changed from defaults
- [ ] Database initialized with migrations
- [ ] CORS origins configured correctly

**Security**
- [ ] All default passwords changed
- [ ] Secrets stored securely
- [ ] Database not publicly accessible
- [ ] Security headers configured
- [ ] Adminer protected or disabled

**Monitoring**
- [ ] Sentry configured
- [ ] Health checks responding
- [ ] Logging configured
- [ ] Alerting configured

**Backup & Recovery**
- [ ] Automated backups configured
- [ ] Backups tested
- [ ] Disaster recovery plan documented

---

**Last Updated**: December 2024
**Version**: 1.0.0

For the existing Traefik-based deployment guide, see `deployment.md.bak`.
