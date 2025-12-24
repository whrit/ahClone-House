# SEO Platform User Guide

Complete guide to using all features of the SEO Platform.

---

## Table of Contents

1. [Dashboard Overview](#dashboard-overview)
2. [Projects Management](#projects-management)
3. [Site Audits](#site-audits)
4. [Keywords (Google Search Console)](#keywords-google-search-console)
5. [Rank Tracking](#rank-tracking)
6. [Backlinks](#backlinks)
7. [PPC (Google Ads)](#ppc-google-ads)
8. [Traffic Panel](#traffic-panel)
9. [User Settings](#user-settings)
10. [Tips and Best Practices](#tips-and-best-practices)

---

## Dashboard Overview

The dashboard is your central hub for accessing all SEO tools and data.

### Main Navigation

Located in the left sidebar:

- **Dashboard**: Overview and quick stats
- **Projects**: List and manage all your projects
- **Settings**: User account settings
- **Admin** (superusers only): User management

### Project Navigation

Once you select a project, additional navigation appears:

- **Overview**: Project summary and recent activity
- **Audits**: Technical SEO audits
- **Keywords**: GSC data and query analysis
- **Rank Tracker**: SERP position monitoring
- **Links**: Backlink analysis
- **PPC**: Google Ads integration
- **Traffic**: Multi-source traffic aggregation
- **Settings**: Project-specific configuration

---

## Projects Management

### Creating a Project

1. Click **"Projects"** in the main navigation
2. Click **"New Project"** button
3. Fill in the form:

   **Required Fields**:
   - **Name**: Project identifier (e.g., "Main Website")
   - **Domain**: Primary domain without protocol (e.g., "example.com")

   **Optional Fields**:
   - **Description**: Internal notes about the project
   - **Settings**: Advanced configuration (can be changed later)

4. Click **"Create"**

### Project Settings

Access via **Project → Settings**:

#### General Settings

- **Name**: Update project name
- **Domain**: Change primary domain (affects all modules)
- **Description**: Update internal notes

#### Audit Settings

- **Max Pages to Crawl**: Limit crawl depth (default: 1000)
  - Higher values = longer audit times
  - Recommended: 500-1000 for most sites
- **Max Pages to Render**: JS rendering limit (default: 100)
  - Rendering is resource-intensive
  - Only critical pages need JS rendering
- **Enable JS Rendering**: Toggle hybrid rendering
  - Automatically detects JavaScript-dependent pages
  - Renders them with Playwright headless browser
- **Follow External Links**: Crawl beyond your domain
  - Usually disabled to focus on your site
  - Enable for competitive analysis
- **User Agent**: Custom crawler identification
- **Respect robots.txt**: Honor crawl directives (recommended: enabled)

#### SERP Settings

- **Refresh Frequency**: How often to update keyword positions
  - Options: Daily, Weekly, Manual
  - Daily recommended for active campaigns
- **Provider**: Position data source
  - **GSC-based** (default): Uses Google Search Console data (free, accurate)
  - **API Provider**: Third-party SERP APIs (requires API key, real-time)

#### Retention Policies

- **Keep Audit Runs**: Days to retain audit history (default: 30)
- **Keep SERP Snapshots**: Days to retain position data (default: 90)
- **Keep Link Snapshots**: Days to retain backlink data (default: 60)

### Deleting a Project

1. Navigate to **Project → Settings**
2. Scroll to **Danger Zone**
3. Click **"Delete Project"**
4. Confirm deletion

**Warning**: This action is permanent and deletes all associated data:
- Audit runs and issues
- Keywords and SERP snapshots
- Backlinks and anchors
- PPC data and traffic records

---

## Site Audits

Site Audits crawl your website to identify technical SEO issues.

### Running an Audit

#### Quick Start

1. Navigate to **Project → Audits**
2. Click **"Run New Audit"**
3. The audit is queued automatically

#### Advanced Options

Click **"Advanced Options"** before running:

- **Starting URL**: Entry point for crawl (default: homepage)
- **Max Pages**: Override project default
- **Render Pages**: Force rendering count
- **Ignore robots.txt**: Override for this audit only

### Audit Status

Audits progress through these states:

1. **Queued**: Waiting for worker availability
2. **Running**: Currently crawling
   - Progress percentage shown
   - Estimated completion time
3. **Completed**: Finished successfully
4. **Failed**: Encountered an error
   - Error message displayed
   - Partial results may be available

### Audit Results

#### Summary Tab

High-level overview:

- **Pages Crawled**: Total pages discovered and analyzed
- **Issues Found**: Total count across all severities
- **Severity Breakdown**:
  - Critical: 🔴 (red badge)
  - High: 🟠 (orange badge)
  - Medium: 🟡 (yellow badge)
  - Low: 🔵 (blue badge)
- **Top Issue Types**: Most common problems
- **Crawl Statistics**:
  - Average response time
  - Error rate
  - Links discovered

#### Issues Tab

Detailed issue explorer with powerful filtering:

**Filters**:
- **Severity**: Filter by Critical, High, Medium, Low
- **Issue Type**: Filter by category (see below)
- **Status**: New issues vs. previously seen
- **Search**: Keyword search in titles and descriptions

**Issue Table Columns**:
- **Severity**: Visual indicator (Critical/High/Medium/Low)
- **Type**: Issue category
- **Title**: Brief description
- **Affected URL**: Page with the issue
- **Details**: Expandable additional context
- **First Seen**: When issue was first detected
- **Status**: New, Existing, Resolved

**Actions**:
- **Export CSV**: Download all issues for external analysis
- **View Details**: Click any issue for full context
- **Compare Runs**: See what changed between audits

#### Issue Types

The platform detects 17 types of SEO issues:

##### Critical Issues

| Type | Description | Impact |
|------|-------------|--------|
| **MISSING_TITLE** | Page has no `<title>` tag | Severe: Prevents indexing |
| **TITLE_TOO_SHORT** | Title < 30 characters | High: Reduces click-through |
| **TITLE_TOO_LONG** | Title > 60 characters | High: Gets truncated in SERPs |
| **STATUS_5XX** | Server error (500, 502, 503) | Critical: Page inaccessible |
| **STATUS_4XX** | Client error (404, 403) | High: Broken links |

##### High Priority Issues

| Type | Description | Impact |
|------|-------------|--------|
| **DUPLICATE_TITLE** | Multiple pages share same title | SEO: Confuses search engines |
| **MISSING_H1** | No H1 heading on page | Moderate: Reduces topical clarity |
| **MULTIPLE_H1** | More than one H1 | Low-Moderate: Dilutes focus |
| **BROKEN_INTERNAL_LINK** | Link to 404 page on your site | User experience degradation |

##### Medium Priority Issues

| Type | Description | Impact |
|------|-------------|--------|
| **MISSING_META_DESC** | No meta description | CTR: Generic snippet shown |
| **META_DESC_TOO_SHORT** | Description < 120 characters | CTR: Wasted SERP real estate |
| **META_DESC_TOO_LONG** | Description > 160 characters | CTR: Gets truncated |
| **SLOW_PAGE** | Load time > 3 seconds | UX: Increased bounce rate |

##### Low Priority Issues

| Type | Description | Impact |
|------|-------------|--------|
| **MISSING_IMAGE_ALT** | Image without alt attribute | Accessibility + image SEO |
| **HEADING_SKIP** | Heading order skip (H1 → H3) | Minor: Semantic structure |
| **TOO_MANY_LINKS** | > 100 links on page | Dilutes link equity |
| **THIN_CONTENT** | < 300 words on page | Quality signal concern |

#### Pages Tab

Explore all crawled pages:

**Table Columns**:
- **URL**: Full page URL
- **Status**: HTTP status code
- **Title**: Page title tag
- **Word Count**: Text content length
- **Links**: Outbound links count
- **Response Time**: Load time in ms
- **Crawl Depth**: Distance from start URL
- **Issues**: Count of issues on this page

**Filters**:
- Status code (200, 301, 404, etc.)
- URL pattern (contains, starts with)
- Response time range
- Issue count

**Bulk Actions**:
- Export selected pages to CSV
- Mark pages for re-crawl

#### Comparing Audits

View changes between audit runs:

1. Navigate to **Audits**
2. Select two audit runs
3. Click **"Compare"**

**Comparison View**:
- **New Issues**: Problems detected in latest run
- **Resolved Issues**: Problems fixed since previous run
- **Persistent Issues**: Still present in both runs
- **Page Changes**: URLs added or removed

### Audit History

The platform retains audit runs based on your retention policy (default: 30 days).

**Benefits of History**:
- Track SEO health trends over time
- Verify that fixes were effective
- Identify recurring issues
- Report on progress to stakeholders

---

## Keywords (Google Search Console)

The Keywords module integrates with Google Search Console to analyze search performance.

### Connecting Google Search Console

#### Prerequisites

- Google account with GSC access
- GSC property with 28+ days of data
- OAuth credentials configured (see [Getting Started](./getting-started.md#connecting-google-search-console))

#### Connection Steps

1. Navigate to **Project → Keywords**
2. Click **"Connect Google Search Console"**
3. Sign in with your Google account
4. Grant permissions to the SEO Platform
5. Select your GSC property from the dropdown
6. Click **"Connect"**

**Note**: One GSC property per project. To switch properties, disconnect and reconnect.

### Syncing Data

#### Initial Sync

After connecting:

1. Click **"Sync Now"**
2. The platform fetches the last 28 days of data
3. Monitor progress in **Jobs** tab
4. Sync typically takes 2-5 minutes

#### Automatic Daily Sync

Once connected, the platform automatically syncs new data daily at 3 AM.

**What Gets Synced**:
- Yesterday's query performance (clicks, impressions, CTR, position)
- Yesterday's page performance
- Updated totals for the trailing 28-day window

### Query Explorer

Analyze search queries that triggered your site in Google results.

**Table Columns**:
- **Query**: Search term
- **Clicks**: Users who clicked your result
- **Impressions**: Times your result was shown
- **CTR**: Click-through rate (clicks ÷ impressions)
- **Position**: Average ranking position
- **Change**: Position change vs. previous period (7 days)

**Filters**:
- Date range (last 7, 14, 28 days)
- Query contains (keyword search)
- Min clicks, impressions, CTR, position
- Device type (desktop, mobile, tablet)
- Country

**Sorting**:
- Click any column header to sort
- Useful sorts:
  - High impressions + low CTR = optimization opportunities
  - Position 8-20 = quick-win keywords
  - Largest position drops = investigate issues

**Export**:
- Click **"Export CSV"** to download all filtered queries
- Includes all columns plus metadata

### Page Explorer

View landing pages that received organic traffic.

**Table Columns**:
- **Page URL**: Landing page
- **Clicks**: Users who clicked through
- **Impressions**: Times this page appeared in results
- **CTR**: Click-through rate
- **Position**: Average position across all queries
- **Queries**: Number of unique queries triggering this page

**Use Cases**:
- Identify top traffic drivers
- Find pages with declining performance
- Spot pages with low CTR (title/meta optimization needed)
- Discover unoptimized high-impression pages

### Opportunities

Auto-detected keyword optimization opportunities.

#### Opportunity Types

##### 1. Low CTR Opportunities

**Criteria**: Queries with:
- Position ≤ 10 (first page)
- CTR < 5%
- Impressions > 100

**Why It Matters**: You're ranking well but not getting clicks.

**Actions**:
- Improve title tag (add power words, numbers, questions)
- Enhance meta description (compelling CTA, unique value prop)
- Add schema markup (ratings, pricing, FAQ)

##### 2. Position 8-20 (Quick Wins)

**Criteria**: Queries ranking on positions 8-20

**Why It Matters**: Small improvements can push you to page 1.

**Actions**:
- Add more comprehensive content
- Improve internal linking to target page
- Update content freshness
- Optimize on-page SEO elements

##### 3. Rising Queries

**Criteria**: Queries with:
- Position improvement ≥ 3 positions in last 7 days
- Impressions > 50

**Why It Matters**: Momentum indicates Google is rewarding your content.

**Actions**:
- Double down: expand content on this topic
- Build more internal links
- Create supporting content (cluster model)

##### 4. Falling Queries

**Criteria**: Queries with:
- Position decline ≥ 3 positions in last 7 days
- Clicks > 10 (previously performing)

**Why It Matters**: You're losing traffic. Investigate quickly.

**Actions**:
- Check if competitor published better content
- Review for technical issues (crawl errors, slow load)
- Update content with fresh information
- Check for keyword cannibalization

**Opportunity Scoring**:

Each opportunity receives a score (0-100) based on:
- Position (higher = more valuable)
- Impressions (higher = more potential)
- Current CTR (lower = more upside)
- Click value (higher = more revenue impact)

Sort by score to prioritize your efforts.

### Query Clusters

Automatically group related queries for topical SEO.

#### How Clustering Works

The platform uses **N-gram overlap** with **Jaccard similarity**:

1. Tokenize queries into words (n-grams)
2. Calculate similarity between query pairs
3. Group queries with similarity > 0.3
4. Name cluster based on most common terms

**Example Cluster**:
```
Cluster: "seo audit tools"
Queries:
- seo audit tool
- best seo audit software
- free seo audit tools
- seo site audit
- technical seo audit tools
```

#### Using Clusters

**Table Columns**:
- **Cluster Name**: Auto-generated from common terms
- **Query Count**: Queries in this cluster
- **Total Clicks**: Sum of all query clicks
- **Total Impressions**: Sum of all query impressions
- **Avg Position**: Average position across cluster
- **Avg CTR**: Average click-through rate

**Expand Cluster**:
- Click any cluster to see member queries
- View individual query performance
- Export cluster queries to CSV

**Use Cases**:
- **Content Planning**: Each cluster = one comprehensive page/post
- **Keyword Research**: Discover related terms you rank for
- **Cannibalization Detection**: Multiple pages targeting same cluster = bad
- **Topic Authority**: Large clusters = strong topical coverage

**Actions**:
- Create pillar content for high-volume clusters
- Internal link cluster pages together
- Add missing queries to existing content
- Consolidate cannibalized pages

---

## Rank Tracking

Monitor keyword positions in search results over time.

### Adding Keywords

#### Quick Add

1. Navigate to **Project → Rank Tracker**
2. Click **"Add Keyword"**
3. Fill in:
   - **Keyword**: Target search term (e.g., "seo audit tool")
   - **Target URL**: (Optional) Specific page you want to rank
   - **Refresh Frequency**: Daily, Weekly, or Manual
4. Click **"Save"**

#### Bulk Import

1. Click **"Import Keywords"**
2. Upload CSV with columns:
   ```
   keyword,target_url,frequency
   seo audit tool,/features/audits,daily
   backlink checker,/features/backlinks,daily
   ```
3. Map columns
4. Click **"Import"**

### Position Data Sources

#### GSC-based Provider (Default)

**How It Works**:
- Fetches average position from Google Search Console
- Uses actual Google data (100% accurate)
- No API costs
- Updates once per day with GSC sync

**Limitations**:
- Only shows average position (not exact SERP rank)
- Requires GSC connection
- Limited to queries with impressions in GSC

**Best For**: Most users, especially those focused on organic traffic.

#### API Provider (Optional)

**How It Works**:
- Queries third-party SERP API (SerpApi, DataForSEO, etc.)
- Real-time position checks
- Exact SERP ranking (position 1-100)
- Tracks all 10+ results on first page

**Requirements**:
- SERP API key
- API costs per query
- Higher refresh quota consumption

**Best For**: Competitive analysis, real-time monitoring, non-GSC properties.

**Configuration**:
```bash
# In .env
SERP_API_KEY=your-api-key
SERP_API_URL=https://serpapi.com/search
```

Enable in **Project Settings → SERP Settings → Provider → API Provider**.

### Viewing Rank History

Click on any keyword to see details:

#### Position Chart

- **X-axis**: Date (last 30 days by default)
- **Y-axis**: Position (lower = better)
- **Data Points**: Daily position observations
- **Trend Line**: 7-day moving average

**Chart Interactions**:
- Hover over points for exact position and date
- Zoom in on specific date range
- Toggle trend line visibility

#### Position Changes

**Position Delta**:
- **+5** (green): Improved 5 positions
- **-3** (red): Declined 3 positions
- **0** (gray): No change

**Change Alerts**:
- Automatic notifications for significant changes (±5 positions)
- Configured in **User Settings → Notifications**

#### SERP Snapshot

View the top 10 search results for this keyword:

**Table Columns**:
- **Rank**: Position (1-10)
- **URL**: Result URL
- **Title**: Page title
- **Description**: Meta description
- **Highlighted**: Your site (if ranking)

**Competitive Analysis**:
- See who you're competing against
- Identify common themes in top results
- Spot opportunities (weak competitors, missing content)

**Snapshot History**:
- Click **"View Previous"** to see historical SERPs
- Compare how SERP features evolved
- Identify new competitors

### Refresh Settings

#### Automatic Refresh

**Daily Refresh** (default):
- Runs at 4 AM server time
- Respects daily cap (default: 100 keywords)
- Prioritizes keywords by:
  1. Manual refresh requests
  2. Longest time since last refresh
  3. Keyword priority (if set)

**Weekly Refresh**:
- Runs every Monday at 4 AM
- Suitable for low-priority keywords
- Reduces quota consumption

#### Manual Refresh

Force immediate refresh:

1. Click keyword
2. Click **"Refresh Now"**
3. Position updates within 1-2 minutes

**Limitations**:
- Consumes daily quota
- Max 10 manual refreshes per day
- Only available for API-based provider (GSC is once-daily)

### Keyword Organization

#### Tags

Organize keywords with tags:

1. Select keywords (checkboxes)
2. Click **"Add Tag"**
3. Enter tag name (e.g., "brand", "commercial", "informational")
4. Click **"Save"**

**Filter by Tag**:
- Click tag badge to filter
- Useful for grouping by:
  - Intent (navigational, informational, transactional)
  - Priority (high, medium, low)
  - Campaign (summer-2024, relaunch, etc.)

#### Priority

Set keyword importance:

- **High**: Business-critical (homepage keywords, main services)
- **Medium**: Supporting content
- **Low**: Long-tail, low-volume

**Benefits**:
- Influences automatic refresh prioritization
- Visual indicators in keyword list
- Filter by priority

### Export and Reporting

#### Export Position Data

1. Select keywords (or click **"Select All"**)
2. Click **"Export"**
3. Choose format:
   - **CSV**: All position data with timestamps
   - **Excel**: Multi-sheet with charts
   - **PDF**: Shareable report with graphs

**Export Contents**:
- Keyword list with current positions
- Position history (30/60/90 days)
- Change summary (improvements, declines)
- SERP snapshots (optional)

#### Scheduled Reports

Configure automatic email reports:

1. Navigate to **Project → Settings → Reports**
2. Click **"Add Report"**
3. Configure:
   - **Name**: "Weekly Rank Report"
   - **Frequency**: Weekly, Monday 9 AM
   - **Recipients**: team@example.com
   - **Include**: Position changes, top movers, SERP snapshots
4. Click **"Save"**

---

## Backlinks

Analyze backlinks using Common Crawl data and competitive intelligence.

### Understanding Common Crawl

**What is Common Crawl?**

Common Crawl is a nonprofit that crawls billions of web pages and makes the data publicly available. The SEO Platform uses this data to discover backlinks.

**How It Works**:

1. Common Crawl archives web pages monthly
2. The platform downloads WARC files (web archive format)
3. Parses HTML to extract links
4. Stores backlinks pointing to your domain

**Limitations**:
- Data is 1-2 months old (not real-time)
- Not as comprehensive as commercial tools (Ahrefs, Majestic)
- Does not include link metrics (DA, PA)

**Benefits**:
- Completely free
- Covers billions of pages
- No API quotas or costs

### Ingesting Backlinks

#### Start Ingestion

1. Navigate to **Project → Links**
2. Click **"Ingest Common Crawl"**
3. Select index:
   - Most recent (e.g., CC-MAIN-2024-10)
   - Specific date range
4. Configure:
   - **Domain**: Your domain (pre-filled)
   - **Max Links**: Limit (default: 100,000)
5. Click **"Start Ingestion"**

**Progress Monitoring**:
- Job queued in background
- View progress in **Jobs** tab
- Typical ingestion: 30-60 minutes

**What Gets Stored**:
- Source URL (page with the link)
- Target URL (page on your site)
- Anchor text
- Link attributes (rel="nofollow", rel="sponsored", rel="ugc")
- Discovery date

#### Re-ingestion

Run ingestion monthly to:
- Discover new backlinks
- Identify lost links
- Track link growth

**Comparison Mode**:
- Click **"Compare Snapshots"**
- Select two ingestion dates
- View:
  - New links gained
  - Links lost
  - Net link growth

### Referring Domains

View domains linking to your site:

**Table Columns**:
- **Domain**: Referring domain (e.g., example.com)
- **Links**: Count of links from this domain
- **Unique Pages**: Unique pages on your site they link to
- **Follow Links**: Count of dofollow links
- **Nofollow Links**: Count of nofollow links
- **First Seen**: When first discovered
- **Last Seen**: Most recent observation

**Sorting**:
- Most links: Largest linkers
- Most unique pages: Broad coverage
- Highest follow %: Quality signals

**Domain Metrics** (if available):
- Domain Authority estimate
- Page count
- Top referring pages

**Actions**:
- Click domain to see all backlinks
- Export to CSV
- Add to monitoring list (alerts for changes)

### Backlinks List

Detailed view of individual links:

**Table Columns**:
- **Source URL**: Page containing the link
- **Target URL**: Page on your site
- **Anchor Text**: Link text
- **Rel Attributes**: nofollow, sponsored, ugc
- **Context**: Surrounding text (snippet)
- **First Seen**: Discovery date
- **Status**: Active, Lost, Redirect

**Filters**:
- Anchor text contains
- Target URL pattern
- Link type (follow, nofollow, sponsored)
- Domain
- Date range

**Bulk Actions**:
- Export selected links
- Mark as disavow candidates
- Add to monitoring

### Anchor Text Analysis

Understand how sites link to you:

**Table Columns**:
- **Anchor Text**: Link text
- **Occurrences**: Times this anchor is used
- **Domains**: Unique domains using this anchor
- **% of Total**: Percentage of all backlinks
- **Type**: Branded, Exact Match, Partial Match, Generic, URL

**Anchor Distribution Health**:

The platform analyzes your anchor text profile:

**Healthy Profile**:
- 40-50% branded anchors ("YourBrand", "YourBrand.com")
- 20-30% exact/partial match ("seo tool", "best seo audit tool")
- 20-30% generic ("click here", "learn more", "website")
- 5-10% naked URLs ("https://example.com")

**Warning Signs**:
- Over-optimization: >50% exact match = potential penalty risk
- Too generic: >60% "click here" = weak relevance signals
- No branded: Lack of brand recognition

**Actions**:
- Identify over-optimized anchors
- Diversify anchor text in outreach
- Disavow spammy exact match links

### Competitive Analysis

Compare your backlink profile with competitors:

#### Overlap Analysis

Find domains linking to both you and a competitor:

1. Click **"Competitive"**
2. Click **"Link Overlap"**
3. Enter competitor domain
4. Click **"Analyze"**

**Results**:
- **Shared Domains**: Link to both
- **Your Unique**: Only link to you
- **Their Unique**: Only link to competitor
- **Opportunity Score**: Likelihood of getting their unique links

**Actions**:
- Reach out to "their unique" domains
- Analyze why shared domains link to both
- Identify link building strategies

#### Link Intersect

Find domains linking to multiple competitors but not you:

1. Click **"Link Intersect"**
2. Enter 2-3 competitor domains
3. Click **"Find Intersect"**

**Results**:
- Domains linking to all competitors
- Not linking to you
- Sorted by opportunity score

**Why It's Valuable**:
- These domains are clearly interested in your niche
- High conversion rate for outreach
- Already know the topic/industry

**Actions**:
- Create outreach list
- Analyze what content competitors got links for
- Replicate and improve content

### New and Lost Links

Track link velocity:

#### New Links Report

**Last 7 Days**:
- Count of new backlinks discovered
- Top new referring domains
- Anchor text of new links
- Target pages gaining links

**Velocity Chart**:
- Daily new link count (30-day history)
- Trend line (growing, stable, declining)

**Alerts**:
- Spike in new links (potential negative SEO)
- Sudden drop (lost link campaign)

#### Lost Links Report

**Last 7 Days**:
- Count of lost backlinks
- Domains that removed links
- Target pages losing links

**Reasons for Link Loss**:
- Page deleted (404)
- Link removed
- Domain expired
- Changed to nofollow

**Actions**:
- Investigate high-value lost links
- Reach out to reclaim (if appropriate)
- Fix broken pages (404s)
- Redirect deleted pages

---

## PPC (Google Ads)

Integrate Google Ads data to analyze paid search campaigns and SEO+PPC overlap.

### Connecting Google Ads

#### Prerequisites

- Active Google Ads account
- Google Ads Developer Token (apply at ads.google.com/intl/en_us/home/tools/api/)
- OAuth credentials configured

#### Connection Steps

1. Navigate to **Project → PPC**
2. Click **"Connect Google Ads"**
3. Sign in with your Google account (must have Ads access)
4. Grant permissions
5. Select Ads account from list
6. Click **"Connect"**

### Syncing Campaign Data

#### Initial Sync

1. Click **"Sync Campaigns"**
2. Select date range (default: last 30 days)
3. Click **"Start Sync"**

**What Gets Synced**:
- Campaign performance (clicks, impressions, cost, conversions)
- Ad group performance
- Keyword performance (search terms)
- Ad copy and extensions
- Quality scores

**Sync Frequency**:
- Manual: On-demand
- Automatic: Daily at 5 AM (optional, configure in settings)

### PPC Dashboard

Campaign overview and trends:

**Summary Cards**:
- **Total Spend**: Last 30 days
- **Total Clicks**: Click volume
- **Avg CPC**: Cost per click
- **Conversions**: Goal completions
- **ROAS**: Return on ad spend

**Charts**:
- **Spend Over Time**: Daily spend trend
- **Click Trend**: Daily clicks
- **Conversion Trend**: Daily conversions
- **CPC Trend**: Average CPC by day

**Campaign Performance Table**:
- **Campaign**: Campaign name
- **Status**: Active, Paused, Ended
- **Clicks**: Click count
- **Impressions**: Impression count
- **CTR**: Click-through rate
- **Cost**: Total spend
- **Conversions**: Conversion count
- **CPA**: Cost per acquisition
- **ROAS**: Return on ad spend

### Keyword Performance

Analyze search terms triggering your ads:

**Table Columns**:
- **Keyword**: Search term
- **Match Type**: Exact, Phrase, Broad
- **Clicks**: Click count
- **Impressions**: Impression count
- **CTR**: Click-through rate
- **Avg CPC**: Average cost per click
- **Cost**: Total spend on this keyword
- **Conversions**: Conversion count
- **Conv Rate**: Conversion rate
- **Quality Score**: Google's 1-10 score

**Filters**:
- Campaign
- Ad group
- Match type
- Quality score range
- Cost range
- Date range

**Insights**:
- Low QS keywords = landing page/relevance issues
- High spend, low conversions = pause or optimize
- High CTR, low conv rate = landing page problem

### SEO+PPC Overlap

Identify keywords you're paying for that you also rank organically:

#### Overlap Report

**Table Columns**:
- **Keyword**: Search term
- **Organic Position**: SEO rank
- **Paid Position**: Ad position
- **PPC Clicks**: Paid clicks
- **PPC Cost**: Spend on this keyword
- **Organic Clicks**: Free clicks (from GSC)
- **Opportunity Score**: Potential savings

**Opportunity Types**:

1. **High Organic, High Paid Spend**:
   - Ranking #1-3 organically
   - Also running paid ads
   - **Action**: Pause ads, rely on organic
   - **Savings**: 100% of ad spend

2. **Low Organic, High Intent**:
   - Ranking #10+
   - High conversion rate in PPC
   - **Action**: Improve organic content while running ads
   - **Keep**: Until organic improves

3. **Branded Terms**:
   - Your brand name
   - Often rank #1 organically
   - **Consideration**: Competitors may bid on your brand
   - **Action**: Run ads defensively (low bids)

**Total Overlap Savings**:
- Sum of potential savings
- Assumes 50% click shift from paid to organic
- Conservative estimate

#### Optimization Recommendations

The platform auto-generates recommendations:

**Recommendation Types**:
- **Pause Keyword**: High organic rank, low value
- **Increase Budget**: Low organic rank, high ROAS
- **Improve Landing Page**: High bounce rate
- **Lower Bids**: High CPC, low conversion
- **Expand Keywords**: High ROAS, low impression share

### Ad Transparency (Competitor Ads)

View competitor ads (using Google Ads Transparency data):

1. Navigate to **PPC → Transparency**
2. Enter competitor domain
3. View:
   - Active ad creatives
   - Ad copy
   - Landing pages
   - Ad duration
   - Estimated spend range

**Use Cases**:
- Competitive research
- Ad copy inspiration
- Identify competitor messaging
- Discover new keyword opportunities

---

## Traffic Panel

Aggregate traffic data from multiple sources for unified reporting.

### Supported Sources

| Source | Data Type | Requirements |
|--------|-----------|--------------|
| **Google Search Console** | Organic clicks | GSC connection |
| **Google Ads** | Paid clicks | Ads connection |
| **Google Analytics 4** | All traffic | GA4 API credentials |
| **Chrome UX Report** | Performance metrics | None (public data) |
| **CSV Import** | Custom | CSV file |

### Connecting Sources

#### Google Analytics 4

1. Navigate to **Traffic → Sources**
2. Click **"Connect GA4"**
3. Sign in with Google account
4. Grant permissions
5. Select GA4 property
6. Click **"Connect"**

**What Gets Synced**:
- Sessions by source/medium
- Pageviews
- Users
- Bounce rate
- Engagement rate
- Conversions

#### Chrome UX Report (CrUX)

Automatic for domains in CrUX dataset (requires 1,000+ users):

**Metrics**:
- First Contentful Paint (FCP)
- Largest Contentful Paint (LCP)
- First Input Delay (FID)
- Cumulative Layout Shift (CLS)
- Time to First Byte (TTFB)

**Breakdown**:
- Desktop vs. Mobile
- Fast / Average / Slow percentages
- 75th percentile values

### CSV Import

Import custom traffic data:

#### Prepare CSV

Required columns:
```csv
date,source,sessions,pageviews,users,bounce_rate
2024-01-01,organic,1234,5678,890,45.2
2024-01-01,direct,567,890,234,52.1
2024-01-01,referral,345,456,123,38.5
```

Optional columns:
- `conversions`
- `revenue`
- `avg_session_duration`
- `pages_per_session`

#### Import Steps

1. Click **"Import CSV"**
2. Upload file
3. Map columns:
   - Date → `date`
   - Source → `source`
   - Sessions → `sessions`
   - etc.
4. Preview data
5. Click **"Import"**

**Validation**:
- Date format: YYYY-MM-DD
- Numeric values for metrics
- Duplicate detection (date + source)

### Traffic Dashboard

Unified view of all traffic sources:

#### Summary Cards

- **Total Sessions**: All sources combined
- **Total Users**: Unique users
- **Total Pageviews**: Page views
- **Avg Bounce Rate**: Weighted average
- **Conversions**: Goal completions
- **Revenue**: Total revenue (if available)

#### Traffic Over Time Chart

**Chart Options**:
- **Metric**: Sessions, Users, Pageviews, Conversions
- **Breakdown**: By source, medium, device, country
- **Date Range**: Last 7, 14, 30, 90 days, custom

**Interactions**:
- Hover for exact values
- Click legend to toggle sources
- Zoom into specific date range

#### Source Breakdown

**Table Columns**:
- **Source**: Traffic source (organic, direct, referral, social, etc.)
- **Sessions**: Session count
- **% of Total**: Percentage of total traffic
- **Users**: Unique users
- **Pageviews**: Total pageviews
- **Bounce Rate**: Percentage bounced
- **Avg Duration**: Average session duration
- **Conversions**: Conversion count
- **Conv Rate**: Conversion rate

**Common Sources**:
- `organic`: Search engines (Google, Bing)
- `direct`: Direct URL entry, bookmarks
- `referral`: Other websites
- `social`: Social media platforms
- `paid`: Paid advertising (PPC)
- `email`: Email campaigns

#### Device Breakdown

Traffic by device type:

- **Desktop**: Traditional computers
- **Mobile**: Smartphones
- **Tablet**: iPads, Android tablets

**Why It Matters**:
- Mobile-first indexing
- Responsive design priorities
- Performance optimization targets

#### Country Breakdown

Geographic traffic distribution:

**Table Columns**:
- **Country**: Country name
- **Sessions**: Session count
- **% of Total**: Percentage
- **Avg Duration**: Average session duration
- **Bounce Rate**: Bounce rate
- **Conversions**: Conversions

**Use Cases**:
- Identify growth markets
- Localization priorities
- International SEO opportunities

### Custom Dashboards

Create custom reports:

1. Click **"New Dashboard"**
2. Add widgets:
   - Metric cards
   - Time series charts
   - Breakdown tables
   - Comparison charts
3. Configure each widget:
   - Metric
   - Date range
   - Filters (source, device, country)
4. Save dashboard
5. Share with team (optional)

---

## User Settings

Manage your account preferences:

### Profile

- **Name**: Display name
- **Email**: Login email (cannot change)
- **Avatar**: Profile picture upload

### Notifications

Configure email alerts:

#### Email Preferences

- **Daily Summary**: Overview of all projects (9 AM)
- **Weekly Report**: Detailed performance report (Monday 9 AM)
- **Issue Alerts**: Critical SEO issues detected (immediate)
- **Rank Alerts**: Significant position changes (daily)
- **Link Alerts**: New/lost backlinks (weekly)

#### Alert Thresholds

- **Position Change**: Alert if keyword moves ± X positions (default: 5)
- **Critical Issues**: Alert if X+ critical issues found (default: 10)
- **Link Velocity**: Alert if links gained/lost > X per day (default: 50)

### Security

- **Change Password**: Update account password
- **Two-Factor Authentication**: Enable 2FA (recommended)
- **Active Sessions**: View and revoke active login sessions
- **API Tokens**: Generate personal API tokens for integrations

### Integrations

Manage connected services:

- **Google OAuth**: View/revoke Google account connection
- **Third-party Apps**: Authorized applications with API access

---

## Tips and Best Practices

### Audit Best Practices

1. **Run audits regularly**: Weekly for active sites, monthly for stable sites
2. **Compare runs**: Always compare to previous audit to track progress
3. **Prioritize Critical/High issues**: Focus on impact, not volume
4. **Fix systematically**: Address one issue type at a time
5. **Re-audit after fixes**: Verify issues are resolved

### Keyword Strategy

1. **Connect GSC early**: Historical data is valuable
2. **Review opportunities weekly**: Low-hanging fruit decays quickly
3. **Track competitors**: Monitor your position vs. theirs
4. **Use clusters for content**: One comprehensive page per cluster
5. **Diversify anchors**: Natural link profile prevents penalties

### Rank Tracking Strategy

1. **Start with priority keywords**: Focus on business-critical terms first
2. **Add long-tail later**: Bulk import after core keywords
3. **Use GSC provider**: Free, accurate, sufficient for most
4. **Check SERP snapshots**: Context matters (features, competitors)
5. **Set realistic expectations**: Rankings fluctuate daily

### Backlink Strategy

1. **Ingest monthly**: Track link velocity
2. **Analyze anchor distribution**: Healthy profile prevents penalties
3. **Monitor competitors**: Link intersect reveals opportunities
4. **Quality over quantity**: 10 relevant links > 100 spam links
5. **Disavow toxic links**: Use Google Search Console disavow tool

### PPC Optimization

1. **Check overlap weekly**: Reduce wasted spend
2. **Pause high-organic keywords**: Save budget for competitive terms
3. **Improve Quality Scores**: Lower CPC, better ad positions
4. **Test ad copy**: A/B test headlines and descriptions
5. **Track conversions**: Optimize for revenue, not just clicks

### Traffic Analysis

1. **Cross-reference sources**: Validate data consistency
2. **Segment by device**: Optimize for mobile-first
3. **Track trends**: Look for seasonal patterns
4. **Set goals**: Measure against benchmarks
5. **Export regularly**: Share with stakeholders

---

## Getting More Help

- **Documentation**: Refer to other guides in `/docs`
- **API Reference**: [api-reference.md](./api-reference.md) for integrations
- **Architecture**: [architecture.md](./architecture.md) for technical details
- **Deployment**: [deployment.md](./deployment.md) for production setup

---

**Happy optimizing!** The SEO Platform gives you enterprise-grade tools to improve your organic search performance. Use this guide as a reference as you explore features.
