# API Reference

Complete API documentation for the SEO Platform REST API.

---

## Table of Contents

1. [Overview](#overview)
2. [Authentication](#authentication)
3. [Error Handling](#error-handling)
4. [Rate Limiting](#rate-limiting)
5. [Endpoints](#endpoints)
   - [Auth](#auth)
   - [Users](#users)
   - [Projects](#projects)
   - [Audits](#audits)
   - [Google Search Console](#google-search-console)
   - [SERP Tracking](#serp-tracking)
   - [Backlinks](#backlinks)
   - [PPC / Ads](#ppc--ads)
   - [Traffic](#traffic)
   - [Jobs](#jobs)
   - [Integrations](#integrations)
   - [Utilities](#utilities)

---

## Overview

### Base URL

```
Production: https://api.yourdomain.com/api/v1
Development: http://localhost:8000/api/v1
```

### Content Type

All requests and responses use JSON:

```
Content-Type: application/json
Accept: application/json
```

### Interactive Documentation

- **Swagger UI**: `https://api.yourdomain.com/docs`
- **ReDoc**: `https://api.yourdomain.com/redoc`
- **OpenAPI Schema**: `https://api.yourdomain.com/openapi.json`

---

## Authentication

### JWT Bearer Token

Most endpoints require authentication via JWT bearer token.

**Header Format**:
```
Authorization: Bearer <access_token>
```

### Obtaining a Token

```http
POST /api/v1/login/access-token
Content-Type: application/x-www-form-urlencoded

username=user@example.com&password=yourpassword
```

**Response** (200 OK):
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer"
}
```

### Token Expiration

- Default expiration: **7 days**
- Refresh by re-authenticating

---

## Error Handling

### Error Response Format

All errors follow a consistent structure:

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Project with id 'abc-123' not found",
    "details": {},
    "request_id": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

### Error Codes

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | `BAD_REQUEST` | Invalid request syntax |
| 401 | `UNAUTHORIZED` | Missing or invalid authentication |
| 403 | `FORBIDDEN` | Insufficient permissions |
| 404 | `NOT_FOUND` | Resource not found |
| 422 | `VALIDATION_ERROR` | Request validation failed |
| 429 | `RATE_LIMIT_EXCEEDED` | Too many requests |
| 500 | `INTERNAL_ERROR` | Server error |
| 502 | `EXTERNAL_SERVICE_ERROR` | Third-party API failure |

### Validation Errors

For 422 errors, FastAPI returns detailed validation info:

```json
{
  "detail": [
    {
      "loc": ["body", "email"],
      "msg": "value is not a valid email address",
      "type": "value_error.email"
    }
  ]
}
```

---

## Rate Limiting

### Limits

| Endpoint Type | Limit | Window |
|---------------|-------|--------|
| Authentication | 5 requests | 1 minute |
| Standard APIs | 100 requests | 1 minute |
| Expensive Operations | 10 requests | 1 minute |

### Rate Limit Headers

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1705312200
```

### Exceeded Response

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 45

{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Please retry after 45 seconds.",
    "details": {
      "retry_after": 45
    }
  }
}
```

---

## Endpoints

### Auth

#### Login

```http
POST /api/v1/login/access-token
```

Authenticate and receive JWT token.

**Request Body** (form-urlencoded):
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `username` | string | Yes | User email address |
| `password` | string | Yes | User password |

**Response** (200 OK):
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "bearer"
}
```

---

### Users

#### Get Current User

```http
GET /api/v1/users/me
Authorization: Bearer <token>
```

**Response** (200 OK):
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "full_name": "John Doe",
  "is_active": true,
  "is_superuser": false
}
```

#### Update Current User

```http
PATCH /api/v1/users/me
Authorization: Bearer <token>
Content-Type: application/json

{
  "full_name": "Jane Doe",
  "email": "jane@example.com"
}
```

#### Register New User

```http
POST /api/v1/users/signup
Content-Type: application/json

{
  "email": "newuser@example.com",
  "password": "securepassword123",
  "full_name": "New User"
}
```

---

### Projects

#### List Projects

```http
GET /api/v1/projects/
Authorization: Bearer <token>
```

**Query Parameters**:
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `skip` | integer | 0 | Pagination offset |
| `limit` | integer | 100 | Max results (1-100) |

**Response** (200 OK):
```json
{
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "My SEO Project",
      "seed_url": "https://example.com",
      "description": "Main website audit",
      "created_by_id": "...",
      "created_at": "2024-01-15T10:00:00Z",
      "settings": {
        "max_pages": 1000,
        "follow_external": false
      }
    }
  ],
  "count": 1
}
```

#### Create Project

```http
POST /api/v1/projects/
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "New Project",
  "seed_url": "https://example.com",
  "description": "Optional description"
}
```

**Response** (200 OK):
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "New Project",
  "seed_url": "https://example.com",
  "description": "Optional description",
  "created_by_id": "...",
  "created_at": "2024-01-15T10:00:00Z",
  "settings": {
    "max_pages": 1000,
    "follow_external": false
  }
}
```

#### Get Project

```http
GET /api/v1/projects/{project_id}
Authorization: Bearer <token>
```

#### Update Project

```http
PUT /api/v1/projects/{project_id}
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "Updated Name",
  "description": "Updated description"
}
```

#### Delete Project

```http
DELETE /api/v1/projects/{project_id}
Authorization: Bearer <token>
```

**Response** (200 OK):
```json
{
  "message": "Project deleted successfully"
}
```

---

### Audits

#### List Audits

```http
GET /api/v1/projects/{project_id}/audits/
Authorization: Bearer <token>
```

**Query Parameters**:
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `skip` | integer | 0 | Pagination offset |
| `limit` | integer | 20 | Max results |

**Response** (200 OK):
```json
{
  "data": [
    {
      "id": "...",
      "project_id": "...",
      "status": "completed",
      "started_at": "2024-01-15T10:00:00Z",
      "completed_at": "2024-01-15T10:15:00Z",
      "pages_crawled": 150,
      "issues_found": 45
    }
  ],
  "count": 5
}
```

#### Start Audit

```http
POST /api/v1/projects/{project_id}/audits/
Authorization: Bearer <token>
Content-Type: application/json

{
  "max_pages": 500,
  "render_js": false
}
```

**Response** (202 Accepted):
```json
{
  "id": "...",
  "status": "pending",
  "started_at": "2024-01-15T10:00:00Z"
}
```

#### Get Audit Details

```http
GET /api/v1/projects/{project_id}/audits/{audit_id}
Authorization: Bearer <token>
```

#### Get Audit Issues

```http
GET /api/v1/projects/{project_id}/audits/{audit_id}/issues
Authorization: Bearer <token>
```

**Query Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `severity` | string | Filter by severity: `critical`, `high`, `medium`, `low` |
| `issue_type` | string | Filter by issue type |
| `skip` | integer | Pagination offset |
| `limit` | integer | Max results |

**Response** (200 OK):
```json
{
  "data": [
    {
      "id": "...",
      "page_url": "https://example.com/page",
      "issue_type": "MISSING_META_DESCRIPTION",
      "severity": "medium",
      "element": "<head>...</head>",
      "suggestion": "Add a meta description between 150-160 characters",
      "created_at": "2024-01-15T10:05:00Z"
    }
  ],
  "count": 45
}
```

#### Get Crawled Pages

```http
GET /api/v1/projects/{project_id}/audits/{audit_id}/pages
Authorization: Bearer <token>
```

#### Compare Audits

```http
GET /api/v1/projects/{project_id}/audits/compare
Authorization: Bearer <token>
```

**Query Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `audit_id_a` | uuid | Yes | First audit ID |
| `audit_id_b` | uuid | Yes | Second audit ID |

**Response** (200 OK):
```json
{
  "audit_a": {...},
  "audit_b": {...},
  "issues": {
    "new": [...],
    "fixed": [...],
    "unchanged": [...]
  },
  "pages": {
    "added": [...],
    "removed": [...],
    "changed": [...]
  }
}
```

---

### Google Search Console

#### Get GSC Connection Status

```http
GET /api/v1/projects/{project_id}/gsc/status
Authorization: Bearer <token>
```

**Response** (200 OK):
```json
{
  "connected": true,
  "site_url": "https://example.com",
  "last_sync": "2024-01-15T08:00:00Z",
  "sync_status": "completed"
}
```

#### Connect GSC (OAuth)

```http
GET /api/v1/projects/{project_id}/gsc/connect
Authorization: Bearer <token>
```

**Response** (200 OK):
```json
{
  "authorization_url": "https://accounts.google.com/o/oauth2/v2/auth?..."
}
```

#### Sync GSC Data

```http
POST /api/v1/projects/{project_id}/gsc/sync
Authorization: Bearer <token>
Content-Type: application/json

{
  "start_date": "2024-01-01",
  "end_date": "2024-01-15"
}
```

#### Get Search Analytics

```http
GET /api/v1/projects/{project_id}/gsc/analytics
Authorization: Bearer <token>
```

**Query Parameters**:
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `start_date` | date | 28 days ago | Start of date range |
| `end_date` | date | today | End of date range |
| `dimension` | string | `query` | Group by: `query`, `page`, `country`, `device` |
| `skip` | integer | 0 | Pagination offset |
| `limit` | integer | 100 | Max results |

**Response** (200 OK):
```json
{
  "data": [
    {
      "query": "seo tools",
      "clicks": 150,
      "impressions": 2500,
      "ctr": 0.06,
      "position": 4.2
    }
  ],
  "totals": {
    "clicks": 5000,
    "impressions": 100000
  }
}
```

#### Get Quick Win Opportunities

```http
GET /api/v1/projects/{project_id}/gsc/opportunities
Authorization: Bearer <token>
```

**Query Parameters**:
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `min_impressions` | integer | 100 | Minimum impressions filter |
| `max_position` | float | 20 | Maximum average position |

**Response** (200 OK):
```json
{
  "opportunities": [
    {
      "query": "seo audit tool",
      "page": "https://example.com/features",
      "clicks": 45,
      "impressions": 1200,
      "position": 8.5,
      "opportunity_score": 0.85,
      "recommendation": "Optimize on-page content for this keyword"
    }
  ]
}
```

#### Get Query Clusters

```http
GET /api/v1/projects/{project_id}/gsc/clusters
Authorization: Bearer <token>
```

**Response** (200 OK):
```json
{
  "clusters": [
    {
      "id": "...",
      "name": "seo tools",
      "queries": ["seo tools", "best seo tools", "seo tool free"],
      "total_clicks": 500,
      "total_impressions": 8000,
      "avg_position": 6.2
    }
  ]
}
```

---

### SERP Tracking

#### List Tracked Keywords

```http
GET /api/v1/projects/{project_id}/serp/keywords
Authorization: Bearer <token>
```

#### Add Keywords

```http
POST /api/v1/projects/{project_id}/serp/keywords
Authorization: Bearer <token>
Content-Type: application/json

{
  "keywords": ["seo tools", "website audit", "rank tracker"]
}
```

#### Get Rank History

```http
GET /api/v1/projects/{project_id}/serp/keywords/{keyword_id}/history
Authorization: Bearer <token>
```

**Query Parameters**:
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `start_date` | date | 30 days ago | Start of date range |
| `end_date` | date | today | End of date range |

**Response** (200 OK):
```json
{
  "keyword": "seo tools",
  "history": [
    {
      "date": "2024-01-15",
      "position": 5,
      "url": "https://example.com/tools",
      "change": -2
    }
  ]
}
```

#### Trigger Rank Check

```http
POST /api/v1/projects/{project_id}/serp/check
Authorization: Bearer <token>
```

---

### Backlinks

#### Get Backlink Overview

```http
GET /api/v1/projects/{project_id}/links/overview
Authorization: Bearer <token>
```

**Response** (200 OK):
```json
{
  "total_backlinks": 1500,
  "referring_domains": 250,
  "dofollow_count": 1200,
  "nofollow_count": 300,
  "avg_domain_rating": 45.5,
  "last_updated": "2024-01-15T00:00:00Z"
}
```

#### List Backlinks

```http
GET /api/v1/projects/{project_id}/links/backlinks
Authorization: Bearer <token>
```

**Query Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `source_domain` | string | Filter by source domain |
| `target_page` | string | Filter by target page |
| `link_type` | string | `dofollow` or `nofollow` |
| `skip` | integer | Pagination offset |
| `limit` | integer | Max results |

#### Get Anchor Text Distribution

```http
GET /api/v1/projects/{project_id}/links/anchors
Authorization: Bearer <token>
```

**Response** (200 OK):
```json
{
  "anchors": [
    {
      "text": "click here",
      "count": 150,
      "percentage": 10.5
    },
    {
      "text": "seo tools",
      "count": 100,
      "percentage": 7.0
    }
  ]
}
```

#### Discover New Backlinks

```http
POST /api/v1/projects/{project_id}/links/discover
Authorization: Bearer <token>
```

Triggers Common Crawl backlink discovery task.

#### Competitor Backlink Gap

```http
POST /api/v1/projects/{project_id}/links/competitive
Authorization: Bearer <token>
Content-Type: application/json

{
  "competitor_domains": ["competitor1.com", "competitor2.com"]
}
```

**Response** (200 OK):
```json
{
  "gap_analysis": [
    {
      "domain": "high-authority-site.com",
      "your_backlinks": 0,
      "competitor1_backlinks": 3,
      "competitor2_backlinks": 2,
      "domain_rating": 75,
      "opportunity": "high"
    }
  ]
}
```

---

### PPC / Ads

#### Get Ads Connection Status

```http
GET /api/v1/projects/{project_id}/ads/status
Authorization: Bearer <token>
```

#### Connect Google Ads

```http
GET /api/v1/projects/{project_id}/ads/connect
Authorization: Bearer <token>
```

#### Sync Ads Data

```http
POST /api/v1/projects/{project_id}/ads/sync
Authorization: Bearer <token>
```

#### Get Campaigns

```http
GET /api/v1/projects/{project_id}/ads/campaigns
Authorization: Bearer <token>
```

**Query Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `start_date` | date | Start of date range |
| `end_date` | date | End of date range |
| `status` | string | `active`, `paused`, `removed` |

**Response** (200 OK):
```json
{
  "data": [
    {
      "campaign_id": "123456789",
      "campaign_name": "Brand Campaign",
      "status": "active",
      "budget_micros": 1000000000,
      "clicks": 500,
      "impressions": 10000,
      "cost_micros": 250000000,
      "conversions": 25
    }
  ],
  "totals": {
    "clicks": 1500,
    "cost_micros": 750000000
  }
}
```

#### Get SEO + PPC Overlap

```http
GET /api/v1/projects/{project_id}/ads/seo-overlap
Authorization: Bearer <token>
```

**Query Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `start_date` | date | Start of date range |
| `end_date` | date | End of date range |
| `overlap_type` | string | `both`, `paid_only`, `organic_only` |

**Response** (200 OK):
```json
{
  "data": [
    {
      "keyword": "seo tools",
      "organic_position": 5.2,
      "organic_clicks": 150,
      "paid_clicks": 200,
      "paid_cost_micros": 50000000,
      "total_clicks": 350,
      "overlap_type": "both",
      "opportunity_score": 0.75,
      "recommendation": "Consider reducing PPC spend on this keyword"
    }
  ],
  "summary": {
    "total_keywords": 500,
    "overlap_count": 150,
    "paid_only_count": 200,
    "organic_only_count": 150
  }
}
```

---

### Traffic

#### Get Traffic Overview

```http
GET /api/v1/projects/{project_id}/traffic/overview
Authorization: Bearer <token>
```

**Query Parameters**:
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `start_date` | date | 30 days ago | Start of date range |
| `end_date` | date | today | End of date range |
| `source` | string | `all` | Filter: `organic`, `paid`, `all` |

**Response** (200 OK):
```json
{
  "data": [
    {
      "date": "2024-01-15",
      "organic_clicks": 500,
      "paid_clicks": 200,
      "total_sessions": 800,
      "conversions": 25
    }
  ],
  "totals": {
    "organic_clicks": 15000,
    "paid_clicks": 6000,
    "total_sessions": 24000
  }
}
```

#### Import Traffic Data (CSV)

```http
POST /api/v1/projects/{project_id}/traffic/import
Authorization: Bearer <token>
Content-Type: multipart/form-data

file: <traffic_data.csv>
source: "google_analytics"
```

---

### Jobs

#### List Background Jobs

```http
GET /api/v1/jobs/
Authorization: Bearer <token>
```

**Query Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `status` | string | `pending`, `running`, `completed`, `failed` |
| `job_type` | string | `audit`, `gsc_sync`, `serp_check`, etc. |

#### Get Job Status

```http
GET /api/v1/jobs/{job_id}
Authorization: Bearer <token>
```

**Response** (200 OK):
```json
{
  "id": "...",
  "job_type": "audit",
  "status": "running",
  "progress": 45,
  "started_at": "2024-01-15T10:00:00Z",
  "message": "Crawling page 45 of 100"
}
```

#### Cancel Job

```http
POST /api/v1/jobs/{job_id}/cancel
Authorization: Bearer <token>
```

---

### Integrations

#### List Integrations

```http
GET /api/v1/projects/{project_id}/integrations/
Authorization: Bearer <token>
```

**Response** (200 OK):
```json
{
  "integrations": [
    {
      "type": "google_search_console",
      "connected": true,
      "site_url": "https://example.com",
      "last_sync": "2024-01-15T08:00:00Z"
    },
    {
      "type": "google_ads",
      "connected": false
    }
  ]
}
```

#### Disconnect Integration

```http
DELETE /api/v1/projects/{project_id}/integrations/{integration_type}
Authorization: Bearer <token>
```

---

### Utilities

#### Health Check

```http
GET /api/v1/utils/health-check/
```

No authentication required.

**Response** (200 OK):
```json
{
  "status": "healthy"
}
```

#### Test Email

```http
POST /api/v1/utils/test-email/
Authorization: Bearer <token>
Content-Type: application/json

{
  "email_to": "test@example.com"
}
```

---

## Pagination

All list endpoints support pagination:

```http
GET /api/v1/projects/?skip=0&limit=20
```

**Response Format**:
```json
{
  "data": [...],
  "count": 100
}
```

| Parameter | Type | Default | Max | Description |
|-----------|------|---------|-----|-------------|
| `skip` | integer | 0 | - | Number of records to skip |
| `limit` | integer | 20-100 | 100 | Max records to return |

---

## Filtering and Sorting

Many endpoints support filtering via query parameters:

```http
GET /api/v1/projects/{project_id}/audits/{audit_id}/issues?severity=high&issue_type=MISSING_H1
```

Sorting is typically by relevance or date (descending).

---

## Webhooks (Coming Soon)

Future support for webhooks on events:
- Audit completed
- New backlinks discovered
- Rank changes detected

---

## SDKs and Client Libraries

### Auto-Generated TypeScript Client

The frontend uses an auto-generated client:

```bash
cd frontend
npm run generate-client
```

Usage:
```typescript
import { ProjectsService, AuditsService } from '@/client'

// List projects
const projects = await ProjectsService.readProjects()

// Start audit
const audit = await AuditsService.startAudit({
  projectId: 'xxx',
  requestBody: { max_pages: 500 }
})
```

---

**Next**: [Deployment Guide](./deployment.md) | [Architecture](./architecture.md)
