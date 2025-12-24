# Architecture Guide

Comprehensive technical architecture documentation for the SEO Platform.

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Technology Stack](#technology-stack)
3. [Service Architecture](#service-architecture)
4. [Data Architecture](#data-architecture)
5. [Security Architecture](#security-architecture)
6. [Integration Patterns](#integration-patterns)
7. [Scalability Considerations](#scalability-considerations)

---

## System Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              INTERNET                                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           TRAEFIK REVERSE PROXY                              │
│  • SSL/TLS Termination  • Load Balancing  • Service Discovery               │
└─────────────────────────────────────────────────────────────────────────────┘
                    │                               │
                    ▼                               ▼
┌───────────────────────────────┐   ┌───────────────────────────────┐
│         FRONTEND              │   │          BACKEND               │
│  ┌─────────────────────────┐  │   │  ┌─────────────────────────┐  │
│  │   React + TanStack      │  │   │  │   FastAPI Application   │  │
│  │   • shadcn/ui           │  │   │  │   • REST API            │  │
│  │   • TailwindCSS         │  │   │  │   • Authentication      │  │
│  │   • TypeScript          │  │   │  │   • Business Logic      │  │
│  └─────────────────────────┘  │   │  └─────────────────────────┘  │
│   dashboard.yourdomain.com    │   │   api.yourdomain.com          │
└───────────────────────────────┘   └───────────────────────────────┘
                                                    │
                    ┌───────────────────────────────┼───────────────────────────┐
                    │                               │                           │
                    ▼                               ▼                           ▼
┌───────────────────────────┐   ┌───────────────────────────┐   ┌───────────────────────────┐
│       POSTGRESQL          │   │          REDIS            │   │     CELERY WORKERS        │
│  ┌─────────────────────┐  │   │  ┌─────────────────────┐  │   │  ┌─────────────────────┐  │
│  │   Primary Database  │  │   │  │   Cache Layer       │  │   │  │   Task Processing   │  │
│  │   • User Data       │  │   │  │   • API Caching     │  │   │  │   • Audits          │  │
│  │   • Projects        │  │   │  │   • Rate Limiting   │  │   │  │   • GSC Sync        │  │
│  │   • Audit Results   │  │   │  │   • Session Store   │  │   │  │   • SERP Tracking   │  │
│  │   • Analytics       │  │   │  │   • Task Queue      │  │   │  │   • Link Analysis   │  │
│  └─────────────────────┘  │   │  └─────────────────────┘  │   │  │   • PPC Sync        │  │
└───────────────────────────┘   └───────────────────────────┘   │  └─────────────────────┘  │
                                                                └───────────────────────────┘
                                                                            │
                                    ┌───────────────────────────────────────┘
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         EXTERNAL SERVICES                                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │  Google     │  │  Google     │  │  Google     │  │  Common     │         │
│  │  OAuth 2.0  │  │  Search     │  │  Ads API    │  │  Crawl      │         │
│  │             │  │  Console    │  │             │  │  Index      │         │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Request Flow

1. **User Request**: Browser sends request to frontend (React SPA)
2. **API Call**: Frontend calls backend REST API via generated TypeScript client
3. **Authentication**: JWT token validated; user context established
4. **Business Logic**: Route handler invokes services
5. **Data Layer**: Services interact with PostgreSQL via SQLModel ORM
6. **Caching**: Frequently accessed data cached in Redis
7. **Background Tasks**: Long-running operations queued to Celery workers
8. **External APIs**: Workers integrate with Google APIs, Common Crawl

---

## Technology Stack

### Backend

| Component | Technology | Version | Purpose |
|-----------|------------|---------|---------|
| **Framework** | FastAPI | 0.110+ | High-performance async web framework |
| **ORM** | SQLModel | 0.0.16+ | Pydantic + SQLAlchemy integration |
| **Database** | PostgreSQL | 17 | Primary relational database |
| **Cache** | Redis | 7+ | Caching, rate limiting, task queue |
| **Task Queue** | Celery | 5.3+ | Distributed task processing |
| **HTTP Client** | httpx | 0.27+ | Async HTTP requests |
| **Auth** | python-jose | - | JWT token handling |
| **Password** | passlib[bcrypt] | - | Secure password hashing |

### Frontend

| Component | Technology | Version | Purpose |
|-----------|------------|---------|---------|
| **Framework** | React | 18+ | UI component library |
| **Routing** | TanStack Router | 1.x | Type-safe file-based routing |
| **State** | TanStack Query | 5.x | Server state management |
| **Forms** | React Hook Form | 7.x | Form handling and validation |
| **UI Kit** | shadcn/ui | - | Accessible component primitives |
| **Styling** | TailwindCSS | 3.x | Utility-first CSS framework |
| **Build** | Vite | 5.x | Fast build tooling |
| **Language** | TypeScript | 5.x | Type-safe JavaScript |

### Infrastructure

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Reverse Proxy** | Traefik | Service discovery, SSL, load balancing |
| **Containerization** | Docker | Application packaging |
| **Orchestration** | Docker Compose | Multi-container management |
| **CI/CD** | GitHub Actions | Automated testing and deployment |

---

## Service Architecture

### Backend Services

The backend follows a layered architecture with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────────┐
│                        API LAYER                                 │
│  app/api/routes/*.py                                             │
│  • Route definitions  • Request validation  • Response models    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     SERVICE LAYER                                │
│  app/services/*/                                                 │
│  • Business logic  • External API integration  • Algorithms      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      DATA LAYER                                  │
│  app/models/*.py + app/crud.py                                   │
│  • SQLModel models  • CRUD operations  • Relationships           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CORE UTILITIES                                │
│  app/core/                                                       │
│  • Config  • Security  • Caching  • Rate Limiting  • OAuth       │
└─────────────────────────────────────────────────────────────────┘
```

### Service Modules

#### Site Audit Engine (`app/services/audit/`)

```
audit/
├── crawler.py      # Async web crawler with breadth-first traversal
├── renderer.py     # JavaScript rendering via Playwright (optional)
├── analyzer.py     # SEO issue detection (50+ rule types)
└── differ.py       # Audit-to-audit comparison
```

**Data Flow**:
```
URL → Crawler → Raw HTML → Analyzer → Issues → Database
                    │
                    └→ Renderer (if JS needed) → Rendered HTML
```

#### GSC Integration (`app/services/gsc/`)

```
gsc/
├── client.py          # Google Search Console API wrapper
├── ingestor.py        # Data sync from GSC to local database
├── opportunities.py   # Quick-win keyword identification
└── clustering.py      # N-gram based query clustering
```

**OAuth Flow**:
```
User → OAuth Consent → Callback → Token Encrypted → Stored
                                        │
                                        ▼
                            Decrypt → GSC API → Data → Database
```

#### SERP Tracking (`app/services/serp/`)

```
serp/
├── providers/
│   ├── base.py          # Abstract provider interface
│   └── gsc_provider.py  # GSC-based position data
└── tracker.py           # Rank snapshot management
```

**Provider Pattern**:
```python
class SERPProvider(ABC):
    @abstractmethod
    async def fetch_rankings(self, keywords: list[str], domain: str) -> list[RankSnapshot]:
        pass
```

#### Backlink Analysis (`app/services/links/`)

```
links/
├── commoncrawl.py   # Common Crawl index querying (free backlinks)
├── aggregator.py    # Backlink aggregation and metrics
└── competitive.py   # Competitor backlink comparison
```

**Processing Pipeline**:
```
Query → Common Crawl API → WARC Records → Parse → Backlinks → Aggregate
```

#### PPC Integration (`app/services/ads/`)

```
ads/
├── google_ads.py    # Google Ads API client
└── overlap.py       # SEO + PPC keyword overlap analysis
```

#### Traffic Panel (`app/services/traffic/`)

```
traffic/
└── panel.py         # Unified traffic metrics from multiple sources
```

### Celery Task Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     CELERY BROKER (Redis)                        │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│   WORKER 1    │   │   WORKER 2    │   │   WORKER 3    │
│ (audit tasks) │   │ (gsc tasks)   │   │ (links tasks) │
└───────────────┘   └───────────────┘   └───────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      CELERY BEAT (Scheduler)                     │
│  • Nightly GSC sync    • Weekly rank checks   • Data retention   │
└─────────────────────────────────────────────────────────────────┘
```

**Task Modules**:
- `app/tasks/audit.py` - Site audit execution
- `app/tasks/gsc.py` - GSC data synchronization
- `app/tasks/serp.py` - SERP tracking snapshots
- `app/tasks/links.py` - Backlink discovery
- `app/tasks/ads.py` - Google Ads sync
- `app/tasks/maintenance.py` - Data cleanup, retention policies

---

## Data Architecture

### Entity Relationship Diagram

```
┌─────────────┐       ┌──────────────┐       ┌───────────────┐
│    USER     │1─────N│   PROJECT    │1─────N│   AUDIT_RUN   │
│             │       │              │       │               │
│ id          │       │ id           │       │ id            │
│ email       │       │ name         │       │ status        │
│ password    │       │ seed_url     │       │ started_at    │
│ is_active   │       │ created_by   │       │ completed_at  │
│ is_superuser│       │ settings     │       │ error_message │
└─────────────┘       └──────────────┘       └───────────────┘
                              │                      │
                              │1                     │1
                              │                      │
                              │N                     │N
                      ┌───────┴───────┐      ┌───────┴───────┐
                      │               │      │               │
              ┌───────────────┐ ┌───────────────┐ ┌───────────────┐
              │ GSC_PROPERTY  │ │ CRAWLED_PAGE  │ │ AUDIT_ISSUE   │
              │               │ │               │ │               │
              │ site_url      │ │ url           │ │ issue_type    │
              │ access_token  │ │ status_code   │ │ severity      │
              │ refresh_token │ │ response_time │ │ element       │
              └───────────────┘ │ html_content  │ │ suggestion    │
                      │         └───────────────┘ └───────────────┘
                      │1
                      │
                      │N
              ┌───────────────┐
              │ GSC_DAILY_ROW │
              │               │
              │ date          │
              │ query         │
              │ page          │
              │ clicks        │
              │ impressions   │
              │ position      │
              └───────────────┘
```

### Model Hierarchy

```
app/models/
├── __init__.py       # Model registry and exports
├── user.py           # User, UserCreate, UserUpdate, UserPublic
├── project.py        # Project, ProjectCreate, ProjectUpdate, ProjectPublic
├── audit.py          # AuditRun, CrawledPage, AuditIssue
├── gsc.py            # GSCProperty, GSCDailyRow, GSCCluster
├── serp.py           # TrackedKeyword, RankSnapshot
├── links.py          # Backlink, BacklinkAnchor, CompetitorBacklinks
├── ads.py            # AdsAccount, AdsCampaign, AdsCampaignDaily
├── job.py            # BackgroundJob (generic task tracking)
└── integration.py    # Integration base for OAuth connections
```

### Indexing Strategy

```python
# Performance-critical indexes
class GSCDailyRow(SQLModel, table=True):
    # Composite index for time-series queries
    __table_args__ = (
        Index("ix_gsc_daily_project_date", "project_id", "date"),
        Index("ix_gsc_daily_query", "query"),
    )

class CrawledPage(SQLModel, table=True):
    url: str = Field(index=True)  # URL lookups
    audit_run_id: UUID = Field(index=True)  # Audit filtering

class AuditIssue(SQLModel, table=True):
    __table_args__ = (
        Index("ix_audit_issue_type_severity", "issue_type", "severity"),
    )
```

### Data Retention

Configured in `app/core/config.py`:

```python
class Settings:
    # Retention periods (days)
    RETENTION_AUDIT_DAYS: int = 90
    RETENTION_GSC_DAYS: int = 365
    RETENTION_SERP_DAYS: int = 180
    RETENTION_BACKLINKS_DAYS: int = 365
```

Enforcement via scheduled Celery task (`app/tasks/maintenance.py`).

---

## Security Architecture

### Authentication Flow

```
┌─────────┐                 ┌─────────┐                 ┌─────────┐
│ Browser │                 │ Backend │                 │   DB    │
└────┬────┘                 └────┬────┘                 └────┬────┘
     │                           │                           │
     │  POST /login (email, pw)  │                           │
     │──────────────────────────>│                           │
     │                           │  SELECT user WHERE email  │
     │                           │──────────────────────────>│
     │                           │           user            │
     │                           │<──────────────────────────│
     │                           │                           │
     │                           │ verify(pw, user.hashed_pw)│
     │                           │                           │
     │    JWT access_token       │                           │
     │<──────────────────────────│                           │
     │                           │                           │
     │  GET /api/v1/me           │                           │
     │  Authorization: Bearer JWT│                           │
     │──────────────────────────>│                           │
     │                           │ decode_jwt(token)         │
     │                           │ SELECT user WHERE id      │
     │                           │──────────────────────────>│
     │                           │           user            │
     │                           │<──────────────────────────│
     │         user data         │                           │
     │<──────────────────────────│                           │
```

### Token Security

**JWT Configuration** (`app/core/security.py`):
```python
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 60 * 24 * 7  # 7 days

def create_access_token(subject: str | Any) -> str:
    expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode = {"exp": expire, "sub": str(subject)}
    return jwt.encode(to_encode, settings.SECRET_KEY, algorithm=ALGORITHM)
```

### OAuth Token Encryption

Google OAuth tokens are encrypted at rest using Fernet symmetric encryption:

```python
# app/core/oauth/google.py
from cryptography.fernet import Fernet

def encrypt_token(token: str) -> str:
    fernet = Fernet(settings.TOKEN_ENCRYPTION_KEY)
    return fernet.encrypt(token.encode()).decode()

def decrypt_token(encrypted: str) -> str:
    fernet = Fernet(settings.TOKEN_ENCRYPTION_KEY)
    return fernet.decrypt(encrypted.encode()).decode()
```

### Rate Limiting

Implemented via SlowAPI + Redis backend:

```python
# app/core/rate_limit.py
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(
    key_func=get_remote_address,
    storage_uri=settings.REDIS_URL,
)

# Endpoint-specific limits
AUTH_LIMIT = "5/minute"      # Login attempts
STANDARD_LIMIT = "100/minute" # Regular API calls
STRICT_LIMIT = "10/minute"    # Expensive operations (audits)
```

### Authorization Patterns

```python
# Role-based access control
from app.api.deps import get_current_user, get_current_superuser

@router.get("/users/")
def list_users(user: User = Depends(get_current_superuser)):
    # Only superusers can list all users
    pass

# Resource-based access control
@router.get("/projects/{project_id}")
def get_project(
    project_id: UUID,
    db: Session = Depends(get_db),
    user: User = Depends(get_current_user),
):
    project = db.get(Project, project_id)
    if project.created_by_id != user.id and not user.is_superuser:
        raise AuthorizationError("access this project")
    return project
```

---

## Integration Patterns

### Google Search Console

```python
# OAuth 2.0 Flow
1. User clicks "Connect GSC"
2. Redirect to Google OAuth consent screen
3. User grants access
4. Callback receives authorization code
5. Exchange code for access_token + refresh_token
6. Encrypt tokens and store in database

# Data Sync Flow
1. Celery task triggered (manual or scheduled)
2. Decrypt access token
3. Check token expiry; refresh if needed
4. Query GSC API with pagination
5. Upsert data to local database
6. Update sync timestamp
```

### Google Ads

```python
# Authentication: Service Account + Customer ID
1. Store refresh token (encrypted)
2. Initialize GoogleAdsClient with credentials
3. Query campaigns, ad groups, keywords
4. Store daily metrics in AdsCampaignDaily
```

### Common Crawl

```python
# Backlink Discovery
1. Query Common Crawl Index API for domain
2. Download WARC records for matching URLs
3. Parse HTML to extract backlinks
4. Deduplicate and aggregate
5. Store in Backlink model
```

---

## Scalability Considerations

### Horizontal Scaling

```
                    ┌─────────────────┐
                    │  Load Balancer  │
                    │    (Traefik)    │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│   Backend 1   │   │   Backend 2   │   │   Backend 3   │
│  (Stateless)  │   │  (Stateless)  │   │  (Stateless)  │
└───────────────┘   └───────────────┘   └───────────────┘
        │                    │                    │
        └────────────────────┼────────────────────┘
                             │
                    ┌────────┴────────┐
                    │                 │
                    ▼                 ▼
            ┌───────────────┐ ┌───────────────┐
            │  PostgreSQL   │ │    Redis      │
            │   (Primary)   │ │   (Cluster)   │
            └───────────────┘ └───────────────┘
```

**Scaling Guidelines**:

| Component | Scaling Method | Notes |
|-----------|----------------|-------|
| Backend | Horizontal | Stateless; add more replicas |
| Celery Workers | Horizontal | Add workers per task type |
| PostgreSQL | Vertical first | Consider read replicas for reads |
| Redis | Cluster mode | For high throughput caching |
| Frontend | CDN | Static assets via CloudFront/Cloudflare |

### Performance Optimizations

1. **Database**:
   - Connection pooling (SQLAlchemy pool)
   - Proper indexing on query columns
   - Eager loading to prevent N+1 queries

2. **Caching**:
   - Redis for expensive computations
   - Function-level caching with TTL
   - Cache invalidation on data changes

3. **Async Processing**:
   - All HTTP calls use async httpx
   - Database queries can use async SQLAlchemy
   - Concurrent task execution with Celery

4. **Rate Limiting**:
   - Protect against abuse
   - Prevent external API quota exhaustion

---

## Monitoring and Observability

### Health Checks

```python
# Backend health endpoint
GET /api/v1/utils/health-check/

# Returns:
{
  "status": "healthy",
  "database": "connected",
  "redis": "connected",
  "celery": "responsive"
}
```

### Logging

Structured logging with correlation IDs:

```python
import logging
import structlog

logger = structlog.get_logger()

@router.get("/projects/{project_id}")
def get_project(project_id: UUID):
    logger.info("Fetching project", project_id=str(project_id))
    # ...
```

### Error Tracking

Sentry integration for production errors:

```python
# app/main.py
import sentry_sdk

if settings.SENTRY_DSN:
    sentry_sdk.init(
        dsn=settings.SENTRY_DSN,
        environment=settings.ENVIRONMENT,
        traces_sample_rate=0.1,
    )
```

### Metrics

Flower dashboard for Celery metrics:
- Task success/failure rates
- Queue depths
- Worker utilization

---

## Deployment Environments

| Environment | Purpose | Database | Features |
|-------------|---------|----------|----------|
| **local** | Development | PostgreSQL (Docker) | Hot reload, debug mode |
| **staging** | Testing | PostgreSQL (shared) | Production-like config |
| **production** | Live | PostgreSQL (managed) | SSL, monitoring, backups |

---

**Next**: [API Reference](./api-reference.md) | [Deployment Guide](./deployment.md)
