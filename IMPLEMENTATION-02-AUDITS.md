# Implementation 02: Site Audit Engine

> **PRD Reference:** Section 6.2 Site Audit Engine (HTML + Hybrid JS)
> **Priority:** P0 (Core feature)
> **Sprint:** 1
> **Dependencies:** Projects module, Celery infrastructure

---

## 1) Overview

The Site Audit Engine crawls websites to detect SEO issues, with support for hybrid JavaScript rendering. This is the most complex module, involving:

- Async HTTP crawler with configurable limits
- Playwright integration for JS rendering
- Issue detection rules with severity levels
- Run-to-run diff tracking
- CSV export functionality

**Pipeline Flow:**
```
Start Audit → Crawl HTML → [Render JS] → Analyze Pages → Compute Diff → Complete
```

---

## 2) Data Models

### 2.1 Audit Models

```python
# backend/app/models/audit.py
import uuid
from datetime import datetime
from enum import Enum
from sqlmodel import SQLModel, Field, Relationship, Column, JSON
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from .project import Project


class AuditStatus(str, Enum):
    QUEUED = "queued"
    CRAWLING = "crawling"
    RENDERING = "rendering"
    ANALYZING = "analyzing"
    DIFFING = "diffing"
    COMPLETED = "completed"
    FAILED = "failed"


class IssueSeverity(str, Enum):
    CRITICAL = "critical"
    HIGH = "high"
    MEDIUM = "medium"
    LOW = "low"


class IssueType(str, Enum):
    # Critical
    SERVER_ERROR_5XX = "server_error_5xx"
    REDIRECT_LOOP = "redirect_loop"
    REDIRECT_CHAIN = "redirect_chain"
    BROKEN_INTERNAL_LINK = "broken_internal_link"

    # High
    CLIENT_ERROR_4XX = "client_error_4xx"
    MISSING_TITLE = "missing_title"
    DUPLICATE_TITLE = "duplicate_title"
    MISSING_META_DESCRIPTION = "missing_meta_description"

    # Medium
    TITLE_TOO_LONG = "title_too_long"
    TITLE_TOO_SHORT = "title_too_short"
    META_DESC_TOO_LONG = "meta_desc_too_long"
    META_DESC_TOO_SHORT = "meta_desc_too_short"
    MISSING_H1 = "missing_h1"
    MULTIPLE_H1 = "multiple_h1"
    MISSING_CANONICAL = "missing_canonical"
    CANONICAL_MISMATCH = "canonical_mismatch"
    NON_HTTPS = "non_https"

    # Low
    THIN_CONTENT = "thin_content"
    ORPHAN_PAGE = "orphan_page"


# Issue type to severity mapping
ISSUE_SEVERITY_MAP: dict[IssueType, IssueSeverity] = {
    IssueType.SERVER_ERROR_5XX: IssueSeverity.CRITICAL,
    IssueType.REDIRECT_LOOP: IssueSeverity.CRITICAL,
    IssueType.REDIRECT_CHAIN: IssueSeverity.CRITICAL,
    IssueType.BROKEN_INTERNAL_LINK: IssueSeverity.CRITICAL,
    IssueType.CLIENT_ERROR_4XX: IssueSeverity.HIGH,
    IssueType.MISSING_TITLE: IssueSeverity.HIGH,
    IssueType.DUPLICATE_TITLE: IssueSeverity.HIGH,
    IssueType.MISSING_META_DESCRIPTION: IssueSeverity.HIGH,
    IssueType.TITLE_TOO_LONG: IssueSeverity.MEDIUM,
    IssueType.TITLE_TOO_SHORT: IssueSeverity.MEDIUM,
    IssueType.META_DESC_TOO_LONG: IssueSeverity.MEDIUM,
    IssueType.META_DESC_TOO_SHORT: IssueSeverity.MEDIUM,
    IssueType.MISSING_H1: IssueSeverity.MEDIUM,
    IssueType.MULTIPLE_H1: IssueSeverity.MEDIUM,
    IssueType.MISSING_CANONICAL: IssueSeverity.MEDIUM,
    IssueType.CANONICAL_MISMATCH: IssueSeverity.MEDIUM,
    IssueType.NON_HTTPS: IssueSeverity.MEDIUM,
    IssueType.THIN_CONTENT: IssueSeverity.LOW,
    IssueType.ORPHAN_PAGE: IssueSeverity.LOW,
}


class AuditRunConfig(SQLModel):
    """Snapshot of crawl config at run time"""
    max_pages: int = 1000
    max_depth: int = 10
    crawl_concurrency: int = 5
    user_agent: str = "SEOPlatformBot/1.0"
    respect_robots_txt: bool = True
    enable_js_rendering: bool = False
    js_render_mode: str = "hybrid"
    max_render_pages: int = 100


class AuditRunStats(SQLModel):
    """Aggregated stats for the audit run"""
    pages_crawled: int = 0
    pages_rendered: int = 0
    pages_analyzed: int = 0
    issues_critical: int = 0
    issues_high: int = 0
    issues_medium: int = 0
    issues_low: int = 0
    internal_links: int = 0
    external_links: int = 0
    broken_links: int = 0
    avg_response_time_ms: float = 0


class AuditRun(SQLModel, table=True):
    __tablename__ = "audit_runs"

    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    project_id: uuid.UUID = Field(foreign_key="projects.id", index=True)
    status: AuditStatus = Field(default=AuditStatus.QUEUED)
    config: dict = Field(default_factory=dict, sa_column=Column(JSON))
    stats: dict = Field(default_factory=dict, sa_column=Column(JSON))
    started_at: datetime | None = None
    finished_at: datetime | None = None
    error_message: str | None = None

    # Progress tracking
    progress_pct: int = 0
    progress_message: str | None = None

    # Timestamps
    created_at: datetime = Field(default_factory=datetime.utcnow)

    # Relationships
    project: "Project" = Relationship(back_populates="audit_runs")
    pages: list["CrawledPage"] = Relationship(
        back_populates="audit_run",
        sa_relationship_kwargs={"cascade": "all, delete-orphan"},
    )
    issues: list["AuditIssue"] = Relationship(
        back_populates="audit_run",
        sa_relationship_kwargs={"cascade": "all, delete-orphan"},
    )
    link_edges: list["AuditLinkEdge"] = Relationship(
        back_populates="audit_run",
        sa_relationship_kwargs={"cascade": "all, delete-orphan"},
    )


class CrawledPage(SQLModel, table=True):
    __tablename__ = "crawled_pages"

    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    audit_run_id: uuid.UUID = Field(foreign_key="audit_runs.id", index=True)

    # URL info
    url: str = Field(index=True)
    final_url: str  # After redirects
    depth: int = 0

    # Response info
    status_code: int
    content_type: str | None = None
    response_time_ms: int = 0

    # Redirect chain
    redirect_chain: list[str] = Field(default_factory=list, sa_column=Column(JSON))

    # Extracted data
    title: str | None = None
    meta_description: str | None = None
    canonical: str | None = None
    h1_count: int = 0
    first_h1: str | None = None
    word_count: int = 0
    meta_robots: str | None = None

    # Rendering info
    is_rendered: bool = False
    rendered_at: datetime | None = None
    rendered_title: str | None = None
    rendered_meta_description: str | None = None
    rendered_h1_count: int | None = None
    rendered_word_count: int | None = None
    content_hash: str | None = None  # Hash of rendered HTML

    # Timestamps
    crawled_at: datetime = Field(default_factory=datetime.utcnow)

    # Relationships
    audit_run: "AuditRun" = Relationship(back_populates="pages")


class AuditIssue(SQLModel, table=True):
    __tablename__ = "audit_issues"

    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    audit_run_id: uuid.UUID = Field(foreign_key="audit_runs.id", index=True)
    page_url: str = Field(index=True)

    issue_type: IssueType
    severity: IssueSeverity
    details: dict = Field(default_factory=dict, sa_column=Column(JSON))

    # For diff tracking
    first_seen_run_id: uuid.UUID | None = None
    is_new: bool = True  # New in this run vs previous

    # Relationships
    audit_run: "AuditRun" = Relationship(back_populates="issues")


class AuditLinkEdge(SQLModel, table=True):
    __tablename__ = "audit_link_edges"

    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    audit_run_id: uuid.UUID = Field(foreign_key="audit_runs.id", index=True)

    source_url: str = Field(index=True)
    target_url: str = Field(index=True)
    anchor_text: str | None = None
    is_internal: bool = True
    is_followed: bool = True  # rel=nofollow check
    target_status_code: int | None = None

    # Relationships
    audit_run: "AuditRun" = Relationship(back_populates="link_edges")


# API Response Models
class AuditRunPublic(SQLModel):
    id: uuid.UUID
    project_id: uuid.UUID
    status: AuditStatus
    config: dict
    stats: dict
    progress_pct: int
    progress_message: str | None
    started_at: datetime | None
    finished_at: datetime | None
    error_message: str | None
    created_at: datetime


class AuditRunsPublic(SQLModel):
    items: list[AuditRunPublic]
    total: int


class CrawledPagePublic(SQLModel):
    id: uuid.UUID
    url: str
    final_url: str
    depth: int
    status_code: int
    title: str | None
    meta_description: str | None
    canonical: str | None
    h1_count: int
    word_count: int
    response_time_ms: int
    is_rendered: bool
    crawled_at: datetime


class CrawledPagesPublic(SQLModel):
    items: list[CrawledPagePublic]
    total: int


class AuditIssuePublic(SQLModel):
    id: uuid.UUID
    page_url: str
    issue_type: IssueType
    severity: IssueSeverity
    details: dict
    is_new: bool


class AuditIssuesPublic(SQLModel):
    items: list[AuditIssuePublic]
    total: int
    by_severity: dict[str, int]
    by_type: dict[str, int]
```

