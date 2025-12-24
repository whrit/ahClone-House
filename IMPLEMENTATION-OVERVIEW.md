# Implementation Overview: MVP SEO Platform

> **Reference:** [BLUEPRINT-PRD.md](./BLUEPRINT-PRD.md)
> **Base Template:** fastapi/full-stack-fastapi-template (already scaffolded)
> **Current State:** Auth + Users working; Zero SEO features implemented

---

## 1) Current State Assessment

### What's Already Working
| Component | Status | Details |
|-----------|--------|---------|
| **Authentication** | ✅ Complete | JWT, OAuth2 password flow, password recovery, email verification |
| **User Management** | ✅ Complete | CRUD, superuser roles, registration, profile updates |
| **Database** | ✅ Ready | PostgreSQL 17, Alembic migrations, UUID primary keys |
| **API Foundation** | ✅ Ready | FastAPI with dependency injection, error handling, CORS |
| **Frontend Shell** | ✅ Ready | React + TanStack Router, shadcn/ui, dark mode, sidebar layout |
| **DevOps** | ✅ Ready | Docker Compose, Traefik, health checks, CI pipeline |
| **OpenAPI Client Gen** | ✅ Ready | Auto-generated TypeScript client from backend schema |

### What's Missing (MVP Scope)
| Feature | Priority | Complexity | Dependencies |
|---------|----------|------------|--------------|
| **Projects Module** | P0 | Low | None |
| **Background Jobs (Celery)** | P0 | Medium | Redis |
| **Site Audit Engine** | P0 | High | Projects, Celery, Playwright |
| **GSC Integration** | P1 | Medium | Projects, Celery, Google OAuth |
| **Keyword Tooling** | P1 | Medium | GSC Integration |
| **SERP/Rank Tracking** | P2 | Medium | Projects, Celery |
| **Backlinks (Common Crawl)** | P2 | High | Celery, optional ClickHouse |
| **PPC + Traffic Panel** | P3 | Medium | Google Ads OAuth |

---

## 2) Architecture Decisions

### 2.1 Backend Module Structure

```
backend/app/
├── api/
│   └── routes/
│       ├── login.py          # (existing)
│       ├── users.py          # (existing)
│       ├── projects.py       # NEW: Project CRUD
│       ├── audits.py         # NEW: Audit runs, issues, pages
│       ├── gsc.py            # NEW: GSC connection, sync, explorers
│       ├── serp.py           # NEW: Keyword targets, rank observations
│       ├── links.py          # NEW: Backlink explorer, competitive
│       ├── ads.py            # NEW: PPC dashboards, overlap
│       ├── traffic.py        # NEW: Traffic panel sources
│       └── alerts.py         # NEW: Alert settings, history
├── models/
│   ├── user.py               # (extract from models.py)
│   ├── project.py            # NEW
│   ├── audit.py              # NEW
│   ├── gsc.py                # NEW
│   ├── serp.py               # NEW
│   ├── links.py              # NEW
│   ├── ads.py                # NEW
│   └── traffic.py            # NEW
├── services/
│   ├── audit/
│   │   ├── crawler.py        # HTML crawler with httpx
│   │   ├── renderer.py       # Playwright JS rendering
│   │   ├── analyzer.py       # Issue detection rules
│   │   └── differ.py         # Run-to-run diff logic
│   ├── gsc/
│   │   ├── client.py         # GSC API wrapper
│   │   ├── ingestor.py       # Daily data pull
│   │   ├── opportunities.py  # Scoring logic
│   │   └── clustering.py     # Query clustering
│   ├── serp/
│   │   ├── providers/
│   │   │   ├── base.py       # SerpProvider interface
│   │   │   ├── gsc_based.py  # Default: GSC position data
│   │   │   └── api_provider.py # Optional: third-party API
│   │   └── tracker.py        # Refresh + observation storage
│   ├── links/
│   │   ├── commoncrawl.py    # CC ingestion pipeline
│   │   ├── aggregator.py     # Referring domain aggregation
│   │   └── competitive.py    # Overlap/intersect queries
│   └── ads/
│       ├── google_ads.py     # Google Ads API client
│       ├── transparency.py   # Ads transparency datasets
│       └── overlap.py        # SEO+PPC overlap analysis
├── tasks/                    # NEW: Celery tasks
│   ├── __init__.py           # Celery app configuration
│   ├── audit.py              # Audit pipeline tasks
│   ├── gsc.py                # GSC sync/backfill tasks
│   ├── serp.py               # SERP refresh tasks
│   ├── links.py              # Common Crawl ingestion tasks
│   └── ads.py                # PPC sync tasks
└── core/
    ├── celery.py             # NEW: Celery app factory
    └── oauth/
        ├── google.py         # NEW: Google OAuth helpers
        └── encryption.py     # NEW: Token encryption
```

