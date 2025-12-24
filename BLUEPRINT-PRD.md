# PRD: MVP SEO Platform (Audits + Keywords + SERP + Links + PPC)

> **Base:** fastapi/full-stack-fastapi-template (FastAPI + SQLModel + Postgres + React/Vite + Tailwind/shadcn + generated TS client + Docker Compose + CI)  
> **Backend:** FastAPI (API) + Celery (jobs) + Redis (broker) + Postgres (core data)  
> **Optional storage (MVP+):** ClickHouse for high-volume link edges / time-series rollups (recommended once data grows)

---

## 1) Summary

### Product
An SEO platform MVP that helps teams improve search performance by combining:

1) **Technical audits** (crawl → detect issues → prioritize fixes → track changes over time)  
2) **Keyword tooling** (Google Search Console ingestion → trends → opportunities → clustering → alerts)  
3) **Rank tracking / SERP snapshots** (provider-based, pluggable; compliant-by-default)  
4) **Backlink index + competitive intelligence** (open web via Common Crawl ingestion)  
5) **PPC + Ads transparency + Traffic panel** (connectors-based: owned accounts first, open datasets where possible)

### MVP positioning
“**Screaming Frog-lite + GSC dashboard + rank tracking + link explorer + PPC reporting**” designed for:
- multi-project teams
- scheduled refreshes
- pipelines/background jobs
- simple, high-signal insights

---

## 2) Goals and Non-goals

### Goals (MVP)
**Core platform**
- Auth + users + (optional) orgs/teams
- Projects with schedules, settings, and data retention
- Background processing with reliable retries, throttling, and progress

**Audits**
- Run audits on demand + on schedule
- Support **hybrid crawling**: HTML-first with **optional JS rendering** via Playwright for selected pages
- Prioritized issues, per-page evidence, diff vs prior run, CSV export, basic report pages

**Keyword tooling (GSC-first)**
- OAuth connect to Google
- Link a GSC property to a project
- Ingest daily query/page performance into snapshots
- Query/page explorers, opportunity scoring, clustering, alerts

**Rank tracking / SERP**
- Track a list of keywords per project and observe positions over time
- **Provider-based** SERP retrieval (default providers are compliant)
- SERP snapshots stored for reproducibility (top N results + metadata)

**Backlinks + competitive**
- Ship a working backlink explorer from an **open web corpus** (Common Crawl ingestion)
- Referring domains, anchors, new/lost links (between crawl snapshots)
- Competitive overlap: “link intersect” and shared referring domains

**PPC + transparency + traffic**
- Owned PPC reporting via Google Ads connector (campaigns, ad groups, keywords)
- Competitor ad library explorer via Ads Transparency datasets (where available) or pluggable provider
- Traffic panel: BYO (GA4 or CSV) + proxies (GSC clicks + CrUX UX), plus optional third-party estimate adapter

### Non-goals (MVP)
- “Global” keyword database like Semrush at full market scale
- Clickstream-based marketwide traffic estimates without BYO or licensed providers
- Full-fledged content writing/generation suite
- Full browser automation designed to circumvent bot protections or violate ToS (see Compliance)

---

## 3) Compliance & Risk Posture (MVP Policy)

### SERP / rank tracking
- The system is **provider-based**. The OSS core ships with **compliant providers** only.
- Any provider that queries search engines directly outside permitted interfaces MUST be:
  - disabled by default
  - documented as “bring-your-own-provider; ensure legal/ToS compliance”
  - isolated behind an interface (no “how to bypass detection” guidance)

### Crawling
- Audit crawler respects robots.txt by default (configurable per project).
- JS rendering is supported for site audits of user-owned/authorized sites.

### Ads & data sources
- PPC reporting is for **owned accounts** via official APIs.
- Competitor ads exploration uses officially published transparency datasets where available.

---

## 4) Target Users & Personas

1) **Founder/Marketer**
- wants “what to fix next”, “what keywords can we win”, “are we improving?”

2) **SEO Specialist**
- wants recurring audits, issue diffs, query slicing, SERP snapshots, link intersect

3) **Engineer/Dev**
- wants reproducible issue evidence, exports, stable endpoints, predictable crawling behavior

---

## 5) UX Flows (MVP)