---

## 3) Services

### 3.1 HTML Crawler

```python
# backend/app/services/audit/crawler.py
import asyncio
import hashlib
import re
from dataclasses import dataclass
from datetime import datetime
from urllib.parse import urljoin, urlparse
from typing import AsyncGenerator

import httpx
from bs4 import BeautifulSoup
from urllib.robotparser import RobotFileParser

from app.core.config import settings


@dataclass
class CrawlResult:
    url: str
    final_url: str
    status_code: int
    content_type: str | None
    response_time_ms: int
    redirect_chain: list[str]
    html: str | None
    error: str | None = None


@dataclass
class ExtractedData:
    title: str | None
    meta_description: str | None
    canonical: str | None
    h1_count: int
    first_h1: str | None
    word_count: int
    meta_robots: str | None
    internal_links: list[str]
    external_links: list[str]
    content_hash: str


class Crawler:
    """Async HTTP crawler with rate limiting and robots.txt support"""

    def __init__(
        self,
        seed_url: str,
        max_pages: int = 1000,
        max_depth: int = 10,
        concurrency: int = 5,
        user_agent: str = "SEOPlatformBot/1.0",
        respect_robots: bool = True,
        include_patterns: list[str] | None = None,
        exclude_patterns: list[str] | None = None,
    ):
        self.seed_url = seed_url
        self.seed_domain = urlparse(seed_url).netloc
        self.max_pages = max_pages
        self.max_depth = max_depth
        self.concurrency = concurrency
        self.user_agent = user_agent
        self.respect_robots = respect_robots
        self.include_patterns = [re.compile(p) for p in (include_patterns or [])]
        self.exclude_patterns = [re.compile(p) for p in (exclude_patterns or [])]

        self.visited: set[str] = set()
        self.queue: asyncio.Queue[tuple[str, int]] = asyncio.Queue()
        self.robots_parser: RobotFileParser | None = None
        self._stop_event = asyncio.Event()

    async def setup(self) -> None:
        """Initialize robots.txt parser"""
        if self.respect_robots:
            robots_url = f"{urlparse(self.seed_url).scheme}://{self.seed_domain}/robots.txt"
            try:
                async with httpx.AsyncClient() as client:
                    response = await client.get(robots_url, timeout=10)
                    if response.status_code == 200:
                        self.robots_parser = RobotFileParser()
                        self.robots_parser.parse(response.text.splitlines())
            except Exception:
                pass  # No robots.txt or error fetching

    def can_fetch(self, url: str) -> bool:
        """Check if URL is allowed by robots.txt"""
        if not self.respect_robots or not self.robots_parser:
            return True
        return self.robots_parser.can_fetch(self.user_agent, url)

    def is_valid_url(self, url: str) -> bool:
        """Check if URL should be crawled"""
        parsed = urlparse(url)

        # Must be same domain
        if parsed.netloc != self.seed_domain:
            return False

        # Must be HTTP(S)
        if parsed.scheme not in ("http", "https"):
            return False

        # Check include patterns
        if self.include_patterns:
            if not any(p.search(url) for p in self.include_patterns):
                return False

        # Check exclude patterns
        if any(p.search(url) for p in self.exclude_patterns):
            return False

        return True

    def normalize_url(self, url: str) -> str:
        """Normalize URL for deduplication"""
        parsed = urlparse(url)
        # Remove fragment, normalize path
        return f"{parsed.scheme}://{parsed.netloc}{parsed.path}"

    async def fetch(self, url: str) -> CrawlResult:
        """Fetch a single URL"""
        start_time = datetime.utcnow()
        redirect_chain = []

        try:
            async with httpx.AsyncClient(
                follow_redirects=True,
                timeout=30,
                headers={"User-Agent": self.user_agent},
            ) as client:
                response = await client.get(url)

                # Track redirect chain
                if response.history:
                    redirect_chain = [str(r.url) for r in response.history]

                elapsed_ms = int((datetime.utcnow() - start_time).total_seconds() * 1000)

                # Only parse HTML
                content_type = response.headers.get("content-type", "")
                html = None
                if "text/html" in content_type:
                    html = response.text

                return CrawlResult(
                    url=url,
                    final_url=str(response.url),
                    status_code=response.status_code,
                    content_type=content_type,
                    response_time_ms=elapsed_ms,
                    redirect_chain=redirect_chain,
                    html=html,
                )

        except httpx.TimeoutException:
            return CrawlResult(
                url=url,
                final_url=url,
                status_code=0,
                content_type=None,
                response_time_ms=30000,
                redirect_chain=[],
                html=None,
                error="timeout",
            )
        except Exception as e:
            return CrawlResult(
                url=url,
                final_url=url,
                status_code=0,
                content_type=None,
                response_time_ms=0,
                redirect_chain=[],
                html=None,
                error=str(e),
            )

    def extract_data(self, html: str, base_url: str) -> ExtractedData:
        """Extract SEO-relevant data from HTML"""
        soup = BeautifulSoup(html, "html.parser")

        # Title
        title_tag = soup.find("title")
        title = title_tag.get_text(strip=True) if title_tag else None

        # Meta description
        meta_desc = soup.find("meta", attrs={"name": "description"})
        meta_description = meta_desc.get("content") if meta_desc else None

        # Canonical
        canonical_tag = soup.find("link", attrs={"rel": "canonical"})
        canonical = canonical_tag.get("href") if canonical_tag else None

        # H1s
        h1_tags = soup.find_all("h1")
        h1_count = len(h1_tags)
        first_h1 = h1_tags[0].get_text(strip=True) if h1_tags else None

        # Word count (approximate)
        text = soup.get_text(separator=" ", strip=True)
        word_count = len(text.split())

        # Meta robots
        meta_robots_tag = soup.find("meta", attrs={"name": "robots"})
        meta_robots = meta_robots_tag.get("content") if meta_robots_tag else None

        # Links
        internal_links = []
        external_links = []
        for link in soup.find_all("a", href=True):
            href = link.get("href")
            absolute_url = urljoin(base_url, href)
            parsed = urlparse(absolute_url)

            if parsed.netloc == self.seed_domain:
                internal_links.append(self.normalize_url(absolute_url))
            elif parsed.scheme in ("http", "https"):
                external_links.append(absolute_url)

        # Content hash
        content_hash = hashlib.md5(html.encode()).hexdigest()

        return ExtractedData(
            title=title,
            meta_description=meta_description,
            canonical=canonical,
            h1_count=h1_count,
            first_h1=first_h1,
            word_count=word_count,
            meta_robots=meta_robots,
            internal_links=list(set(internal_links)),
            external_links=list(set(external_links)),
            content_hash=content_hash,
        )

    async def crawl(self) -> AsyncGenerator[tuple[CrawlResult, ExtractedData | None, int], None]:
        """
        Crawl starting from seed URL.
        Yields (CrawlResult, ExtractedData, depth) for each page.
        """
        await self.setup()

        # Start with seed URL
        await self.queue.put((self.seed_url, 0))
        self.visited.add(self.normalize_url(self.seed_url))

        workers = []
        results_queue: asyncio.Queue = asyncio.Queue()

        async def worker():
            while not self._stop_event.is_set():
                try:
                    url, depth = await asyncio.wait_for(
                        self.queue.get(), timeout=5.0
                    )
                except asyncio.TimeoutError:
                    if self.queue.empty():
                        break
                    continue

                if len(self.visited) >= self.max_pages:
                    self._stop_event.set()
                    break

                if not self.can_fetch(url):
                    continue

                result = await self.fetch(url)

                extracted = None
                if result.html:
                    extracted = self.extract_data(result.html, result.final_url)

                    # Add internal links to queue
                    if depth < self.max_depth:
                        for link in extracted.internal_links:
                            normalized = self.normalize_url(link)
                            if normalized not in self.visited and self.is_valid_url(link):
                                self.visited.add(normalized)
                                await self.queue.put((link, depth + 1))

                await results_queue.put((result, extracted, depth))
                self.queue.task_done()

        # Start workers
        for _ in range(self.concurrency):
            workers.append(asyncio.create_task(worker()))

        # Yield results as they come in
        while True:
            try:
                result = await asyncio.wait_for(results_queue.get(), timeout=1.0)
                yield result
            except asyncio.TimeoutError:
                # Check if all workers are done
                if all(w.done() for w in workers) and results_queue.empty():
                    break

        # Cleanup
        for w in workers:
            w.cancel()

    def stop(self) -> None:
        """Stop the crawl"""
        self._stop_event.set()
```