### 2.2 Frontend Page Structure

```
frontend/src/routes/
├── _layout/
│   ├── index.tsx                    # Dashboard (modify for overview)
│   ├── projects/
│   │   ├── index.tsx                # Project list
│   │   ├── new.tsx                  # Create project
│   │   └── $projectId/
│   │       ├── index.tsx            # Project overview
│   │       ├── settings.tsx         # Project settings
│   │       ├── audits/
│   │       │   ├── index.tsx        # Audit runs list
│   │       │   └── $auditId/
│   │       │       ├── index.tsx    # Audit summary
│   │       │       ├── issues.tsx   # Issues explorer
│   │       │       └── pages.tsx    # Pages explorer
│   │       ├── keywords/
│   │       │   ├── index.tsx        # Query explorer
│   │       │   ├── pages.tsx        # Page explorer
│   │       │   ├── opportunities.tsx
│   │       │   └── clusters.tsx
│   │       ├── rank-tracker/
│   │       │   ├── index.tsx        # Keyword targets list
│   │       │   └── $keywordId.tsx   # Keyword detail + SERP viewer
│   │       ├── links/
│   │       │   ├── index.tsx        # Referring domains
│   │       │   ├── backlinks.tsx    # Backlinks list
│   │       │   ├── anchors.tsx      # Anchor text analysis
│   │       │   └── competitive.tsx  # Overlap/intersect
│   │       ├── ppc/
│   │       │   ├── index.tsx        # PPC dashboard
│   │       │   ├── overlap.tsx      # SEO+PPC overlap
│   │       │   └── transparency.tsx # Competitor ads
│   │       └── traffic/
│   │           └── index.tsx        # Traffic panel
│   └── integrations/
│       └── google.tsx               # Google OAuth callback
```

### 2.3 Database Schema Strategy

**Approach:** Keep all models in Postgres initially. Plan for ClickHouse migration of `BacklinkEdge` table when data volume grows.

**Migration Strategy:**
1. Create new Alembic migration per feature module
2. Use UUID primary keys (already pattern in codebase)
3. Use JSON columns for flexible settings/metadata
4. Add indexes on foreign keys and common query patterns

### 2.4 Background Job Architecture

```
┌─────────────────┐    ┌─────────────┐    ┌─────────────────┐
│   Celery Beat   │───▶│    Redis    │◀───│  Celery Worker  │
│   (Scheduler)   │    │   (Broker)  │    │   (Executor)    │
└─────────────────┘    └─────────────┘    └─────────────────┘
                                                   │
                                                   ▼
                                          ┌─────────────────┐
                                          │   PostgreSQL    │
                                          │   (Results)     │
                                          └─────────────────┘
```

**Queue Routing:**
- `audit.crawl` → high-priority queue (CPU-bound)
- `audit.render` → render queue (Playwright workers)
- `gsc.sync` → default queue (I/O-bound)
- `serp.refresh` → default queue with rate limiting
- `links.ingest` → low-priority queue (bulk processing)

---

## 3) Sprint Implementation Plan

