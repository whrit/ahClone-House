# Getting Started with SEO Platform

This guide will help you get the SEO Platform up and running in 15 minutes.

---

## Prerequisites

Before you begin, ensure you have the following installed:

### Required Software

| Tool | Version | Purpose | Installation |
|------|---------|---------|--------------|
| **Docker** | 24.0+ | Container runtime | [docker.com](https://www.docker.com/get-started) |
| **Docker Compose** | 2.x+ | Multi-container orchestration | Included with Docker Desktop |
| **Git** | Latest | Version control | [git-scm.com](https://git-scm.com/) |

### Optional (for development)

| Tool | Version | Purpose |
|------|---------|---------|
| **Node.js** | 20+ | Frontend development |
| **Python** | 3.12+ | Backend development |
| **uv** | Latest | Python package manager |

### System Requirements

- **CPU**: 2 cores minimum (4+ recommended)
- **RAM**: 4GB minimum (8GB+ recommended)
- **Storage**: 20GB minimum (40GB+ recommended)
- **OS**: Linux, macOS, or Windows with WSL2

---

## Installation

### Step 1: Clone the Repository

```bash
git clone https://github.com/yourusername/seo-platform.git
cd seo-platform
```

### Step 2: Configure Environment Variables

Copy the example environment file and customize it:

```bash
cp .env.example .env
```

Edit `.env` with your preferred editor:

```bash
# Minimal configuration for local development
DOMAIN=localhost
FRONTEND_HOST=http://localhost:5173
ENVIRONMENT=local

PROJECT_NAME="SEO Platform"
STACK_NAME=seo-platform

# Change these from defaults!
SECRET_KEY=changethis
FIRST_SUPERUSER=admin@example.com
FIRST_SUPERUSER_PASSWORD=changethis

# Database
POSTGRES_SERVER=localhost
POSTGRES_PORT=5432
POSTGRES_DB=app
POSTGRES_USER=postgres
POSTGRES_PASSWORD=changethis

# Optional: Google OAuth (for GSC and Ads integration)
# GOOGLE_CLIENT_ID=your-client-id.apps.googleusercontent.com
# GOOGLE_CLIENT_SECRET=your-client-secret
```

#### Generate Secure Secrets

For production, generate secure random values:

```bash
# Generate SECRET_KEY
python -c "import secrets; print(secrets.token_urlsafe(32))"

# Generate POSTGRES_PASSWORD
openssl rand -base64 32

# Generate TOKEN_ENCRYPTION_KEY (if using Google OAuth)
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

### Step 3: Start the Development Stack

```bash
docker compose up -d
```

This command will:
1. Pull required Docker images (PostgreSQL, Redis, etc.)
2. Build the backend and frontend containers
3. Run database migrations
4. Create the first superuser account
5. Start all services

#### Wait for Services to Start

Monitor the logs to ensure all services are healthy:

```bash
docker compose logs -f
```

Look for these success messages:
- `backend_1    | Application startup complete`
- `worker_1     | celery@worker ready`
- `beat_1       | beat: Starting...`

Press `Ctrl+C` to stop following logs.

### Step 4: Verify Installation

Check that all containers are running:

```bash
docker compose ps
```

You should see:
- `db` (PostgreSQL) - healthy
- `redis` - healthy
- `backend` - running
- `frontend` - running
- `worker` (Celery) - running
- `beat` (Celery Beat) - running
- `flower` (monitoring) - running
- `adminer` (database UI) - running

### Step 5: Access the Platform

Open your browser and navigate to:

| Service | URL | Purpose |
|---------|-----|---------|
| **Frontend Dashboard** | http://localhost:5173 | Main user interface |
| **Backend API Docs** | http://localhost:8000/docs | Interactive API documentation |
| **Flower (Celery UI)** | http://localhost:5555 | Background job monitoring |
| **Adminer (Database UI)** | http://localhost:8080 | Database administration |

---

## First Login

1. **Navigate to the frontend**: http://localhost:5173

2. **Login with your superuser credentials**:
   - Email: `admin@example.com` (or what you set in `.env`)
   - Password: `changethis` (or what you set in `.env`)

3. **You should see the dashboard** with an empty projects list.

---

## Creating Your First Project

### Step 1: Create a Project

1. Click **"New Project"** button in the dashboard
2. Fill in the project details:
   - **Name**: "My Website"
   - **Domain**: "example.com"
   - **Description**: "Main company website"
3. Click **"Create"**

### Step 2: Configure Project Settings (Optional)

Click on your newly created project, then navigate to **Settings**:

- **Audit Settings**:
  - Max pages to crawl: 1000 (default)
  - JS rendering enabled: Yes
  - Follow external links: No

- **SERP Settings**:
  - Refresh frequency: Daily
  - Provider: GSC-based (default)

---

## Running Your First Audit

### Step 1: Start an Audit

1. Navigate to your project
2. Click **"Audits"** in the sidebar
3. Click **"Run New Audit"**
4. The audit will be queued and start processing

### Step 2: Monitor Audit Progress

- View the audit status in the **Audits** list
- Status will change from `queued` → `running` → `completed`
- Click on the audit to see real-time progress

### Step 3: View Audit Results

Once completed, explore:

- **Summary**: Overview of issues by severity
- **Issues**: Detailed list of SEO problems
  - Filter by severity (Critical, High, Medium, Low)
  - Filter by type (Title, Meta, Headings, Images, etc.)
  - Export to CSV
- **Pages**: All crawled pages with their metrics
  - Status codes
  - Response times
  - Word counts
  - Links found

### Understanding Issue Severity

| Severity | Description | Examples |
|----------|-------------|----------|
| **Critical** | Major SEO problems | Missing title tags, 5xx errors |
| **High** | Important issues | Duplicate titles, broken links |
| **Medium** | Moderate concerns | Missing meta descriptions, slow load times |
| **Low** | Minor optimizations | Image alt text, heading order |

---

## Connecting Google Search Console

### Prerequisites

1. You need a Google account with access to a GSC property
2. The property must have at least 28 days of data

### Step 1: Set Up Google OAuth

Add these to your `.env` file:

```bash
GOOGLE_CLIENT_ID=your-client-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your-client-secret
GOOGLE_REDIRECT_URI=http://localhost:8000/api/v1/integrations/google/callback
TOKEN_ENCRYPTION_KEY=your-fernet-key
```

To get Google OAuth credentials:

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project or select existing
3. Enable **Google Search Console API**
4. Create OAuth 2.0 credentials (Web application)
5. Add authorized redirect URI: `http://localhost:8000/api/v1/integrations/google/callback`
6. Copy the Client ID and Client Secret

### Step 2: Connect GSC

1. Navigate to your project
2. Click **"Keywords"** in the sidebar
3. Click **"Connect Google Search Console"**
4. Authorize the application
5. Select your GSC property from the list

### Step 3: Sync Data

1. Click **"Sync Now"** to fetch the latest 28 days of data
2. Wait for the sync to complete (check status in Jobs)
3. Explore the data:
   - **Query Explorer**: Search queries with clicks, impressions, CTR, position
   - **Page Explorer**: Top landing pages by traffic
   - **Opportunities**: Low-CTR queries, positions 8-20, rising/falling queries
   - **Clusters**: Auto-grouped related queries

---

## Tracking Keyword Rankings

### Step 1: Add Keywords to Track

1. Navigate to **Rank Tracker** in your project
2. Click **"Add Keyword"**
3. Enter:
   - **Keyword**: "seo platform"
   - **Target URL**: Leave blank for auto-detection
   - **Refresh frequency**: Daily
4. Click **"Save"**

### Step 2: Wait for First Refresh

- If you connected GSC, the position will be fetched from GSC data (instant)
- Otherwise, the keyword will be queued for the next SERP refresh
- Check the **Jobs** section to monitor refresh status

### Step 3: View Ranking History

1. Click on a keyword to see details
2. View:
   - **Position trend chart**: 30-day history
   - **SERP snapshot**: Top 10 results for the keyword
   - **Change alerts**: Position improvements/declines

---

## Exploring Backlinks

### Step 1: Ingest Common Crawl Data

1. Navigate to **Links** in your project
2. Click **"Ingest Common Crawl"**
3. Select a recent Common Crawl index (e.g., CC-MAIN-2024-10)
4. Click **"Start Ingestion"**

This will:
- Download WARC files from Common Crawl
- Parse HTML and extract links
- Store backlinks pointing to your domain

**Note**: Ingestion can take 30+ minutes for large sites.

### Step 2: Explore Backlink Data

Once ingestion completes:

- **Referring Domains**: Top domains linking to you
  - Domain authority indicators
  - Link counts
  - Follow/nofollow ratio

- **Backlinks**: Individual link details
  - Source URL and anchor text
  - Target page
  - Link attributes (nofollow, sponsored, ugc)

- **Anchors**: Most used anchor text
  - Frequency analysis
  - Natural vs exact match distribution

- **Competitive**: Compare backlinks with competitors
  - Overlap analysis
  - Unique backlinks
  - Link intersect

---

## Viewing PPC Data (Google Ads)

### Prerequisites

1. Google Ads account with active campaigns
2. Google Ads Developer Token (apply at Google Ads API center)

### Step 1: Configure Google Ads OAuth

Add to `.env`:

```bash
GOOGLE_ADS_DEVELOPER_TOKEN=your-developer-token
```

### Step 2: Connect Google Ads

1. Navigate to **PPC** in your project
2. Click **"Connect Google Ads"**
3. Authorize and select your Ads account

### Step 3: Sync Campaign Data

1. Click **"Sync Campaigns"**
2. View:
   - Campaign performance (clicks, impressions, cost, conversions)
   - Keyword performance by ad group
   - SEO+PPC overlap (keywords you rank for organically that you're also bidding on)

---

## Using the Traffic Panel

The Traffic Panel aggregates data from multiple sources:

### Supported Sources

1. **Google Analytics 4** (via API)
2. **Google Search Console** (already connected)
3. **Chrome UX Report** (CrUX - public dataset)
4. **CSV Import** (custom data)

### Import CSV Traffic Data

1. Navigate to **Traffic** in your project
2. Click **"Import CSV"**
3. Upload a CSV with columns:
   ```
   date,source,sessions,pageviews,users,bounce_rate
   2024-01-01,organic,1234,5678,890,45.2
   2024-01-02,direct,567,890,234,52.1
   ```
4. Map columns to platform fields
5. Click **"Import"**

### View Aggregated Traffic

- Multi-source traffic chart
- Breakdown by source (organic, direct, referral, social)
- Date range filtering
- Export options

---

## Common Tasks

### Scheduling Background Jobs

Background jobs run automatically via Celery Beat:

| Job | Frequency | Purpose |
|-----|-----------|---------|
| **GSC Daily Sync** | Daily at 3 AM | Fetch yesterday's GSC data |
| **SERP Refresh** | Daily at 4 AM | Update keyword positions |
| **Audit Cleanup** | Weekly | Delete audits older than 30 days |
| **Snapshot Cleanup** | Weekly | Delete SERP snapshots older than 90 days |

### Monitoring Background Jobs

1. Open Flower UI: http://localhost:5555
2. View:
   - Active tasks
   - Completed tasks
   - Failed tasks
   - Worker status

### Viewing Logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f backend
docker compose logs -f worker

# Last 100 lines
docker compose logs --tail=100 backend
```

### Stopping the Platform

```bash
# Stop all services
docker compose down

# Stop and remove volumes (WARNING: deletes database!)
docker compose down -v
```

### Updating the Platform

```bash
# Pull latest code
git pull origin main

# Rebuild containers
docker compose build

# Restart services
docker compose up -d
```

---

## Troubleshooting

### Services Won't Start

**Check logs**:
```bash
docker compose logs backend
```

**Common issues**:
- Port already in use (change ports in `docker-compose.yml`)
- Missing environment variables (check `.env`)
- Database connection failed (wait for `db` service to be healthy)

### Can't Log In

1. Verify superuser was created:
   ```bash
   docker compose logs prestart
   ```
2. Reset password in database:
   ```bash
   docker compose exec backend python -c "from app import crud; from app.core.db import SessionLocal; ..."
   ```

### Audit Not Starting

1. Check Celery worker is running:
   ```bash
   docker compose logs worker
   ```
2. Check Redis connection:
   ```bash
   docker compose exec redis redis-cli ping
   ```
3. View task status in Flower: http://localhost:5555

### GSC Connection Fails

1. Verify OAuth credentials in `.env`
2. Check redirect URI matches Google Cloud Console
3. Ensure GSC API is enabled in your Google Cloud project
4. Check backend logs for detailed error messages

---

## Next Steps

Now that you have the platform running:

1. **Read the [User Guide](./user-guide.md)** for detailed feature documentation
2. **Check the [API Reference](./api-reference.md)** to integrate with other tools
3. **Explore [Development Guide](./development.md)** if you want to customize the platform
4. **Review [Deployment Guide](./deployment.md)** for production deployment

---

## Getting Help

- **Documentation**: Check other guides in this folder
- **API Issues**: View logs with `docker compose logs backend`
- **Frontend Issues**: View browser console and `docker compose logs frontend`
- **Background Jobs**: Check Flower UI at http://localhost:5555

---

**Congratulations!** You now have a fully functional SEO platform running locally. Start exploring the features and optimizing your website's SEO.