### 3.2 JavaScript Renderer

```python
# backend/app/services/audit/renderer.py
import asyncio
from dataclasses import dataclass
from datetime import datetime
from typing import Optional

from playwright.async_api import async_playwright, Browser, Page


@dataclass
class RenderResult:
    url: str
    html: str
    title: str | None
    meta_description: str | None
    h1_count: int
    word_count: int
    render_time_ms: int
    error: str | None = None


class Renderer:
    """Playwright-based JavaScript renderer"""

    def __init__(
        self,
        timeout_ms: int = 30000,
        user_agent: str = "SEOPlatformBot/1.0",
    ):
        self.timeout_ms = timeout_ms
        self.user_agent = user_agent
        self._browser: Browser | None = None
        self._playwright = None

    async def start(self) -> None:
        """Initialize Playwright browser"""
        self._playwright = await async_playwright().start()
        self._browser = await self._playwright.chromium.launch(
            headless=True,
            args=["--no-sandbox", "--disable-setuid-sandbox"],
        )

    async def stop(self) -> None:
        """Close browser and cleanup"""
        if self._browser:
            await self._browser.close()
        if self._playwright:
            await self._playwright.stop()

    async def render(self, url: str) -> RenderResult:
        """Render a page with JavaScript execution"""
        if not self._browser:
            await self.start()

        start_time = datetime.utcnow()

        try:
            context = await self._browser.new_context(
                user_agent=self.user_agent,
            )
            page = await context.new_page()

            # Navigate and wait for network idle
            await page.goto(url, wait_until="networkidle", timeout=self.timeout_ms)

            # Wait a bit more for JS to settle
            await asyncio.sleep(1)

            # Extract rendered content
            html = await page.content()

            # Extract specific elements via JavaScript
            data = await page.evaluate("""
                () => {
                    const title = document.title || null;
                    const metaDesc = document.querySelector('meta[name="description"]');
                    const h1s = document.querySelectorAll('h1');
                    const bodyText = document.body ? document.body.innerText : '';

                    return {
                        title: title,
                        meta_description: metaDesc ? metaDesc.content : null,
                        h1_count: h1s.length,
                        word_count: bodyText.split(/\\s+/).filter(w => w).length
                    };
                }
            """)

            await context.close()

            elapsed_ms = int((datetime.utcnow() - start_time).total_seconds() * 1000)

            return RenderResult(
                url=url,
                html=html,
                title=data["title"],
                meta_description=data["meta_description"],
                h1_count=data["h1_count"],
                word_count=data["word_count"],
                render_time_ms=elapsed_ms,
            )

        except Exception as e:
            elapsed_ms = int((datetime.utcnow() - start_time).total_seconds() * 1000)
            return RenderResult(
                url=url,
                html="",
                title=None,
                meta_description=None,
                h1_count=0,
                word_count=0,
                render_time_ms=elapsed_ms,
                error=str(e),
            )

    def should_render(
        self,
        html_word_count: int,
        has_title: bool,
        has_h1: bool,
        min_word_threshold: int = 50,
    ) -> bool:
        """
        Determine if a page needs JS rendering (hybrid mode).
        Returns True if:
        - HTML word count is below threshold (thin/empty body)
        - Missing critical elements that JS might populate
        """
        if html_word_count < min_word_threshold:
            return True
        if not has_title and not has_h1:
            return True
        return False
```

### 3.3 Issue Analyzer