### Sprint 0: Foundation (3 days)
**Goal:** Add Celery infrastructure and Project module

| Task | Details | Files |
|------|---------|-------|
| Add Redis to docker-compose | Broker for Celery | `docker-compose.yml` |
| Add Celery worker + beat services | Background processing | `docker-compose.yml`, `backend/app/core/celery.py` |
| Create Project models | Project, ProjectSettings | `backend/app/models/project.py` |
| Create Project API routes | CRUD endpoints | `backend/app/api/routes/projects.py` |
| Create Project frontend pages | List, create, overview | `frontend/src/routes/_layout/projects/` |
| Add job status tracking | JobRun model for progress | `backend/app/models/job.py` |

**Deliverable:** Users can create projects, Celery processes test tasks

---

### Sprint 1: Site Audit Engine (2 weeks)
**Goal:** HTML crawl + hybrid JS rendering + issue detection

| Task | Details | Files |
|------|---------|-------|
| Create Audit models | AuditRun, CrawledPage, Issue, LinkEdge | `backend/app/models/audit.py` |
| Build HTML crawler | httpx-based async crawler | `backend/app/services/audit/crawler.py` |
| Build JS renderer | Playwright integration | `backend/app/services/audit/renderer.py` |
| Implement issue rules | Critical/High/Medium/Low rules | `backend/app/services/audit/analyzer.py` |
| Create diff logic | New/resolved issue tracking | `backend/app/services/audit/differ.py` |
| Create Celery task pipeline | Crawl → Render → Analyze → Diff | `backend/app/tasks/audit.py` |
| Build Audit API routes | Run, status, issues, pages, export | `backend/app/api/routes/audits.py` |
| Build Audit UI | Dashboard, issues table, page detail | `frontend/src/routes/_layout/projects/$projectId/audits/` |

**Deliverable:** Users can run audits and see prioritized issues

---

### Sprint 2: GSC Integration (2 weeks)
**Goal:** OAuth connect + data ingestion + keyword tooling

| Task | Details | Files |
|------|---------|-------|
| Add Google OAuth | Token storage with encryption | `backend/app/core/oauth/google.py` |
| Create GSC models | GSCProperty, GSCQueryDaily, KeywordCluster | `backend/app/models/gsc.py` |
| Build GSC client | Search Analytics API wrapper | `backend/app/services/gsc/client.py` |
| Build ingestion pipeline | Daily sync + backfill | `backend/app/services/gsc/ingestor.py` |
| Implement opportunity scoring | High impressions + low CTR logic | `backend/app/services/gsc/opportunities.py` |
| Implement query clustering | N-gram overlap clustering | `backend/app/services/gsc/clustering.py` |
| Create GSC API routes | Connect, sync, explorers | `backend/app/api/routes/gsc.py` |
| Build Keywords UI | Query/page explorers, opportunities | `frontend/src/routes/_layout/projects/$projectId/keywords/` |

**Deliverable:** Users can connect GSC and explore keyword data

---

### Sprint 3: SERP/Rank Tracking (1 week)
**Goal:** Provider-based rank tracking with SERP snapshots

| Task | Details | Files |
|------|---------|-------|
| Create SERP models | KeywordTarget, SerpSnapshot, RankObservation | `backend/app/models/serp.py` |
| Define SerpProvider interface | Input/output contract | `backend/app/services/serp/providers/base.py` |
| Implement GSC-based provider | Position from GSC data | `backend/app/services/serp/providers/gsc_based.py` |
| Build refresh task | Scheduled + on-demand refresh | `backend/app/tasks/serp.py` |
| Create SERP API routes | Keywords CRUD, observations | `backend/app/api/routes/serp.py` |
| Build Rank Tracker UI | Keyword list, trends, SERP viewer | `frontend/src/routes/_layout/projects/$projectId/rank-tracker/` |

**Deliverable:** Users can track keyword positions over time

---

### Sprint 4: Backlinks (2 weeks)
**Goal:** Common Crawl ingestion + backlink explorer