### Flow A: Create Project → Run Audit
1) Create Project (seed URL + settings + schedules)
2) Click **Run Audit**
3) Status: Queued → Crawling (HTML) → Rendering (optional) → Analyzing → Complete
4) Results:
- summary cards
- issue tables + filters + export
- per-page detail with evidence and “rendered vs non-rendered” comparison (if applicable)
- diff vs last run

### Flow B: Connect Google → Link GSC property → Keyword insights
1) Connect Google account (OAuth)
2) Select/link GSC property for project
3) Initial sync (default last 90 days + optional backfill)
4) Explore:
- queries explorer
- pages explorer
- opportunities
- clusters
- alerts

### Flow C: Add Keywords → Rank Tracking (Provider-based)
1) Add keyword targets (keyword, locale, device, engine/provider)
2) Run now / schedule
3) View position trends + SERP snapshot viewer (top N results)

### Flow D: Backlinks + Competitive
1) Enter domain or pick a project domain
2) View:
- referring domains
- backlinks list
- anchors
- new/lost links (between Common Crawl snapshots ingested)
3) Competitive:
- competitor overlap
- link intersect

### Flow E: PPC + Transparency + Traffic
1) Connect Ads (owned account)
2) View PPC dashboards + landing page mapping
3) Optional: Transparency explorer for competitor creatives
4) Traffic panel:
- GA4 sessions (if connected) OR CSV import
- GSC clicks + CrUX UX (origin-level)
- optional third-party estimate provider

---

## 6) Functional Requirements

### 6.1 Projects
- CRUD Projects
- Settings:
  - `seed_url`
  - scope (subdomains, query params)
  - crawl limits (max pages, depth, concurrency)
  - user-agent
  - robots.txt respect (default on)
  - include/exclude regex patterns
  - schedules:
    - audit frequency (off/daily/weekly)
    - GSC sync frequency (daily default)
    - SERP refresh frequency (daily/weekly)
    - backlinks refresh (depends on ingestion cadence)
  - retention policy (keep last N audit runs, last N days SERP snapshots)

**Acceptance**
- Project list shows last audit, last GSC sync, last SERP refresh, last link snapshot, last PPC sync.

---

### 6.2 Site Audit Engine (HTML + Hybrid JS)

#### Crawl modes
- **HTML-only (default)**
- **Hybrid (MVP target):** HTML-first; render with Playwright when:
  - HTML body is “thin/empty” (below threshold)
  - critical selectors missing (optional configurable CSS selectors)
  - SPA detection heuristic triggers (e.g., only root div + scripts)

- **JS-only** (optional per project)

#### Render budgets (MVP)
- max rendered pages per run
- max render time per page
- render concurrency separate from HTML concurrency

#### Collected fields (per page)
- final URL after redirects
- status code
- content type
- title, meta description
- canonical
- H1 count + first H1
- word count (basic)
- meta robots (index/noindex)
- response time
- internal outlinks + broken internal links
- (Hybrid) rendered HTML snapshot hash + extracted fields

#### Issue rules (MVP)
**Critical**
- 5xx pages
- redirect loops / excessive redirect chains
- broken internal links (targets 4xx/5xx)

**High**
- internal 4xx pages
- missing title
- duplicate title (exact duplicates)
- missing meta description (configurable: high or medium)

**Medium**
- title too long/short
- meta description too long/short
- missing H1 / multiple H1s
- canonical issues (missing self canonical; canonical to different domain)
- non-https URLs (if project is https)

**Low**
- thin content (word count below threshold)
- orphan pages (optional: sitemap-discovered but not internally linked)

#### Outputs
- Audit summary dashboard
- Issues table (filter/sort/paginate/export)
- Pages table (filter/sort/paginate)
- Page detail view with evidence + raw snippets
- Diff vs last run (new/resolved, deltas)

**Acceptance**
- Audit runs reliably in background.
- Hybrid rendering works and is visible in page evidence.
- Users can export issues CSV from filtered views.

---

### 6.3 Keyword Tooling (GSC-first)

#### OAuth + linking
- Connect Google account
- List accessible GSC properties
- Link 1 property to a project (MVP)
- Secure token storage (encrypted)

#### Ingestion
- Pull search analytics:
  - dimensions: query, page
  - optional: country, device
  - metrics: clicks, impressions, ctr, position
- Default windows:
  - last 28 days
  - last 90 days
- Backfill task:
  - chunked date ranges
  - throttling + retry/backoff
- Store daily snapshots (for trending)