```python
# backend/app/services/audit/analyzer.py
from dataclasses import dataclass
from typing import Generator

from app.models.audit import (
    IssueType,
    IssueSeverity,
    ISSUE_SEVERITY_MAP,
    CrawledPage,
)


@dataclass
class DetectedIssue:
    page_url: str
    issue_type: IssueType
    severity: IssueSeverity
    details: dict


class IssueAnalyzer:
    """Analyze crawled pages and detect SEO issues"""

    # Configurable thresholds
    TITLE_MIN_LENGTH = 10
    TITLE_MAX_LENGTH = 60
    META_DESC_MIN_LENGTH = 50
    META_DESC_MAX_LENGTH = 160
    THIN_CONTENT_THRESHOLD = 300
    REDIRECT_CHAIN_MAX = 3

    def __init__(self, seed_url: str):
        self.seed_url = seed_url
        self.seed_is_https = seed_url.startswith("https://")

    def analyze_page(self, page: CrawledPage) -> Generator[DetectedIssue, None, None]:
        """Analyze a single page and yield detected issues"""

        # Status code issues
        if 500 <= page.status_code < 600:
            yield DetectedIssue(
                page_url=page.url,
                issue_type=IssueType.SERVER_ERROR_5XX,
                severity=IssueSeverity.CRITICAL,
                details={"status_code": page.status_code},
            )
            return  # Don't analyze error pages further

        if 400 <= page.status_code < 500:
            yield DetectedIssue(
                page_url=page.url,
                issue_type=IssueType.CLIENT_ERROR_4XX,
                severity=IssueSeverity.HIGH,
                details={"status_code": page.status_code},
            )
            return

        # Redirect issues
        if page.redirect_chain:
            if len(page.redirect_chain) > self.REDIRECT_CHAIN_MAX:
                yield DetectedIssue(
                    page_url=page.url,
                    issue_type=IssueType.REDIRECT_CHAIN,
                    severity=IssueSeverity.CRITICAL,
                    details={
                        "chain_length": len(page.redirect_chain),
                        "chain": page.redirect_chain,
                    },
                )

            # Check for loops
            if page.url in page.redirect_chain or page.final_url in page.redirect_chain[:-1]:
                yield DetectedIssue(
                    page_url=page.url,
                    issue_type=IssueType.REDIRECT_LOOP,
                    severity=IssueSeverity.CRITICAL,
                    details={"chain": page.redirect_chain},
                )

        # Use rendered values if available, otherwise HTML values
        title = page.rendered_title if page.is_rendered else page.title
        meta_desc = page.rendered_meta_description if page.is_rendered else page.meta_description
        h1_count = page.rendered_h1_count if page.is_rendered and page.rendered_h1_count else page.h1_count
        word_count = page.rendered_word_count if page.is_rendered and page.rendered_word_count else page.word_count

        # Title issues
        if not title:
            yield DetectedIssue(
                page_url=page.url,
                issue_type=IssueType.MISSING_TITLE,
                severity=IssueSeverity.HIGH,
                details={},
            )
        elif len(title) < self.TITLE_MIN_LENGTH:
            yield DetectedIssue(
                page_url=page.url,
                issue_type=IssueType.TITLE_TOO_SHORT,
                severity=IssueSeverity.MEDIUM,
                details={"length": len(title), "title": title},
            )
        elif len(title) > self.TITLE_MAX_LENGTH:
            yield DetectedIssue(
                page_url=page.url,
                issue_type=IssueType.TITLE_TOO_LONG,
                severity=IssueSeverity.MEDIUM,
                details={"length": len(title), "title": title},
            )

        # Meta description issues
        if not meta_desc:
            yield DetectedIssue(
                page_url=page.url,
                issue_type=IssueType.MISSING_META_DESCRIPTION,
                severity=IssueSeverity.HIGH,
                details={},
            )
        elif len(meta_desc) < self.META_DESC_MIN_LENGTH:
            yield DetectedIssue(
                page_url=page.url,
                issue_type=IssueType.META_DESC_TOO_SHORT,
                severity=IssueSeverity.MEDIUM,
                details={"length": len(meta_desc)},
            )
        elif len(meta_desc) > self.META_DESC_MAX_LENGTH:
            yield DetectedIssue(
                page_url=page.url,
                issue_type=IssueType.META_DESC_TOO_LONG,
                severity=IssueSeverity.MEDIUM,
                details={"length": len(meta_desc)},
            )

        # H1 issues
        if h1_count == 0:
            yield DetectedIssue(
                page_url=page.url,
                issue_type=IssueType.MISSING_H1,
                severity=IssueSeverity.MEDIUM,
                details={},
            )
        elif h1_count > 1:
            yield DetectedIssue(
                page_url=page.url,
                issue_type=IssueType.MULTIPLE_H1,
                severity=IssueSeverity.MEDIUM,
                details={"count": h1_count},
            )

        # Canonical issues
        if not page.canonical:
            yield DetectedIssue(
                page_url=page.url,
                issue_type=IssueType.MISSING_CANONICAL,
                severity=IssueSeverity.MEDIUM,
                details={},
            )
        elif page.canonical != page.final_url and page.canonical != page.url:
            # Check if canonical points to different domain
            from urllib.parse import urlparse
            canonical_domain = urlparse(page.canonical).netloc
            page_domain = urlparse(page.final_url).netloc
            if canonical_domain != page_domain:
                yield DetectedIssue(
                    page_url=page.url,
                    issue_type=IssueType.CANONICAL_MISMATCH,
                    severity=IssueSeverity.MEDIUM,
                    details={"canonical": page.canonical, "page_url": page.final_url},
                )

        # HTTPS issues
        if self.seed_is_https and not page.final_url.startswith("https://"):
            yield DetectedIssue(
                page_url=page.url,
                issue_type=IssueType.NON_HTTPS,
                severity=IssueSeverity.MEDIUM,
                details={"final_url": page.final_url},
            )

        # Thin content
        if word_count < self.THIN_CONTENT_THRESHOLD:
            yield DetectedIssue(
                page_url=page.url,
                issue_type=IssueType.THIN_CONTENT,
                severity=IssueSeverity.LOW,
                details={"word_count": word_count},
            )


class DuplicateDetector:
    """Detect duplicate titles and content across pages"""

    def __init__(self):
        self.titles: dict[str, list[str]] = {}  # title -> [urls]
        self.content_hashes: dict[str, list[str]] = {}  # hash -> [urls]

    def add_page(self, url: str, title: str | None, content_hash: str | None) -> None:
        """Register a page for duplicate detection"""
        if title:
            normalized = title.strip().lower()
            if normalized not in self.titles:
                self.titles[normalized] = []
            self.titles[normalized].append(url)

        if content_hash:
            if content_hash not in self.content_hashes:
                self.content_hashes[content_hash] = []
            self.content_hashes[content_hash].append(url)

    def get_duplicate_title_issues(self) -> Generator[DetectedIssue, None, None]:
        """Yield issues for duplicate titles"""
        for title, urls in self.titles.items():
            if len(urls) > 1:
                for url in urls:
                    yield DetectedIssue(
                        page_url=url,
                        issue_type=IssueType.DUPLICATE_TITLE,
                        severity=IssueSeverity.HIGH,
                        details={
                            "title": title,
                            "duplicate_urls": [u for u in urls if u != url],
                        },
                    )
```

### 3.4 Diff Engine

```python
# backend/app/services/audit/differ.py
from dataclasses import dataclass
from typing import Optional
import uuid

from sqlmodel import Session, select

from app.models.audit import AuditRun, AuditIssue, IssueType


@dataclass
class DiffResult:
    new_issues: int
    resolved_issues: int
    unchanged_issues: int


class AuditDiffer:
    """Compare audit runs to track new/resolved issues"""

    def __init__(self, session: Session):
        self.session = session

    def get_previous_run(self, project_id: uuid.UUID, current_run_id: uuid.UUID) -> Optional[AuditRun]:
        """Get the most recent completed audit run before the current one"""
        statement = (
            select(AuditRun)
            .where(AuditRun.project_id == project_id)
            .where(AuditRun.id != current_run_id)
            .where(AuditRun.status == "completed")
            .order_by(AuditRun.finished_at.desc())
            .limit(1)
        )
        return self.session.exec(statement).first()

    def compute_diff(self, current_run: AuditRun, previous_run: Optional[AuditRun]) -> DiffResult:
        """
        Compare current run issues with previous run.
        Updates is_new and first_seen_run_id on current issues.
        """
        if not previous_run:
            # First run - all issues are new
            for issue in current_run.issues:
                issue.is_new = True
                issue.first_seen_run_id = current_run.id
            self.session.commit()
            return DiffResult(
                new_issues=len(current_run.issues),
                resolved_issues=0,
                unchanged_issues=0,
            )

        # Build lookup of previous issues by (page_url, issue_type)
        previous_issues: dict[tuple[str, IssueType], AuditIssue] = {}
        for issue in previous_run.issues:
            key = (issue.page_url, issue.issue_type)
            previous_issues[key] = issue

        new_count = 0
        unchanged_count = 0

        # Compare current issues
        for issue in current_run.issues:
            key = (issue.page_url, issue.issue_type)
            if key in previous_issues:
                # Issue existed before
                issue.is_new = False
                issue.first_seen_run_id = previous_issues[key].first_seen_run_id
                unchanged_count += 1
                del previous_issues[key]  # Remove from remaining
            else:
                # New issue
                issue.is_new = True
                issue.first_seen_run_id = current_run.id
                new_count += 1

        # Remaining in previous_issues are resolved
        resolved_count = len(previous_issues)

        self.session.commit()

        return DiffResult(
            new_issues=new_count,
            resolved_issues=resolved_count,
            unchanged_issues=unchanged_count,
        )
```