| Task | Details | Files |
|------|---------|-------|
| Create Links models | LinkSnapshot, BacklinkEdge, RefDomainAgg | `backend/app/models/links.py` |
| Build CC ingestion pipeline | Subset download + parsing | `backend/app/services/links/commoncrawl.py` |
| Build aggregation logic | Referring domains, anchors | `backend/app/services/links/aggregator.py` |
| Implement competitive queries | Overlap, intersect | `backend/app/services/links/competitive.py` |
| Create Links API routes | Domain queries, competitive | `backend/app/api/routes/links.py` |
| Build Backlinks UI | Explorer, referring domains, anchors | `frontend/src/routes/_layout/projects/$projectId/links/` |

**Deliverable:** Users can explore backlinks and competitive intel

---

### Sprint 5: PPC + Traffic (2 weeks)
**Goal:** Google Ads integration + traffic panel

| Task | Details | Files |
|------|---------|-------|
| Add Google Ads OAuth | Account linking | `backend/app/core/oauth/google.py` |
| Create Ads models | AdsAccount, AdsCampaignDaily, AdsKeywordDaily | `backend/app/models/ads.py` |
| Create Traffic models | TrafficDaily, multi-source | `backend/app/models/traffic.py` |
| Build Google Ads client | Campaigns, keywords, landing pages | `backend/app/services/ads/google_ads.py` |
| Build SEO+PPC overlap | Keyword matching | `backend/app/services/ads/overlap.py` |
| Build transparency explorer | Dataset integration | `backend/app/services/ads/transparency.py` |
| Create Ads/Traffic API routes | Dashboards, overlap, panel | `backend/app/api/routes/ads.py`, `traffic.py` |
| Build PPC UI | Dashboard, overlap, transparency | `frontend/src/routes/_layout/projects/$projectId/ppc/` |
| Build Traffic Panel UI | Multi-source dashboard | `frontend/src/routes/_layout/projects/$projectId/traffic/` |

**Deliverable:** Users can view PPC data and traffic panel

---

### Sprint 6: Hardening (1 week)
**Goal:** Polish, performance, documentation

| Task | Details |
|------|---------|
| Add rate limiting | API endpoint throttling |
| Add retention policies | Cleanup old audit runs, snapshots |
| Performance tuning | Query optimization, caching |
| Error handling polish | User-friendly error messages |
| CI/CD hardening | Integration tests, E2E tests |
| Deployment docs | Production deployment guide |

---

## 4) Cross-Cutting Concerns

### 4.1 Error Handling Pattern

```python
# backend/app/core/exceptions.py
class SEOPlatformError(Exception):
    """Base exception for SEO platform"""
    pass

class CrawlError(SEOPlatformError):
    """Raised when crawl fails"""
    pass

class ProviderError(SEOPlatformError):
    """Raised when external provider fails"""
    pass

class QuotaExceededError(SEOPlatformError):
    """Raised when quota/rate limit exceeded"""
    pass
```

### 4.2 Job Progress Tracking

```python
# backend/app/models/job.py
class JobRun(SQLModel, table=True):
    id: uuid.UUID
    job_type: str  # "audit", "gsc_sync", "serp_refresh", etc.
    project_id: uuid.UUID
    status: str  # "queued", "running", "completed", "failed"
    progress_pct: int = 0
    progress_message: str | None
    started_at: datetime | None
    finished_at: datetime | None
    error_message: str | None
    result_json: dict | None
```

### 4.3 Feature Flags

```python
# backend/app/core/config.py
class Settings(BaseSettings):
    # Feature flags
    ENABLE_JS_RENDERING: bool = True
    ENABLE_SERP_API_PROVIDER: bool = False  # Disabled by default
    ENABLE_COMMON_CRAWL_INGESTION: bool = True
    ENABLE_GOOGLE_ADS: bool = True

    # Limits
    MAX_PAGES_PER_AUDIT: int = 1000
    MAX_RENDER_PAGES_PER_AUDIT: int = 100
    MAX_KEYWORDS_PER_PROJECT: int = 500
    SERP_REFRESH_DAILY_CAP: int = 100
```