#### Dashboards
**Query explorer**
- substring search
- sort by clicks/impressions/ctr/position change
- compare 28d vs prior 28d

**Page explorer**
- top pages
- biggest gainers/losers

**Opportunities**
- high impressions + low CTR
- position 8–20 + high impressions (“quick wins”)
- risers/fallers

**Clustering (MVP)**
- normalization + tokenization
- n-gram overlap clustering
- aggregate metrics per cluster

**Acceptance**
- Connect/link works end-to-end.
- Ingestion is scheduled and respects quotas.
- Opportunities and clusters render meaningful results.

---

### 6.4 Rank Tracking / SERP Snapshots (Provider-Based)

#### Provider interface (MVP)
Define a `SerpProvider` contract:
- input: keyword + locale + device + engine + depth (top N)
- output: ordered results list with url/title/snippet/domain + timestamp + provider metadata
- errors: rate limit, captcha/blocked, provider outage (handled gracefully)

#### Default providers (MVP)
- **GSC-based “rank trend”** (average position for owned properties)
- **Optional compliant snapshot provider** (e.g., approved search API) behind configuration

#### Keyword targets
- User-defined keyword list per project
- Attributes:
  - keyword text
  - locale (country/language)
  - device (desktop/mobile)
  - engine/provider
  - schedule

#### UI
- Keyword list with:
  - latest position
  - change vs last period
  - sparkline/trend
- SERP snapshot viewer:
  - top N results
  - highlight your domain presence

**Acceptance**
- User can add keywords and see stored observations over time.
- Providers are swappable without changing core logic.
- Failures don’t break the product; show partial data and logs.

---

### 6.5 Backlinks + Competitive Intelligence (Common Crawl-Based)

#### Ingestion (MVP)
- Import link edges from Common Crawl datasets (start with a subset / latest monthly snapshot)
- Extract/store:
  - source_url, target_url
  - source_domain, target_domain
  - anchor text (if available)
  - nofollow/sponsored/ugc flags (if parseable)
  - first_seen, last_seen (by snapshot)

#### Metrics & views
- Referring domains (count, new/lost)
- Backlinks list (filter by domain, anchor, nofollow)
- Anchors summary (top anchors)
- New/lost links between ingested snapshots

#### Competitive views
- Competitor overlap (shared referring domains)
- Link intersect (domains linking to competitors but not you)

#### Storage
- MVP can start in Postgres for small subsets.
- Add ClickHouse when edges grow (recommended for performance).

**Acceptance**
- Domain input returns real referring domains and backlinks.
- New/lost works across at least two ingested snapshots.
- Competitive overlap and intersect are computed.

---

### 6.6 PPC + Transparency + Traffic Panel

#### PPC (owned accounts)
- Google Ads connector (OAuth)
- Ingest:
  - campaigns, ad groups, keywords, metrics
  - landing page mapping (final URL)
- Dashboard:
  - top campaigns
  - top paid keywords
  - landing pages + spend/clicks

#### SEO + PPC overlap
- Match paid keywords with GSC queries
- Show overlap and gaps:
  - “high paid spend, low organic clicks”
  - “high organic opportunity, no paid coverage”

#### Competitor ads (Transparency)
- Transparency explorer module:
  - advertiser lookup
  - creative listings + seen ranges
  - geo/date filters
- Source: dataset-backed where available; otherwise pluggable provider

#### Traffic panel
- Owned:
  - GA4 connector OR CSV import
- Proxies:
  - GSC clicks
  - CrUX origin-level UX metrics
- Optional:
  - third-party “traffic estimate” provider adapter

**Acceptance**
- Owned PPC dashboards work with connected Ads account.
- Overlap report loads and is actionable.
- Traffic panel shows at least GSC + one owned/proxy source.

---

## 7) Non-Functional Requirements

### Performance
- Audit: handle 1,000 pages/run without crashing; supports paging and exports.
- SERP: fetch snapshots for 100 keywords/day per project (configurable limits).
- Links: support at least one Common Crawl subset ingest and interactive queries.

### Reliability
- Celery retries with exponential backoff
- Idempotent tasks
- Progress tracking for long jobs (crawl, ingest, backfill)
- Failure visibility (admin UI + optional Flower)

### Security
- JWT auth
- encrypted OAuth token storage
- least-privilege scopes for Google
- rate limiting on APIs
- audit logs for integration actions (optional MVP)