---

## 4) Celery Tasks

```python
# backend/app/tasks/audit.py
import uuid
from datetime import datetime
from celery import shared_task, chain
from sqlmodel import Session

from app.core.db import engine
from app.models.audit import (
    AuditRun,
    AuditStatus,
    CrawledPage,
    AuditIssue,
    AuditLinkEdge,
    AuditRunStats,
)
from app.models.project import Project
from app.services.audit.crawler import Crawler
from app.services.audit.renderer import Renderer
from app.services.audit.analyzer import IssueAnalyzer, DuplicateDetector
from app.services.audit.differ import AuditDiffer


@shared_task(bind=True)
def run_audit(self, project_id: str, audit_run_id: str) -> dict:
    """
    Main audit task - orchestrates the full pipeline.
    Uses Celery chain for sequential steps.
    """
    return chain(
        crawl_pages.s(project_id, audit_run_id),
        render_pages.s(project_id, audit_run_id),
        analyze_pages.s(project_id, audit_run_id),
        compute_diff.s(project_id, audit_run_id),
    ).apply_async()


@shared_task(bind=True)
def crawl_pages(self, project_id: str, audit_run_id: str) -> dict:
    """Crawl all pages starting from seed URL"""
    with Session(engine) as session:
        audit_run = session.get(AuditRun, uuid.UUID(audit_run_id))
        project = session.get(Project, uuid.UUID(project_id))

        if not audit_run or not project:
            raise ValueError("Audit run or project not found")

        # Update status
        audit_run.status = AuditStatus.CRAWLING
        audit_run.started_at = datetime.utcnow()
        session.commit()

        config = audit_run.config
        crawler = Crawler(
            seed_url=project.seed_url,
            max_pages=config.get("max_pages", 1000),
            max_depth=config.get("max_depth", 10),
            concurrency=config.get("crawl_concurrency", 5),
            user_agent=config.get("user_agent", "SEOPlatformBot/1.0"),
            respect_robots=config.get("respect_robots_txt", True),
        )

        import asyncio

        async def run_crawl():
            pages_crawled = 0
            total_response_time = 0

            async for result, extracted, depth in crawler.crawl():
                # Save page
                page = CrawledPage(
                    audit_run_id=audit_run.id,
                    url=result.url,
                    final_url=result.final_url,
                    depth=depth,
                    status_code=result.status_code,
                    content_type=result.content_type,
                    response_time_ms=result.response_time_ms,
                    redirect_chain=result.redirect_chain,
                )

                if extracted:
                    page.title = extracted.title
                    page.meta_description = extracted.meta_description
                    page.canonical = extracted.canonical
                    page.h1_count = extracted.h1_count
                    page.first_h1 = extracted.first_h1
                    page.word_count = extracted.word_count
                    page.meta_robots = extracted.meta_robots
                    page.content_hash = extracted.content_hash

                    # Save link edges
                    for link in extracted.internal_links:
                        edge = AuditLinkEdge(
                            audit_run_id=audit_run.id,
                            source_url=result.final_url,
                            target_url=link,
                            is_internal=True,
                        )
                        session.add(edge)

                session.add(page)
                pages_crawled += 1
                total_response_time += result.response_time_ms

                # Update progress
                if pages_crawled % 10 == 0:
                    progress = min(int((pages_crawled / config.get("max_pages", 1000)) * 100), 100)
                    audit_run.progress_pct = progress
                    audit_run.progress_message = f"Crawled {pages_crawled} pages"
                    session.commit()

            return pages_crawled, total_response_time

        pages_crawled, total_response_time = asyncio.run(run_crawl())

        # Update stats
        audit_run.stats["pages_crawled"] = pages_crawled
        audit_run.stats["avg_response_time_ms"] = (
            total_response_time / pages_crawled if pages_crawled > 0 else 0
        )
        session.commit()

        return {"pages_crawled": pages_crawled}


@shared_task(bind=True)
def render_pages(self, crawl_result: dict, project_id: str, audit_run_id: str) -> dict:
    """Render pages that need JavaScript execution"""
    with Session(engine) as session:
        audit_run = session.get(AuditRun, uuid.UUID(audit_run_id))
        if not audit_run:
            raise ValueError("Audit run not found")

        config = audit_run.config
        if not config.get("enable_js_rendering", False):
            return {"pages_rendered": 0}

        audit_run.status = AuditStatus.RENDERING
        session.commit()

        # Get pages that need rendering
        pages_to_render = []
        for page in audit_run.pages:
            if page.status_code == 200:
                renderer = Renderer()
                if config.get("js_render_mode") == "always":
                    pages_to_render.append(page)
                elif config.get("js_render_mode") == "hybrid":
                    if renderer.should_render(
                        page.word_count,
                        page.title is not None,
                        page.h1_count > 0,
                    ):
                        pages_to_render.append(page)

        # Limit to budget
        max_render = config.get("max_render_pages", 100)
        pages_to_render = pages_to_render[:max_render]

        import asyncio

        async def run_render():
            renderer = Renderer()
            await renderer.start()

            rendered_count = 0
            try:
                for page in pages_to_render:
                    result = await renderer.render(page.final_url)
                    if not result.error:
                        page.is_rendered = True
                        page.rendered_at = datetime.utcnow()
                        page.rendered_title = result.title
                        page.rendered_meta_description = result.meta_description
                        page.rendered_h1_count = result.h1_count
                        page.rendered_word_count = result.word_count
                        rendered_count += 1

                    # Update progress
                    progress = int((rendered_count / len(pages_to_render)) * 100)
                    audit_run.progress_pct = progress
                    audit_run.progress_message = f"Rendered {rendered_count}/{len(pages_to_render)} pages"
                    session.commit()
            finally:
                await renderer.stop()

            return rendered_count

        pages_rendered = asyncio.run(run_render())

        audit_run.stats["pages_rendered"] = pages_rendered
        session.commit()

        return {"pages_rendered": pages_rendered}


@shared_task(bind=True)
def analyze_pages(self, render_result: dict, project_id: str, audit_run_id: str) -> dict:
    """Analyze pages and detect issues"""
    with Session(engine) as session:
        audit_run = session.get(AuditRun, uuid.UUID(audit_run_id))
        project = session.get(Project, uuid.UUID(project_id))

        if not audit_run or not project:
            raise ValueError("Audit run or project not found")

        audit_run.status = AuditStatus.ANALYZING
        session.commit()

        analyzer = IssueAnalyzer(project.seed_url)
        duplicate_detector = DuplicateDetector()

        issues_by_severity = {"critical": 0, "high": 0, "medium": 0, "low": 0}
        issues_by_type: dict[str, int] = {}

        # First pass: collect data for duplicate detection
        for page in audit_run.pages:
            title = page.rendered_title if page.is_rendered else page.title
            duplicate_detector.add_page(page.url, title, page.content_hash)

        # Second pass: analyze pages
        for page in audit_run.pages:
            for detected in analyzer.analyze_page(page):
                issue = AuditIssue(
                    audit_run_id=audit_run.id,
                    page_url=detected.page_url,
                    issue_type=detected.issue_type,
                    severity=detected.severity,
                    details=detected.details,
                )
                session.add(issue)
                issues_by_severity[detected.severity.value] += 1
                issues_by_type[detected.issue_type.value] = (
                    issues_by_type.get(detected.issue_type.value, 0) + 1
                )

        # Add duplicate title issues
        for detected in duplicate_detector.get_duplicate_title_issues():
            issue = AuditIssue(
                audit_run_id=audit_run.id,
                page_url=detected.page_url,
                issue_type=detected.issue_type,
                severity=detected.severity,
                details=detected.details,
            )
            session.add(issue)
            issues_by_severity[detected.severity.value] += 1
            issues_by_type[detected.issue_type.value] = (
                issues_by_type.get(detected.issue_type.value, 0) + 1
            )

        # Update stats
        audit_run.stats["issues_critical"] = issues_by_severity["critical"]
        audit_run.stats["issues_high"] = issues_by_severity["high"]
        audit_run.stats["issues_medium"] = issues_by_severity["medium"]
        audit_run.stats["issues_low"] = issues_by_severity["low"]
        session.commit()

        return {
            "issues_by_severity": issues_by_severity,
            "issues_by_type": issues_by_type,
        }


@shared_task(bind=True)
def compute_diff(self, analyze_result: dict, project_id: str, audit_run_id: str) -> dict:
    """Compare with previous run and mark new/resolved issues"""
    with Session(engine) as session:
        audit_run = session.get(AuditRun, uuid.UUID(audit_run_id))
        project = session.get(Project, uuid.UUID(project_id))

        if not audit_run or not project:
            raise ValueError("Audit run or project not found")

        audit_run.status = AuditStatus.DIFFING
        session.commit()

        differ = AuditDiffer(session)
        previous_run = differ.get_previous_run(project.id, audit_run.id)
        diff_result = differ.compute_diff(audit_run, previous_run)

        # Complete the audit
        audit_run.status = AuditStatus.COMPLETED
        audit_run.finished_at = datetime.utcnow()
        audit_run.progress_pct = 100
        audit_run.progress_message = "Audit complete"

        # Update project last audit timestamp
        project.last_audit_at = datetime.utcnow()
        session.commit()

        return {
            "new_issues": diff_result.new_issues,
            "resolved_issues": diff_result.resolved_issues,
            "unchanged_issues": diff_result.unchanged_issues,
        }
```