### 4.4 OAuth Token Encryption

```python
# backend/app/core/oauth/encryption.py
from cryptography.fernet import Fernet

def encrypt_token(token: str, key: bytes) -> str:
    """Encrypt OAuth token for storage"""
    f = Fernet(key)
    return f.encrypt(token.encode()).decode()

def decrypt_token(encrypted: str, key: bytes) -> str:
    """Decrypt stored OAuth token"""
    f = Fernet(key)
    return f.decrypt(encrypted.encode()).decode()
```

### 4.5 API Response Patterns

**Paginated List Response:**
```python
class PaginatedResponse(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int
    page_size: int
    has_more: bool
```

**Job Status Response:**
```python
class JobStatusResponse(BaseModel):
    job_id: uuid.UUID
    status: str
    progress_pct: int
    message: str | None
```

---

## 5) Testing Strategy

### 5.1 Backend Testing

```
tests/
├── unit/
│   ├── test_audit_rules.py      # Issue detection logic
│   ├── test_url_normalization.py
│   ├── test_opportunity_scoring.py
│   └── test_clustering.py
├── integration/
│   ├── test_audit_pipeline.py   # End-to-end audit run
│   ├── test_gsc_ingestion.py
│   └── test_serp_providers.py
└── fixtures/
    ├── crawled_pages.json
    └── gsc_responses.json
```

### 5.2 Frontend Testing

```
tests/
├── e2e/
│   ├── project-crud.spec.ts
│   ├── audit-run.spec.ts
│   ├── keyword-explorer.spec.ts
│   └── rank-tracker.spec.ts
└── fixtures/
    └── mock-api-responses.ts
```

---

## 6) File Reference Index

Detailed implementation specs for each module:

| Document | Covers |
|----------|--------|
| [IMPLEMENTATION-01-PROJECTS.md](./IMPLEMENTATION-01-PROJECTS.md) | Project CRUD, settings, schedules |
| [IMPLEMENTATION-02-AUDITS.md](./IMPLEMENTATION-02-AUDITS.md) | Crawler, renderer, analyzer, differ |
| [IMPLEMENTATION-03-GSC-KEYWORDS.md](./IMPLEMENTATION-03-GSC-KEYWORDS.md) | GSC OAuth, ingestion, explorers, clustering |
| [IMPLEMENTATION-04-SERP-RANK-TRACKING.md](./IMPLEMENTATION-04-SERP-RANK-TRACKING.md) | Provider interface, keyword targets |
| [IMPLEMENTATION-05-BACKLINKS.md](./IMPLEMENTATION-05-BACKLINKS.md) | Common Crawl, aggregation, competitive |
| [IMPLEMENTATION-06-PPC-TRAFFIC.md](./IMPLEMENTATION-06-PPC-TRAFFIC.md) | Google Ads, transparency, traffic panel |
| [IMPLEMENTATION-07-BACKGROUND-JOBS.md](./IMPLEMENTATION-07-BACKGROUND-JOBS.md) | Celery setup, task routing, scheduling |

---

## 7) Quick Start Commands

```bash
# Start development stack
docker-compose up -d

# Run backend tests
docker-compose exec backend pytest

# Run frontend tests
docker-compose exec frontend npm test

# Generate TypeScript client from OpenAPI
cd frontend && npm run generate-client

# Run Alembic migration
docker-compose exec backend alembic upgrade head

# Create new migration
docker-compose exec backend alembic revision --autogenerate -m "Add project models"
```

---

## 8) Next Steps

1. **Read each implementation doc** in order (01 → 07)
2. **Start with Sprint 0** tasks to add Celery and Projects
3. **Follow TDD** - write tests before implementation
4. **Use the generated TypeScript client** after adding new endpoints
5. **Keep PRD as source of truth** for acceptance criteria