### Observability
- structured logs
- job metrics (queued/running/failed)
- system metrics (queue depth, DB size growth)

---

## 8) Technical Architecture

### Services (docker-compose)
- `backend` (FastAPI)
- `frontend` (React)
- `db` (Postgres)
- `redis` (broker)
- `worker` (Celery worker)
- `beat` (Celery Beat)
- optional `flower`
- optional `clickhouse` (if enabling link/time-series at scale)
- optional `traefik` (keep if deploying via template)

### Modules (backend)
- `projects/`
- `audit/`
- `gsc/`
- `serp/`
  - `providers/` (default + optional)
- `links/`
- `ads/`
  - `providers/` (ads/transparency)
- `traffic/`
  - `providers/` (ga4/csv/crux/third-party)
- `alerts/`
- `worker/` (celery app + task routing)

### Architectural principle
**Provider Interfaces + Feature Flags**
- Every “sensitive/variable data source” is a provider.
- Default providers are first-party or open datasets.
- Optional providers are pluggable and disabled by default.

---

## 9) Data Model (SQLModel / Postgres)

### Core
- `User` (template)
- `Project`
  - id, name, seed_url, settings_json, created_by, created_at
- `ProjectMember` (optional MVP)
- `IntegrationAccount`
  - provider, user_id, tokens_encrypted, metadata_json

### Audits
- `AuditRun`
  - id, project_id, status, started_at, finished_at, crawl_config_json, stats_json
- `CrawledPage`
  - id, audit_run_id, url, final_url, status_code, content_type, title, meta_description, canonical, h1, word_count, response_time_ms, depth, is_rendered, rendered_hash
- `LinkEdge`
  - id, audit_run_id, source_url, target_url, is_internal, target_status_code
- `Issue`
  - id, audit_run_id, page_url, issue_type, severity, details_json, first_seen_at, last_seen_at
- `IssueAggregate`
  - audit_run_id, issue_type, severity, count

### GSC
- `GSCProperty`
  - id, project_id, site_url, verified, linked_at, search_type_default
- `GSCQueryDaily`
  - project_id, date, query, page, clicks, impressions, ctr, position, country?, device?, search_type
- `KeywordCluster`
  - id, project_id, label, created_at
- `KeywordClusterMember`
  - cluster_id, query, weight

### SERP / Rank Tracking
- `KeywordTarget`
  - id, project_id, keyword, locale, device, provider_key, schedule, created_at
- `SerpSnapshot`
  - id, keyword_target_id, fetched_at, provider_key, raw_json, status
- `RankObservation`
  - id, keyword_target_id, fetched_at, rank, url, domain, title, snippet

### Links (Common Crawl-based)
- `LinkSnapshot`
  - id, source="commoncrawl", crawl_id, ingested_at, subset_spec_json
- `BacklinkEdge`
  - snapshot_id, source_url, target_url, source_domain, target_domain, anchor, flags_json, first_seen, last_seen
- `RefDomainAgg`
  - snapshot_id, target_domain, ref_domain, backlinks_count, first_seen, last_seen
- Competitive views may be materialized tables or computed views.

### PPC + Traffic
- `AdsAccount`
  - id, project_id, customer_id, linked_at
- `AdsCampaignDaily`
  - project_id, date, campaign_id, clicks, impressions, cost, conversions, etc.
- `AdsKeywordDaily`
  - project_id, date, keyword_text, match_type, clicks, cost, conversions, final_url
- `TransparencyCreative`
  - source_key, advertiser_id, creative_id, seen_from, seen_to, geo_json, metadata_json
- `TrafficDaily`
  - project_id, date, source_key ("ga4"|"csv"|"gsc"|"crux"|"thirdparty"), metric_key, value, dimensions_json

---

## 10) API Requirements (FastAPI)

### Projects
- `POST /projects`
- `GET /projects`
- `GET /projects/{id}`
- `PATCH /projects/{id}`
- `DELETE /projects/{id}`

### Audits
- `POST /projects/{id}/audits`
- `GET /projects/{id}/audits`
- `GET /audits/{audit_run_id}`
- `GET /audits/{audit_run_id}/issues`
- `GET /audits/{audit_run_id}/pages`
- `GET /audits/{audit_run_id}/pages/{page_id}`
- `GET /audits/{audit_run_id}/export/issues.csv`