---

## 5) API Routes

```python
# backend/app/api/routes/audits.py
import uuid
from typing import Any
from datetime import datetime
from io import StringIO
import csv

from fastapi import APIRouter, Depends, HTTPException, Query
from fastapi.responses import StreamingResponse
from sqlmodel import select, func

from app.api.deps import CurrentUser, SessionDep
from app.models.project import Project
from app.models.audit import (
    AuditRun,
    AuditRunPublic,
    AuditRunsPublic,
    AuditStatus,
    CrawledPage,
    CrawledPagePublic,
    CrawledPagesPublic,
    AuditIssue,
    AuditIssuePublic,
    AuditIssuesPublic,
    IssueSeverity,
    IssueType,
)
from app.tasks.audit import run_audit

router = APIRouter(prefix="/projects/{project_id}/audits", tags=["audits"])


def get_project_or_404(
    session: SessionDep, project_id: uuid.UUID, current_user: CurrentUser
) -> Project:
    project = session.get(Project, project_id)
    if not project:
        raise HTTPException(status_code=404, detail="Project not found")
    if project.created_by_id != current_user.id:
        raise HTTPException(status_code=403, detail="Not authorized")
    return project


@router.post("/", response_model=AuditRunPublic)
def start_audit(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
) -> Any:
    """Start a new audit run for the project"""
    project = get_project_or_404(session, project_id, current_user)

    # Create audit run with project settings snapshot
    settings = project.settings or {}
    audit_run = AuditRun(
        project_id=project.id,
        config={
            "max_pages": settings.get("max_pages", 1000),
            "max_depth": settings.get("max_depth", 10),
            "crawl_concurrency": settings.get("crawl_concurrency", 5),
            "user_agent": settings.get("user_agent", "SEOPlatformBot/1.0"),
            "respect_robots_txt": settings.get("respect_robots_txt", True),
            "enable_js_rendering": settings.get("enable_js_rendering", False),
            "js_render_mode": settings.get("js_render_mode", "hybrid"),
            "max_render_pages": settings.get("max_render_pages", 100),
        },
        stats={},
    )
    session.add(audit_run)
    session.commit()
    session.refresh(audit_run)

    # Queue the Celery task
    run_audit.delay(str(project.id), str(audit_run.id))

    return audit_run


@router.get("/", response_model=AuditRunsPublic)
def list_audits(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
    skip: int = Query(default=0, ge=0),
    limit: int = Query(default=20, le=100),
) -> Any:
    """List all audit runs for the project"""
    project = get_project_or_404(session, project_id, current_user)

    count_statement = (
        select(func.count())
        .select_from(AuditRun)
        .where(AuditRun.project_id == project.id)
    )
    total = session.exec(count_statement).one()

    statement = (
        select(AuditRun)
        .where(AuditRun.project_id == project.id)
        .offset(skip)
        .limit(limit)
        .order_by(AuditRun.created_at.desc())
    )
    audits = session.exec(statement).all()

    return AuditRunsPublic(items=audits, total=total)


@router.get("/{audit_id}", response_model=AuditRunPublic)
def get_audit(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
    audit_id: uuid.UUID,
) -> Any:
    """Get a specific audit run"""
    project = get_project_or_404(session, project_id, current_user)

    audit = session.get(AuditRun, audit_id)
    if not audit or audit.project_id != project.id:
        raise HTTPException(status_code=404, detail="Audit not found")

    return audit


@router.get("/{audit_id}/issues", response_model=AuditIssuesPublic)
def get_audit_issues(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
    audit_id: uuid.UUID,
    skip: int = Query(default=0, ge=0),
    limit: int = Query(default=50, le=200),
    severity: IssueSeverity | None = None,
    issue_type: IssueType | None = None,
    is_new: bool | None = None,
) -> Any:
    """Get issues for an audit run with filtering"""
    project = get_project_or_404(session, project_id, current_user)

    audit = session.get(AuditRun, audit_id)
    if not audit or audit.project_id != project.id:
        raise HTTPException(status_code=404, detail="Audit not found")

    # Build query with filters
    statement = select(AuditIssue).where(AuditIssue.audit_run_id == audit_id)

    if severity:
        statement = statement.where(AuditIssue.severity == severity)
    if issue_type:
        statement = statement.where(AuditIssue.issue_type == issue_type)
    if is_new is not None:
        statement = statement.where(AuditIssue.is_new == is_new)

    # Count
    count_statement = select(func.count()).select_from(statement.subquery())
    total = session.exec(count_statement).one()

    # Paginate
    statement = statement.offset(skip).limit(limit).order_by(AuditIssue.severity)
    issues = session.exec(statement).all()

    # Aggregate counts
    by_severity = {}
    by_type = {}
    for sev in IssueSeverity:
        count = session.exec(
            select(func.count())
            .select_from(AuditIssue)
            .where(AuditIssue.audit_run_id == audit_id)
            .where(AuditIssue.severity == sev)
        ).one()
        by_severity[sev.value] = count

    for it in IssueType:
        count = session.exec(
            select(func.count())
            .select_from(AuditIssue)
            .where(AuditIssue.audit_run_id == audit_id)
            .where(AuditIssue.issue_type == it)
        ).one()
        if count > 0:
            by_type[it.value] = count

    return AuditIssuesPublic(
        items=issues,
        total=total,
        by_severity=by_severity,
        by_type=by_type,
    )


@router.get("/{audit_id}/pages", response_model=CrawledPagesPublic)
def get_audit_pages(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
    audit_id: uuid.UUID,
    skip: int = Query(default=0, ge=0),
    limit: int = Query(default=50, le=200),
    status_code: int | None = None,
    is_rendered: bool | None = None,
) -> Any:
    """Get crawled pages for an audit run"""
    project = get_project_or_404(session, project_id, current_user)

    audit = session.get(AuditRun, audit_id)
    if not audit or audit.project_id != project.id:
        raise HTTPException(status_code=404, detail="Audit not found")

    statement = select(CrawledPage).where(CrawledPage.audit_run_id == audit_id)

    if status_code:
        statement = statement.where(CrawledPage.status_code == status_code)
    if is_rendered is not None:
        statement = statement.where(CrawledPage.is_rendered == is_rendered)

    count_statement = select(func.count()).select_from(statement.subquery())
    total = session.exec(count_statement).one()

    statement = statement.offset(skip).limit(limit).order_by(CrawledPage.crawled_at)
    pages = session.exec(statement).all()

    return CrawledPagesPublic(items=pages, total=total)


@router.get("/{audit_id}/pages/{page_id}", response_model=CrawledPagePublic)
def get_page_detail(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
    audit_id: uuid.UUID,
    page_id: uuid.UUID,
) -> Any:
    """Get detailed info for a single crawled page"""
    project = get_project_or_404(session, project_id, current_user)

    audit = session.get(AuditRun, audit_id)
    if not audit or audit.project_id != project.id:
        raise HTTPException(status_code=404, detail="Audit not found")

    page = session.get(CrawledPage, page_id)
    if not page or page.audit_run_id != audit_id:
        raise HTTPException(status_code=404, detail="Page not found")

    return page


@router.get("/{audit_id}/export/issues.csv")
def export_issues_csv(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
    audit_id: uuid.UUID,
    severity: IssueSeverity | None = None,
    issue_type: IssueType | None = None,
) -> StreamingResponse:
    """Export issues as CSV"""
    project = get_project_or_404(session, project_id, current_user)

    audit = session.get(AuditRun, audit_id)
    if not audit or audit.project_id != project.id:
        raise HTTPException(status_code=404, detail="Audit not found")

    statement = select(AuditIssue).where(AuditIssue.audit_run_id == audit_id)
    if severity:
        statement = statement.where(AuditIssue.severity == severity)
    if issue_type:
        statement = statement.where(AuditIssue.issue_type == issue_type)

    issues = session.exec(statement).all()

    # Generate CSV
    output = StringIO()
    writer = csv.writer(output)
    writer.writerow(["URL", "Issue Type", "Severity", "Is New", "Details"])

    for issue in issues:
        writer.writerow([
            issue.page_url,
            issue.issue_type.value,
            issue.severity.value,
            "Yes" if issue.is_new else "No",
            str(issue.details),
        ])

    output.seek(0)

    return StreamingResponse(
        iter([output.getvalue()]),
        media_type="text/csv",
        headers={
            "Content-Disposition": f"attachment; filename=audit_{audit_id}_issues.csv"
        },
    )
```

---

## 6) Frontend Pages

### 6.1 Audit Runs List

```tsx
// frontend/src/routes/_layout/projects/$projectId/audits/index.tsx
import { createFileRoute, Link } from "@tanstack/react-router"
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query"
import { AuditsService } from "@/client"
import { Button } from "@/components/ui/button"
import { Badge } from "@/components/ui/badge"
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table"
import { Play, Loader2 } from "lucide-react"
import { formatDistanceToNow, format } from "date-fns"
import { Progress } from "@/components/ui/progress"

export const Route = createFileRoute("/_layout/projects/$projectId/audits/")({
  component: AuditsPage,
})

const STATUS_COLORS: Record<string, string> = {
  queued: "secondary",
  crawling: "default",
  rendering: "default",
  analyzing: "default",
  diffing: "default",
  completed: "success",
  failed: "destructive",
}

function AuditsPage() {
  const { projectId } = Route.useParams()
  const queryClient = useQueryClient()

  const { data, isLoading } = useQuery({
    queryKey: ["projects", projectId, "audits"],
    queryFn: () => AuditsService.listAudits({ projectId }),
    refetchInterval: 5000, // Poll for status updates
  })

  const startMutation = useMutation({
    mutationFn: () => AuditsService.startAudit({ projectId }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["projects", projectId, "audits"] })
    },
  })

  if (isLoading) {
    return <div>Loading...</div>
  }

  return (
    <div className="container mx-auto py-6">
      <div className="flex items-center justify-between mb-6">
        <div>
          <h1 className="text-2xl font-bold">Site Audits</h1>
          <p className="text-muted-foreground">
            Crawl your site and detect SEO issues
          </p>
        </div>
        <Button onClick={() => startMutation.mutate()} disabled={startMutation.isPending}>
          {startMutation.isPending ? (
            <Loader2 className="mr-2 h-4 w-4 animate-spin" />
          ) : (
            <Play className="mr-2 h-4 w-4" />
          )}
          Run Audit
        </Button>
      </div>

      <div className="border rounded-lg">
        <Table>
          <TableHeader>
            <TableRow>
              <TableHead>Status</TableHead>
              <TableHead>Started</TableHead>
              <TableHead>Pages</TableHead>
              <TableHead>Issues</TableHead>
              <TableHead>Progress</TableHead>
            </TableRow>
          </TableHeader>
          <TableBody>
            {data?.items.map((audit) => (
              <TableRow key={audit.id}>
                <TableCell>
                  <Link to={`/projects/${projectId}/audits/${audit.id}`}>
                    <Badge variant={STATUS_COLORS[audit.status] as any}>
                      {audit.status}
                    </Badge>
                  </Link>
                </TableCell>
                <TableCell>
                  {audit.started_at
                    ? formatDistanceToNow(new Date(audit.started_at), {
                        addSuffix: true,
                      })
                    : "—"}
                </TableCell>
                <TableCell>{audit.stats?.pages_crawled || 0}</TableCell>
                <TableCell>
                  <div className="flex gap-1">
                    {audit.stats?.issues_critical > 0 && (
                      <Badge variant="destructive">
                        {audit.stats.issues_critical} critical
                      </Badge>
                    )}
                    {audit.stats?.issues_high > 0 && (
                      <Badge variant="warning">
                        {audit.stats.issues_high} high
                      </Badge>
                    )}
                  </div>
                </TableCell>
                <TableCell className="w-[200px]">
                  {audit.status !== "completed" && audit.status !== "failed" ? (
                    <div className="space-y-1">
                      <Progress value={audit.progress_pct} />
                      <p className="text-xs text-muted-foreground">
                        {audit.progress_message}
                      </p>
                    </div>
                  ) : (
                    <span className="text-sm">
                      {audit.finished_at &&
                        format(new Date(audit.finished_at), "MMM d, h:mm a")}
                    </span>
                  )}
                </TableCell>
              </TableRow>
            ))}
          </TableBody>
        </Table>
      </div>
    </div>
  )
}
```