### GSC
- `POST /integrations/google/connect`
- `GET /projects/{id}/gsc/properties`
- `POST /projects/{id}/gsc/link`
- `POST /projects/{id}/gsc/sync`
- `POST /projects/{id}/gsc/backfill`
- `GET /projects/{id}/gsc/queries`
- `GET /projects/{id}/gsc/pages`
- `GET /projects/{id}/gsc/opportunities`
- `GET /projects/{id}/gsc/clusters`

### SERP / Rank Tracking
- `POST /projects/{id}/serp/keywords`
- `GET /projects/{id}/serp/keywords`
- `DELETE /serp/keywords/{keyword_target_id}`
- `POST /serp/keywords/{keyword_target_id}/refresh`
- `GET /serp/keywords/{keyword_target_id}/observations`
- `GET /serp/keywords/{keyword_target_id}/snapshots/{snapshot_id}`

### Links
- `GET /links/domain/{domain}/refdomains`
- `GET /links/domain/{domain}/backlinks`
- `GET /links/domain/{domain}/anchors`
- `GET /links/domain/{domain}/new-lost` (requires two snapshots)
- `GET /links/domain/{domain}/overlap?competitors=...`
- `GET /links/domain/{domain}/intersect?competitors=...`

### PPC + Traffic
- `POST /integrations/google-ads/connect`
- `POST /projects/{id}/ads/link`
- `POST /projects/{id}/ads/sync`
- `GET /projects/{id}/ads/campaigns`
- `GET /projects/{id}/ads/keywords`
- `GET /projects/{id}/ads/landing-pages`
- `GET /projects/{id}/ads/seo-overlap`
- `GET /transparency/search` (advertiser/geo/date filters)

- `POST /projects/{id}/traffic/connect/ga4`
- `POST /projects/{id}/traffic/import/csv`
- `GET /projects/{id}/traffic/panel`

### Alerts
- `GET /projects/{id}/alerts/settings`
- `PUT /projects/{id}/alerts/settings`
- `GET /projects/{id}/alerts/history`

---

## 11) Background Jobs (Celery Task Graph)

### Audit pipeline
1) `audit.run_crawl(project_id, audit_run_id)`
2) `audit.render_pages(audit_run_id)` (if Hybrid/JS)
3) `audit.analyze_pages(audit_run_id)`
4) `audit.compute_diff(project_id, audit_run_id)`

### GSC pipeline
1) `gsc.sync_property(project_id)`
2) `gsc.backfill(project_id, date_range_chunks)`
3) `gsc.compute_opportunities(project_id, window)`
4) `gsc.cluster_queries(project_id, window)`

### SERP pipeline
1) `serp.refresh_keyword(keyword_target_id)` (provider throttling)
2) `serp.refresh_project(project_id)` (batch refresh respecting daily caps)

### Links pipeline
1) `links.ingest_commoncrawl(crawl_id, subset_spec)`
2) `links.build_aggregates(snapshot_id)`
3) `links.compute_competitive_views(snapshot_id)`

### PPC + Traffic pipeline
1) `ads.sync_account(project_id)`
2) `ads.compute_overlap(project_id, window)`
3) `traffic.sync_ga4(project_id)` (if connected)
4) `traffic.sync_crux(project_id)` (if enabled)
5) `traffic.build_panel(project_id, window)`

### Schedules (Celery Beat)
- Daily:
  - `gsc.sync_property` (all linked projects)
  - `serp.refresh_project` (projects with rank tracking enabled)
  - `ads.sync_account` (linked projects)
- Weekly (default):
  - `audit.run_crawl` (scheduled projects)
- Ingestion cadence:
  - `links.ingest_commoncrawl` (monthly or as configured)
- Daily digest:
  - `alerts.send_digest`

---

## 12) Frontend Requirements (React/Vite + shadcn)

### Pages
- Auth screens (template)
- Projects:
  - list + create + overview dashboard
- Audits:
  - run status/progress
  - issues table
  - pages table
  - page details (rendered vs non-rendered evidence)
- Keywords (GSC):
  - query explorer
  - page explorer
  - opportunities
  - clusters
- Rank Tracking:
  - keyword targets list
  - trend charts
  - SERP snapshot viewer
- Links:
  - backlink explorer
  - referring domains
  - anchors
  - overlap/intersect
- PPC:
  - campaigns/keywords/landing pages
  - SEO + PPC overlap
  - transparency explorer