### 6.2 Audit Detail with Issues

```tsx
// frontend/src/routes/_layout/projects/$projectId/audits/$auditId/index.tsx
import { createFileRoute, Link } from "@tanstack/react-router"
import { useQuery } from "@tanstack/react-query"
import { AuditsService } from "@/client"
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card"
import { Badge } from "@/components/ui/badge"
import { Tabs, TabsContent, TabsList, TabsTrigger } from "@/components/ui/tabs"
import { Button } from "@/components/ui/button"
import { Download, AlertTriangle, AlertCircle, Info, CheckCircle2 } from "lucide-react"

export const Route = createFileRoute(
  "/_layout/projects/$projectId/audits/$auditId/"
)({
  component: AuditDetailPage,
})

function AuditDetailPage() {
  const { projectId, auditId } = Route.useParams()

  const { data: audit, isLoading: auditLoading } = useQuery({
    queryKey: ["audits", auditId],
    queryFn: () => AuditsService.getAudit({ projectId, auditId }),
  })

  const { data: issues, isLoading: issuesLoading } = useQuery({
    queryKey: ["audits", auditId, "issues"],
    queryFn: () => AuditsService.getAuditIssues({ projectId, auditId }),
  })

  if (auditLoading || issuesLoading) {
    return <div>Loading...</div>
  }

  return (
    <div className="container mx-auto py-6">
      {/* Summary Cards */}
      <div className="grid grid-cols-4 gap-4 mb-6">
        <SummaryCard
          title="Critical"
          count={issues?.by_severity?.critical || 0}
          icon={<AlertTriangle className="h-4 w-4 text-red-500" />}
          variant="destructive"
        />
        <SummaryCard
          title="High"
          count={issues?.by_severity?.high || 0}
          icon={<AlertCircle className="h-4 w-4 text-orange-500" />}
          variant="warning"
        />
        <SummaryCard
          title="Medium"
          count={issues?.by_severity?.medium || 0}
          icon={<Info className="h-4 w-4 text-yellow-500" />}
          variant="default"
        />
        <SummaryCard
          title="Low"
          count={issues?.by_severity?.low || 0}
          icon={<CheckCircle2 className="h-4 w-4 text-blue-500" />}
          variant="secondary"
        />
      </div>

      {/* Tabs */}
      <Tabs defaultValue="issues">
        <div className="flex items-center justify-between mb-4">
          <TabsList>
            <TabsTrigger value="issues">Issues ({issues?.total || 0})</TabsTrigger>
            <TabsTrigger value="pages">
              Pages ({audit?.stats?.pages_crawled || 0})
            </TabsTrigger>
          </TabsList>
          <Button variant="outline" asChild>
            <a
              href={`/api/v1/projects/${projectId}/audits/${auditId}/export/issues.csv`}
              download
            >
              <Download className="mr-2 h-4 w-4" />
              Export CSV
            </a>
          </Button>
        </div>

        <TabsContent value="issues">
          <IssuesTable projectId={projectId} auditId={auditId} />
        </TabsContent>

        <TabsContent value="pages">
          <PagesTable projectId={projectId} auditId={auditId} />
        </TabsContent>
      </Tabs>
    </div>
  )
}

function SummaryCard({
  title,
  count,
  icon,
  variant,
}: {
  title: string
  count: number
  icon: React.ReactNode
  variant: string
}) {
  return (
    <Card>
      <CardHeader className="flex flex-row items-center justify-between pb-2">
        <CardTitle className="text-sm font-medium">{title}</CardTitle>
        {icon}
      </CardHeader>
      <CardContent>
        <div className="text-2xl font-bold">{count}</div>
      </CardContent>
    </Card>
  )
}

// IssuesTable and PagesTable components would be similar data tables
// with filtering, sorting, and pagination
```

---

## 7) Testing

### 7.1 Unit Tests for Issue Rules

```python
# backend/tests/unit/test_audit_rules.py
import pytest
from app.services.audit.analyzer import IssueAnalyzer, DetectedIssue
from app.models.audit import CrawledPage, IssueType, IssueSeverity


@pytest.fixture
def analyzer():
    return IssueAnalyzer(seed_url="https://example.com")


def test_detect_missing_title(analyzer):
    page = CrawledPage(
        audit_run_id="...",
        url="https://example.com/page",
        final_url="https://example.com/page",
        status_code=200,
        title=None,
        meta_description="Some description",
        h1_count=1,
        word_count=500,
    )

    issues = list(analyzer.analyze_page(page))
    assert any(i.issue_type == IssueType.MISSING_TITLE for i in issues)


def test_detect_5xx_error(analyzer):
    page = CrawledPage(
        audit_run_id="...",
        url="https://example.com/error",
        final_url="https://example.com/error",
        status_code=500,
    )

    issues = list(analyzer.analyze_page(page))
    assert len(issues) == 1
    assert issues[0].issue_type == IssueType.SERVER_ERROR_5XX
    assert issues[0].severity == IssueSeverity.CRITICAL


def test_detect_thin_content(analyzer):
    page = CrawledPage(
        audit_run_id="...",
        url="https://example.com/thin",
        final_url="https://example.com/thin",
        status_code=200,
        title="Valid Title",
        meta_description="Valid description that is long enough",
        h1_count=1,
        word_count=50,  # Below 300 threshold
    )

    issues = list(analyzer.analyze_page(page))
    assert any(i.issue_type == IssueType.THIN_CONTENT for i in issues)


def test_detect_redirect_chain(analyzer):
    page = CrawledPage(
        audit_run_id="...",
        url="https://example.com/a",
        final_url="https://example.com/d",
        status_code=200,
        redirect_chain=[
            "https://example.com/a",
            "https://example.com/b",
            "https://example.com/c",
            "https://example.com/d",
        ],
        title="Title",
        h1_count=1,
        word_count=500,
    )

    issues = list(analyzer.analyze_page(page))
    assert any(i.issue_type == IssueType.REDIRECT_CHAIN for i in issues)
```

---

## 8) Acceptance Criteria

From PRD Section 6.2:

- [x] Run audits on demand + on schedule
- [x] Support hybrid crawling: HTML-first with optional JS rendering
- [x] Prioritized issues with severity levels
- [x] Per-page evidence and details
- [x] Diff vs prior run (new/resolved tracking)
- [x] CSV export from filtered views
- [x] Render budgets (max rendered pages per run)

---

## 9) Implementation Checklist

```
[ ] Create backend/app/models/audit.py with all models
[ ] Create Alembic migration for audit tables
[ ] Create backend/app/services/audit/crawler.py
[ ] Create backend/app/services/audit/renderer.py
[ ] Create backend/app/services/audit/analyzer.py
[ ] Create backend/app/services/audit/differ.py
[ ] Create backend/app/tasks/audit.py with Celery tasks
[ ] Create backend/app/api/routes/audits.py
[ ] Register routes in backend/app/api/main.py
[ ] Add Playwright to backend dependencies (pyproject.toml)
[ ] Create frontend routes for audits
[ ] Generate TypeScript client
[ ] Write unit tests for issue rules
[ ] Write integration tests for crawl pipeline
[ ] Write E2E tests for audit UI flow
```

---

## 10) Dependencies

**Before this module:**
- Projects module (parent entity)
- Celery infrastructure (background tasks)

**Python packages to add:**
```toml
# backend/pyproject.toml
httpx = "^0.27.0"
beautifulsoup4 = "^4.12.0"
playwright = "^1.42.0"
lxml = "^5.1.0"
```

**After this module:**
- GSC module can use audit pages for matching
- SERP module can track pages discovered in audits