- Traffic panel:
  - GA4/csv connect/import
  - combined dashboard
- Settings:
  - schedules
  - providers (enabled/disabled)
  - retention & limits
  - alerts config

### UI requirements (MVP)
- Tables: filter/sort/paginate/export
- Clear job status indicators (queued/running/failed)
- “Run now” actions for each module
- Provider status badges (enabled/disabled)

---

## 13) Testing & QA

### Backend
- Unit tests:
  - audit rules
  - URL normalization/canonicalization
  - provider interfaces contract tests (mock providers)
  - GSC transforms and opportunity scoring
- Integration tests:
  - enqueue job → worker completion
  - DB writes and reads for audits/GSC/SERP
- Security tests:
  - auth-required endpoints
  - token encryption round-trip

### Frontend
- Playwright E2E:
  - create project
  - run audit (test worker mode)
  - connect GSC (mock)
  - add keyword targets + see observations (mock provider)
  - view link explorer (seeded fixtures)
  - view PPC dashboard (seeded fixtures)

---

## 14) Metrics & Observability

### Product metrics
- audit runs/week; pages/run
- issues by severity/type; new vs resolved
- GSC rows/day; opportunity count; cluster count
- SERP refresh success rate; keywords tracked
- link snapshots ingested; referring domains count
- PPC sync count; overlap insights generated
- traffic panel coverage by source

### System metrics
- queue depth; task runtime; failure rate
- crawl fetch errors/timeouts
- provider errors (rate limited, outage)
- DB growth and query latency

---

## 15) Milestones & Sprints (Suggested)

### Sprint 0: Template adoption + worker lane (1–3 days)
- Fork template; rename example domain to Projects
- Add Redis + worker + beat + basic task routing
- Implement progress/status model for jobs

### Sprint 1: Audit MVP + Hybrid rendering (1–2 weeks)
- HTML crawl + storage
- Playwright render worker queue + hybrid mode
- Issue rules (critical/high first), exports, UI

### Sprint 2: GSC MVP (1–2 weeks)
- OAuth connect + link property
- daily ingestion + 28/90 windows
- explorers + opportunities + basic clustering

### Sprint 3: SERP Rank Tracking MVP (1 week)
- Provider interface + default provider(s)
- keyword targets + scheduled refresh
- snapshots + trend UI

### Sprint 4: Links MVP (1–2 weeks)
- Common Crawl ingest (subset)
- backlink explorer + referring domains + anchors
- overlap/intersect views

### Sprint 5: PPC + Transparency + Traffic MVP (1–2 weeks)
- Google Ads connector + PPC dashboards
- SEO+PPC overlap
- transparency explorer (dataset/provider)
- traffic panel (GA4/csv + GSC + CrUX)

### Sprint 6: Hardening + polish (1 week)
- performance tuning
- quotas/limits config
- improved onboarding + docs
- CI hardening + deployment docs

---

## 16) Risks & Mitigations

- Crawl explosion / infinite URL spaces  
  - max pages/depth, URL normalization, query param rules, include/exclude patterns

- SERP provider reliability / legal concerns  
  - compliant-by-default providers; adapter interface; strict feature flags; no circumvention guidance

- GSC quotas & sampling  
  - throttling, chunked backfills, incremental sync, transparent “data freshness” UX

- Link data volume  
  - start with subsets; optional ClickHouse; materialized aggregates

- OAuth token security  
  - encryption at rest; rotation; least scopes; audit logs of integration actions

- Cost management  
  - enforce budgets: render limits, SERP caps, ingest subsets

---

## 17) MVP Definition of Done (Updated)

A new user can:
1) sign up, create a project
2) run an audit (HTML + optional hybrid rendering) and see issues + exports + diffs
3) connect GSC, ingest data, view explorers/opportunities/clusters, and receive an alert digest
4) add keywords and see rank observations + SERP snapshots via enabled providers
5) enter a domain and see backlink/referring-domain data (from ingested Common Crawl snapshot) plus overlap/intersect
6) connect Google Ads and view PPC dashboards + SEO overlap; view transparency explorer (where available); see traffic panel (GA4/csv + proxies)

System requirements:
- docker-compose runs the full stack locally
- background jobs are reliable (retries/backoff) with visible failures
- CI passes (unit + integration + basic E2E)
- provider-based modules are swappable and feature-flagged

---
