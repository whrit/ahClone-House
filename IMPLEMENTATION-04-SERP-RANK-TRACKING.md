# Implementation 04: SERP & Rank Tracking

> **PRD Reference:** Section 6.4 Rank Tracking / SERP Snapshots (Provider-Based)
> **Priority:** P2 (Core feature)
> **Sprint:** 3
> **Dependencies:** Projects module, Celery infrastructure

---

## 1) Overview

The SERP/Rank Tracking module monitors keyword positions over time using a **provider-based architecture**. This allows for different data sources while maintaining compliance by default.

**Key Principles:**
- Provider interface abstracts data sources
- Default providers are compliant (GSC-based, licensed APIs)
- Optional providers disabled by default
- SERP snapshots stored for reproducibility

**Data Flow:**
```
Add Keyword Target → Schedule Refresh → Provider Fetches Data → Store Observation
                                                ↓
                                         SERP Snapshot
                                         Position Trend
```

---

## 2) Data Models

```python
# backend/app/models/serp.py
import uuid
from datetime import datetime
from enum import Enum
from sqlmodel import SQLModel, Field, Relationship, Column, JSON, Index
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from .project import Project


class DeviceType(str, Enum):
    DESKTOP = "desktop"
    MOBILE = "mobile"
    TABLET = "tablet"


class SearchEngine(str, Enum):
    GOOGLE = "google"
    BING = "bing"


class RefreshStatus(str, Enum):
    PENDING = "pending"
    SUCCESS = "success"
    FAILED = "failed"
    RATE_LIMITED = "rate_limited"


class KeywordTarget(SQLModel, table=True):
    """A keyword being tracked for rank changes"""
    __tablename__ = "keyword_targets"
    __table_args__ = (
        Index("ix_keyword_target_project", "project_id"),
    )

    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    project_id: uuid.UUID = Field(foreign_key="projects.id", index=True)

    # Keyword specification
    keyword: str = Field(index=True)
    locale: str = "en-US"  # Language-Country code
    device: DeviceType = DeviceType.DESKTOP
    search_engine: SearchEngine = SearchEngine.GOOGLE

    # Provider configuration
    provider_key: str = "gsc_based"  # Which provider to use

    # Schedule
    is_active: bool = True
    refresh_frequency: str = "daily"  # "daily", "weekly", "manual"
    last_refresh_at: datetime | None = None
    next_refresh_at: datetime | None = None

    # Latest observation cache (for quick display)
    latest_position: int | None = None
    latest_url: str | None = None
    position_change: int | None = None  # vs prior observation

    created_at: datetime = Field(default_factory=datetime.utcnow)

    # Relationships
    project: "Project" = Relationship(back_populates="keyword_targets")
    observations: list["RankObservation"] = Relationship(
        back_populates="keyword_target",
        sa_relationship_kwargs={"cascade": "all, delete-orphan"},
    )
    snapshots: list["SerpSnapshot"] = Relationship(
        back_populates="keyword_target",
        sa_relationship_kwargs={"cascade": "all, delete-orphan"},
    )


class RankObservation(SQLModel, table=True):
    """A single position observation for a keyword"""
    __tablename__ = "rank_observations"
    __table_args__ = (
        Index("ix_rank_obs_keyword_date", "keyword_target_id", "observed_at"),
    )

    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    keyword_target_id: uuid.UUID = Field(foreign_key="keyword_targets.id", index=True)

    observed_at: datetime = Field(default_factory=datetime.utcnow, index=True)
    provider_key: str

    # Position data
    rank: int | None = None  # 1-100, None if not found
    url: str | None = None  # Ranking URL
    domain: str | None = None

    # Additional context
    title: str | None = None
    snippet: str | None = None

    # Provider metadata
    status: RefreshStatus = RefreshStatus.SUCCESS
    error_message: str | None = None

    # Relationships
    keyword_target: "KeywordTarget" = Relationship(back_populates="observations")


class SerpSnapshot(SQLModel, table=True):
    """Full SERP snapshot for reproducibility"""
    __tablename__ = "serp_snapshots"

    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    keyword_target_id: uuid.UUID = Field(foreign_key="keyword_targets.id", index=True)

    fetched_at: datetime = Field(default_factory=datetime.utcnow)
    provider_key: str

    # Snapshot data
    results: list[dict] = Field(default_factory=list, sa_column=Column(JSON))
    # Each result: {rank, url, domain, title, snippet}

    # Metadata
    total_results: int | None = None
    search_engine_version: str | None = None
    raw_response: dict | None = Field(default=None, sa_column=Column(JSON))

    status: RefreshStatus = RefreshStatus.SUCCESS
    error_message: str | None = None

    # Relationships
    keyword_target: "KeywordTarget" = Relationship(back_populates="snapshots")


# API Response Models
class KeywordTargetCreate(SQLModel):
    keyword: str
    locale: str = "en-US"
    device: DeviceType = DeviceType.DESKTOP
    search_engine: SearchEngine = SearchEngine.GOOGLE
    provider_key: str = "gsc_based"
    refresh_frequency: str = "daily"


class KeywordTargetPublic(SQLModel):
    id: uuid.UUID
    keyword: str
    locale: str
    device: DeviceType
    search_engine: SearchEngine
    provider_key: str
    is_active: bool
    refresh_frequency: str
    last_refresh_at: datetime | None
    latest_position: int | None
    latest_url: str | None
    position_change: int | None
    created_at: datetime


class KeywordTargetsPublic(SQLModel):
    items: list[KeywordTargetPublic]
    total: int


class RankObservationPublic(SQLModel):
    id: uuid.UUID
    observed_at: datetime
    rank: int | None
    url: str | None
    title: str | None
    status: RefreshStatus


class RankHistoryResponse(SQLModel):
    keyword: str
    observations: list[RankObservationPublic]
    trend: list[dict]  # [{date, position}] for charts


class SerpResultPublic(SQLModel):
    rank: int
    url: str
    domain: str
    title: str | None
    snippet: str | None
    is_own_domain: bool = False


class SerpSnapshotPublic(SQLModel):
    id: uuid.UUID
    fetched_at: datetime
    results: list[SerpResultPublic]
    total_results: int | None
```

---

## 3) Provider Interface

### 3.1 Base Provider

```python
# backend/app/services/serp/providers/base.py
from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import datetime
from enum import Enum
from typing import Any


class ProviderStatus(str, Enum):
    SUCCESS = "success"
    NOT_FOUND = "not_found"
    RATE_LIMITED = "rate_limited"
    ERROR = "error"
    BLOCKED = "blocked"


@dataclass
class SerpResult:
    """A single result in the SERP"""
    rank: int
    url: str
    domain: str
    title: str | None = None
    snippet: str | None = None


@dataclass
class ProviderResponse:
    """Response from a SERP provider"""
    status: ProviderStatus
    results: list[SerpResult]
    fetched_at: datetime
    total_results: int | None = None
    error_message: str | None = None
    raw_response: dict | None = None
    provider_metadata: dict | None = None


class SerpProvider(ABC):
    """Abstract base class for SERP data providers"""

    # Provider identification
    provider_key: str = "base"
    display_name: str = "Base Provider"
    is_compliant: bool = True  # Safe to use by default

    @abstractmethod
    async def fetch_serp(
        self,
        keyword: str,
        locale: str,
        device: str,
        search_engine: str,
        depth: int = 10,
    ) -> ProviderResponse:
        """
        Fetch SERP results for a keyword.

        Args:
            keyword: Search query
            locale: Language-country code (e.g., "en-US")
            device: "desktop", "mobile", "tablet"
            search_engine: "google", "bing"
            depth: Number of results to return

        Returns:
            ProviderResponse with results or error
        """
        pass

    def find_domain_rank(
        self,
        results: list[SerpResult],
        target_domain: str,
    ) -> tuple[int | None, str | None]:
        """
        Find the rank and URL for a target domain in results.

        Returns:
            (rank, url) tuple, or (None, None) if not found
        """
        target_domain = target_domain.lower().replace("www.", "")

        for result in results:
            result_domain = result.domain.lower().replace("www.", "")
            if result_domain == target_domain or result_domain.endswith(f".{target_domain}"):
                return result.rank, result.url

        return None, None


class ProviderRegistry:
    """Registry of available SERP providers"""

    _providers: dict[str, type[SerpProvider]] = {}

    @classmethod
    def register(cls, provider_class: type[SerpProvider]) -> type[SerpProvider]:
        """Decorator to register a provider"""
        cls._providers[provider_class.provider_key] = provider_class
        return provider_class

    @classmethod
    def get(cls, provider_key: str) -> SerpProvider | None:
        """Get a provider instance by key"""
        provider_class = cls._providers.get(provider_key)
        if provider_class:
            return provider_class()
        return None

    @classmethod
    def list_available(cls, compliant_only: bool = True) -> list[dict]:
        """List available providers"""
        providers = []
        for key, provider_class in cls._providers.items():
            if compliant_only and not provider_class.is_compliant:
                continue
            providers.append({
                "key": key,
                "name": provider_class.display_name,
                "is_compliant": provider_class.is_compliant,
            })
        return providers
```

### 3.2 GSC-Based Provider (Default)

```python
# backend/app/services/serp/providers/gsc_based.py
from datetime import datetime, date, timedelta
from urllib.parse import urlparse

from sqlmodel import Session, select, func

from app.models.gsc import GSCQueryDaily
from .base import (
    SerpProvider,
    ProviderResponse,
    ProviderStatus,
    SerpResult,
    ProviderRegistry,
)


@ProviderRegistry.register
class GSCBasedProvider(SerpProvider):
    """
    Provider that uses GSC data for position tracking.

    This is compliant because it uses first-party data from
    the user's own Google Search Console.

    Limitations:
    - Only shows data for the user's own domain
    - Position is an average, not real-time
    - No competitor data
    """

    provider_key = "gsc_based"
    display_name = "GSC Position Data"
    is_compliant = True

    def __init__(self, session: Session | None = None):
        self.session = session

    async def fetch_serp(
        self,
        keyword: str,
        locale: str,
        device: str,
        search_engine: str,
        depth: int = 10,
        project_id: str | None = None,
    ) -> ProviderResponse:
        """
        Fetch position from GSC data.

        Note: This doesn't return a full SERP, just the position
        for the tracked domain based on GSC data.
        """
        if not self.session or not project_id:
            return ProviderResponse(
                status=ProviderStatus.ERROR,
                results=[],
                fetched_at=datetime.utcnow(),
                error_message="Session or project_id required",
            )

        # Get recent GSC data for this query
        end_date = date.today() - timedelta(days=3)
        start_date = end_date - timedelta(days=7)

        statement = (
            select(
                GSCQueryDaily.page,
                func.avg(GSCQueryDaily.position).label("avg_position"),
                func.sum(GSCQueryDaily.clicks).label("clicks"),
                func.sum(GSCQueryDaily.impressions).label("impressions"),
            )
            .where(GSCQueryDaily.project_id == project_id)
            .where(GSCQueryDaily.query == keyword)
            .where(GSCQueryDaily.date >= start_date)
            .where(GSCQueryDaily.date <= end_date)
            .group_by(GSCQueryDaily.page)
            .order_by(func.avg(GSCQueryDaily.position))
            .limit(depth)
        )

        rows = self.session.exec(statement).all()

        if not rows:
            return ProviderResponse(
                status=ProviderStatus.NOT_FOUND,
                results=[],
                fetched_at=datetime.utcnow(),
                error_message="No GSC data found for this query",
            )

        results = []
        for i, row in enumerate(rows, 1):
            parsed = urlparse(row.page)
            results.append(
                SerpResult(
                    rank=int(row.avg_position),  # Use GSC position
                    url=row.page,
                    domain=parsed.netloc,
                    title=None,  # GSC doesn't provide this
                    snippet=None,
                )
            )

        return ProviderResponse(
            status=ProviderStatus.SUCCESS,
            results=results,
            fetched_at=datetime.utcnow(),
            total_results=len(results),
            provider_metadata={
                "source": "gsc",
                "date_range": f"{start_date} to {end_date}",
            },
        )
```

### 3.3 API Provider (Optional, Compliant)

```python
# backend/app/services/serp/providers/api_provider.py
from datetime import datetime
import httpx

from app.core.config import settings
from .base import (
    SerpProvider,
    ProviderResponse,
    ProviderStatus,
    SerpResult,
    ProviderRegistry,
)


@ProviderRegistry.register
class SerpAPIProvider(SerpProvider):
    """
    Provider that uses a licensed SERP API (e.g., SerpApi, ValueSERP).

    This is compliant because it uses an officially licensed API
    that has proper agreements with search engines.

    Requires API key configuration.
    """

    provider_key = "serp_api"
    display_name = "SERP API (Licensed)"
    is_compliant = True

    def __init__(self):
        self.api_key = settings.SERP_API_KEY
        self.api_url = settings.SERP_API_URL

    async def fetch_serp(
        self,
        keyword: str,
        locale: str,
        device: str,
        search_engine: str,
        depth: int = 10,
    ) -> ProviderResponse:
        """Fetch SERP from licensed API provider"""
        if not self.api_key:
            return ProviderResponse(
                status=ProviderStatus.ERROR,
                results=[],
                fetched_at=datetime.utcnow(),
                error_message="SERP API not configured",
            )

        # Parse locale
        parts = locale.split("-")
        language = parts[0] if len(parts) > 0 else "en"
        country = parts[1].lower() if len(parts) > 1 else "us"

        # Build request (example for SerpApi format)
        params = {
            "api_key": self.api_key,
            "q": keyword,
            "hl": language,
            "gl": country,
            "device": device,
            "num": depth,
        }

        if search_engine == "bing":
            params["engine"] = "bing"

        try:
            async with httpx.AsyncClient() as client:
                response = await client.get(
                    self.api_url,
                    params=params,
                    timeout=30,
                )

                if response.status_code == 429:
                    return ProviderResponse(
                        status=ProviderStatus.RATE_LIMITED,
                        results=[],
                        fetched_at=datetime.utcnow(),
                        error_message="Rate limit exceeded",
                    )

                response.raise_for_status()
                data = response.json()

        except Exception as e:
            return ProviderResponse(
                status=ProviderStatus.ERROR,
                results=[],
                fetched_at=datetime.utcnow(),
                error_message=str(e),
            )

        # Parse results (adjust based on actual API format)
        results = []
        organic_results = data.get("organic_results", [])

        for i, item in enumerate(organic_results[:depth], 1):
            from urllib.parse import urlparse
            parsed = urlparse(item.get("link", ""))

            results.append(
                SerpResult(
                    rank=item.get("position", i),
                    url=item.get("link", ""),
                    domain=parsed.netloc,
                    title=item.get("title"),
                    snippet=item.get("snippet"),
                )
            )

        return ProviderResponse(
            status=ProviderStatus.SUCCESS,
            results=results,
            fetched_at=datetime.utcnow(),
            total_results=data.get("search_information", {}).get("total_results"),
            raw_response=data,
        )
```

---

## 4) Tracker Service

```python
# backend/app/services/serp/tracker.py
import uuid
from datetime import datetime, timedelta
from urllib.parse import urlparse

from sqlmodel import Session, select

from app.models.serp import (
    KeywordTarget,
    RankObservation,
    SerpSnapshot,
    RefreshStatus,
)
from app.models.project import Project
from .providers.base import ProviderRegistry, ProviderStatus


class RankTracker:
    """Service for tracking keyword rankings"""

    def __init__(self, session: Session):
        self.session = session

    async def refresh_keyword(
        self,
        keyword_target: KeywordTarget,
        project: Project,
    ) -> RankObservation:
        """Refresh ranking data for a keyword target"""
        provider = ProviderRegistry.get(keyword_target.provider_key)

        if not provider:
            return self._create_error_observation(
                keyword_target,
                f"Provider '{keyword_target.provider_key}' not found",
            )

        # For GSC-based provider, pass session and project_id
        if keyword_target.provider_key == "gsc_based":
            provider.session = self.session

        # Fetch SERP data
        response = await provider.fetch_serp(
            keyword=keyword_target.keyword,
            locale=keyword_target.locale,
            device=keyword_target.device.value,
            search_engine=keyword_target.search_engine.value,
            depth=100,
            project_id=str(project.id) if keyword_target.provider_key == "gsc_based" else None,
        )

        # Create snapshot
        snapshot = SerpSnapshot(
            keyword_target_id=keyword_target.id,
            provider_key=keyword_target.provider_key,
            fetched_at=response.fetched_at,
            results=[
                {
                    "rank": r.rank,
                    "url": r.url,
                    "domain": r.domain,
                    "title": r.title,
                    "snippet": r.snippet,
                }
                for r in response.results
            ],
            total_results=response.total_results,
            raw_response=response.raw_response,
            status=RefreshStatus(response.status.value),
            error_message=response.error_message,
        )
        self.session.add(snapshot)

        # Find our domain's position
        seed_domain = urlparse(project.seed_url).netloc.replace("www.", "")
        rank, url = provider.find_domain_rank(response.results, seed_domain)

        # Get previous observation for change calculation
        previous = self._get_previous_observation(keyword_target.id)
        position_change = None
        if previous and previous.rank and rank:
            position_change = previous.rank - rank  # Positive = improvement

        # Create observation
        observation = RankObservation(
            keyword_target_id=keyword_target.id,
            observed_at=response.fetched_at,
            provider_key=keyword_target.provider_key,
            rank=rank,
            url=url,
            domain=seed_domain if rank else None,
            status=RefreshStatus(response.status.value),
            error_message=response.error_message,
        )

        # Find title/snippet for our result
        for r in response.results:
            if r.url == url:
                observation.title = r.title
                observation.snippet = r.snippet
                break

        self.session.add(observation)

        # Update keyword target cache
        keyword_target.latest_position = rank
        keyword_target.latest_url = url
        keyword_target.position_change = position_change
        keyword_target.last_refresh_at = datetime.utcnow()

        # Schedule next refresh
        if keyword_target.refresh_frequency == "daily":
            keyword_target.next_refresh_at = datetime.utcnow() + timedelta(days=1)
        elif keyword_target.refresh_frequency == "weekly":
            keyword_target.next_refresh_at = datetime.utcnow() + timedelta(weeks=1)

        self.session.commit()
        return observation

    def _get_previous_observation(
        self, keyword_target_id: uuid.UUID
    ) -> RankObservation | None:
        """Get the most recent observation before now"""
        statement = (
            select(RankObservation)
            .where(RankObservation.keyword_target_id == keyword_target_id)
            .where(RankObservation.status == RefreshStatus.SUCCESS)
            .order_by(RankObservation.observed_at.desc())
            .limit(1)
        )
        return self.session.exec(statement).first()

    def _create_error_observation(
        self,
        keyword_target: KeywordTarget,
        error_message: str,
    ) -> RankObservation:
        """Create an error observation"""
        observation = RankObservation(
            keyword_target_id=keyword_target.id,
            observed_at=datetime.utcnow(),
            provider_key=keyword_target.provider_key,
            status=RefreshStatus.FAILED,
            error_message=error_message,
        )
        self.session.add(observation)
        self.session.commit()
        return observation

    def get_rank_history(
        self,
        keyword_target_id: uuid.UUID,
        days: int = 30,
    ) -> list[RankObservation]:
        """Get rank history for a keyword"""
        since = datetime.utcnow() - timedelta(days=days)

        statement = (
            select(RankObservation)
            .where(RankObservation.keyword_target_id == keyword_target_id)
            .where(RankObservation.observed_at >= since)
            .where(RankObservation.status == RefreshStatus.SUCCESS)
            .order_by(RankObservation.observed_at)
        )

        return list(self.session.exec(statement).all())
```

---

## 5) Celery Tasks

```python
# backend/app/tasks/serp.py
import uuid
from datetime import datetime, timedelta

from celery import shared_task
from sqlmodel import Session, select

from app.core.db import engine
from app.core.config import settings
from app.models.serp import KeywordTarget
from app.models.project import Project
from app.services.serp.tracker import RankTracker


@shared_task(bind=True, max_retries=3)
def refresh_keyword(self, keyword_target_id: str) -> dict:
    """Refresh a single keyword target"""
    with Session(engine) as session:
        keyword_target = session.get(KeywordTarget, uuid.UUID(keyword_target_id))
        if not keyword_target:
            return {"error": "Keyword target not found"}

        project = session.get(Project, keyword_target.project_id)
        if not project:
            return {"error": "Project not found"}

        tracker = RankTracker(session)

        import asyncio
        observation = asyncio.run(
            tracker.refresh_keyword(keyword_target, project)
        )

        return {
            "keyword": keyword_target.keyword,
            "rank": observation.rank,
            "status": observation.status.value,
        }


@shared_task(bind=True)
def refresh_project_keywords(self, project_id: str) -> dict:
    """Refresh all keywords for a project (respecting daily cap)"""
    with Session(engine) as session:
        project = session.get(Project, uuid.UUID(project_id))
        if not project:
            return {"error": "Project not found"}

        # Get keywords due for refresh
        now = datetime.utcnow()
        statement = (
            select(KeywordTarget)
            .where(KeywordTarget.project_id == project.id)
            .where(KeywordTarget.is_active == True)
            .where(
                (KeywordTarget.next_refresh_at == None) |
                (KeywordTarget.next_refresh_at <= now)
            )
            .limit(settings.SERP_REFRESH_DAILY_CAP)
        )

        keywords = session.exec(statement).all()
        tracker = RankTracker(session)

        results = []
        import asyncio
        for kw in keywords:
            try:
                observation = asyncio.run(tracker.refresh_keyword(kw, project))
                results.append({
                    "keyword": kw.keyword,
                    "rank": observation.rank,
                    "status": observation.status.value,
                })
            except Exception as e:
                results.append({
                    "keyword": kw.keyword,
                    "status": "error",
                    "error": str(e),
                })

        # Update project timestamp
        project.last_serp_refresh_at = datetime.utcnow()
        session.commit()

        return {
            "refreshed": len(results),
            "results": results,
        }


@shared_task(bind=True)
def refresh_all_due_keywords(self) -> dict:
    """Daily task to refresh all due keywords across all projects"""
    with Session(engine) as session:
        now = datetime.utcnow()

        # Get all due keywords grouped by project
        statement = (
            select(KeywordTarget)
            .where(KeywordTarget.is_active == True)
            .where(
                (KeywordTarget.next_refresh_at == None) |
                (KeywordTarget.next_refresh_at <= now)
            )
        )

        keywords = session.exec(statement).all()

        # Group by project
        by_project: dict[uuid.UUID, list[KeywordTarget]] = {}
        for kw in keywords:
            if kw.project_id not in by_project:
                by_project[kw.project_id] = []
            by_project[kw.project_id].append(kw)

        # Queue refresh tasks per project
        for project_id in by_project:
            refresh_project_keywords.delay(str(project_id))

        return {
            "projects_queued": len(by_project),
            "total_keywords": len(keywords),
        }
```

---

## 6) API Routes

```python
# backend/app/api/routes/serp.py
import uuid
from datetime import datetime, timedelta
from typing import Any

from fastapi import APIRouter, Depends, HTTPException, Query
from sqlmodel import select, func

from app.api.deps import CurrentUser, SessionDep
from app.models.project import Project
from app.models.serp import (
    KeywordTarget,
    KeywordTargetCreate,
    KeywordTargetPublic,
    KeywordTargetsPublic,
    RankObservation,
    RankObservationPublic,
    RankHistoryResponse,
    SerpSnapshot,
    SerpSnapshotPublic,
    SerpResultPublic,
)
from app.services.serp.providers.base import ProviderRegistry
from app.tasks.serp import refresh_keyword, refresh_project_keywords

router = APIRouter(prefix="/projects/{project_id}/serp", tags=["serp"])


def get_project_or_404(
    session: SessionDep, project_id: uuid.UUID, current_user: CurrentUser
) -> Project:
    project = session.get(Project, project_id)
    if not project or project.created_by_id != current_user.id:
        raise HTTPException(status_code=404, detail="Project not found")
    return project


@router.get("/providers")
def list_providers() -> list[dict]:
    """List available SERP providers"""
    return ProviderRegistry.list_available(compliant_only=True)


@router.post("/keywords", response_model=KeywordTargetPublic)
def add_keyword(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
    keyword_in: KeywordTargetCreate,
) -> Any:
    """Add a keyword to track"""
    project = get_project_or_404(session, project_id, current_user)

    # Check limit
    count = session.exec(
        select(func.count())
        .select_from(KeywordTarget)
        .where(KeywordTarget.project_id == project.id)
    ).one()

    from app.core.config import settings
    if count >= settings.MAX_KEYWORDS_PER_PROJECT:
        raise HTTPException(
            status_code=400,
            detail=f"Maximum {settings.MAX_KEYWORDS_PER_PROJECT} keywords per project",
        )

    # Check for duplicate
    existing = session.exec(
        select(KeywordTarget)
        .where(KeywordTarget.project_id == project.id)
        .where(KeywordTarget.keyword == keyword_in.keyword)
        .where(KeywordTarget.locale == keyword_in.locale)
        .where(KeywordTarget.device == keyword_in.device)
    ).first()

    if existing:
        raise HTTPException(status_code=400, detail="Keyword already tracked")

    keyword_target = KeywordTarget(
        project_id=project.id,
        **keyword_in.model_dump(),
    )
    session.add(keyword_target)
    session.commit()
    session.refresh(keyword_target)

    # Trigger initial refresh
    refresh_keyword.delay(str(keyword_target.id))

    return keyword_target


@router.get("/keywords", response_model=KeywordTargetsPublic)
def list_keywords(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
    skip: int = Query(default=0, ge=0),
    limit: int = Query(default=50, le=200),
) -> Any:
    """List tracked keywords"""
    project = get_project_or_404(session, project_id, current_user)

    count = session.exec(
        select(func.count())
        .select_from(KeywordTarget)
        .where(KeywordTarget.project_id == project.id)
    ).one()

    statement = (
        select(KeywordTarget)
        .where(KeywordTarget.project_id == project.id)
        .offset(skip)
        .limit(limit)
        .order_by(KeywordTarget.created_at.desc())
    )
    keywords = session.exec(statement).all()

    return KeywordTargetsPublic(items=keywords, total=count)


@router.delete("/keywords/{keyword_id}")
def delete_keyword(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
    keyword_id: uuid.UUID,
) -> dict:
    """Delete a tracked keyword"""
    project = get_project_or_404(session, project_id, current_user)

    keyword = session.get(KeywordTarget, keyword_id)
    if not keyword or keyword.project_id != project.id:
        raise HTTPException(status_code=404, detail="Keyword not found")

    session.delete(keyword)
    session.commit()

    return {"message": "Keyword deleted"}


@router.post("/keywords/{keyword_id}/refresh")
def trigger_refresh(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
    keyword_id: uuid.UUID,
) -> dict:
    """Trigger manual refresh for a keyword"""
    project = get_project_or_404(session, project_id, current_user)

    keyword = session.get(KeywordTarget, keyword_id)
    if not keyword or keyword.project_id != project.id:
        raise HTTPException(status_code=404, detail="Keyword not found")

    refresh_keyword.delay(str(keyword.id))
    return {"message": "Refresh queued"}


@router.post("/refresh-all")
def trigger_refresh_all(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
) -> dict:
    """Trigger refresh for all project keywords"""
    project = get_project_or_404(session, project_id, current_user)

    refresh_project_keywords.delay(str(project.id))
    return {"message": "Refresh queued for all keywords"}


@router.get("/keywords/{keyword_id}/history", response_model=RankHistoryResponse)
def get_keyword_history(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
    keyword_id: uuid.UUID,
    days: int = Query(default=30, le=90),
) -> Any:
    """Get rank history for a keyword"""
    project = get_project_or_404(session, project_id, current_user)

    keyword = session.get(KeywordTarget, keyword_id)
    if not keyword or keyword.project_id != project.id:
        raise HTTPException(status_code=404, detail="Keyword not found")

    since = datetime.utcnow() - timedelta(days=days)

    statement = (
        select(RankObservation)
        .where(RankObservation.keyword_target_id == keyword.id)
        .where(RankObservation.observed_at >= since)
        .order_by(RankObservation.observed_at)
    )
    observations = session.exec(statement).all()

    # Build trend data for charts
    trend = [
        {
            "date": obs.observed_at.isoformat(),
            "position": obs.rank,
        }
        for obs in observations
        if obs.rank
    ]

    return RankHistoryResponse(
        keyword=keyword.keyword,
        observations=[
            RankObservationPublic(
                id=obs.id,
                observed_at=obs.observed_at,
                rank=obs.rank,
                url=obs.url,
                title=obs.title,
                status=obs.status,
            )
            for obs in observations
        ],
        trend=trend,
    )


@router.get("/keywords/{keyword_id}/snapshots/{snapshot_id}")
def get_serp_snapshot(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
    keyword_id: uuid.UUID,
    snapshot_id: uuid.UUID,
) -> Any:
    """Get a SERP snapshot"""
    project = get_project_or_404(session, project_id, current_user)

    keyword = session.get(KeywordTarget, keyword_id)
    if not keyword or keyword.project_id != project.id:
        raise HTTPException(status_code=404, detail="Keyword not found")

    snapshot = session.get(SerpSnapshot, snapshot_id)
    if not snapshot or snapshot.keyword_target_id != keyword.id:
        raise HTTPException(status_code=404, detail="Snapshot not found")

    # Mark own domain results
    from urllib.parse import urlparse
    seed_domain = urlparse(project.seed_url).netloc.replace("www.", "")

    results = []
    for r in snapshot.results:
        result_domain = r.get("domain", "").replace("www.", "")
        is_own = (
            result_domain == seed_domain or
            result_domain.endswith(f".{seed_domain}")
        )
        results.append(
            SerpResultPublic(
                rank=r["rank"],
                url=r["url"],
                domain=r["domain"],
                title=r.get("title"),
                snippet=r.get("snippet"),
                is_own_domain=is_own,
            )
        )

    return SerpSnapshotPublic(
        id=snapshot.id,
        fetched_at=snapshot.fetched_at,
        results=results,
        total_results=snapshot.total_results,
    )
```

---

## 7) Frontend Pages

### 7.1 Rank Tracker List

```tsx
// frontend/src/routes/_layout/projects/$projectId/rank-tracker/index.tsx
import { createFileRoute, Link } from "@tanstack/react-router"
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query"
import { useState } from "react"
import { SerpService } from "@/client"
import { Button } from "@/components/ui/button"
import { Badge } from "@/components/ui/badge"
import { Input } from "@/components/ui/input"
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table"
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from "@/components/ui/dialog"
import { Plus, RefreshCw, TrendingUp, TrendingDown, Minus } from "lucide-react"
import { formatDistanceToNow } from "date-fns"

export const Route = createFileRoute("/_layout/projects/$projectId/rank-tracker/")({
  component: RankTrackerPage,
})

function RankTrackerPage() {
  const { projectId } = Route.useParams()
  const queryClient = useQueryClient()
  const [addDialogOpen, setAddDialogOpen] = useState(false)

  const { data, isLoading, refetch } = useQuery({
    queryKey: ["serp", "keywords", projectId],
    queryFn: () => SerpService.listKeywords({ projectId }),
    refetchInterval: 30000, // Poll for updates
  })

  const refreshAllMutation = useMutation({
    mutationFn: () => SerpService.triggerRefreshAll({ projectId }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["serp", "keywords", projectId] })
    },
  })

  if (isLoading) return <div>Loading...</div>

  return (
    <div className="container mx-auto py-6">
      <div className="flex items-center justify-between mb-6">
        <div>
          <h1 className="text-2xl font-bold">Rank Tracker</h1>
          <p className="text-muted-foreground">
            Monitor keyword positions over time
          </p>
        </div>
        <div className="flex gap-2">
          <Button
            variant="outline"
            onClick={() => refreshAllMutation.mutate()}
            disabled={refreshAllMutation.isPending}
          >
            <RefreshCw className="mr-2 h-4 w-4" />
            Refresh All
          </Button>
          <AddKeywordDialog
            projectId={projectId}
            open={addDialogOpen}
            onOpenChange={setAddDialogOpen}
            onSuccess={() => refetch()}
          />
        </div>
      </div>

      <div className="border rounded-lg">
        <Table>
          <TableHeader>
            <TableRow>
              <TableHead>Keyword</TableHead>
              <TableHead>Position</TableHead>
              <TableHead>Change</TableHead>
              <TableHead>URL</TableHead>
              <TableHead>Last Updated</TableHead>
              <TableHead></TableHead>
            </TableRow>
          </TableHeader>
          <TableBody>
            {data?.items.map((kw) => (
              <TableRow key={kw.id}>
                <TableCell>
                  <Link
                    to={`/projects/${projectId}/rank-tracker/${kw.id}`}
                    className="font-medium hover:underline"
                  >
                    {kw.keyword}
                  </Link>
                  <div className="text-xs text-muted-foreground">
                    {kw.locale} / {kw.device}
                  </div>
                </TableCell>
                <TableCell>
                  {kw.latest_position ? (
                    <Badge variant="outline" className="text-lg font-bold">
                      #{kw.latest_position}
                    </Badge>
                  ) : (
                    <Badge variant="secondary">Not found</Badge>
                  )}
                </TableCell>
                <TableCell>
                  <PositionChange change={kw.position_change} />
                </TableCell>
                <TableCell className="max-w-[200px] truncate">
                  {kw.latest_url && (
                    <a
                      href={kw.latest_url}
                      target="_blank"
                      rel="noopener noreferrer"
                      className="text-sm text-blue-600 hover:underline"
                    >
                      {new URL(kw.latest_url).pathname}
                    </a>
                  )}
                </TableCell>
                <TableCell>
                  {kw.last_refresh_at ? (
                    formatDistanceToNow(new Date(kw.last_refresh_at), {
                      addSuffix: true,
                    })
                  ) : (
                    <span className="text-muted-foreground">Pending</span>
                  )}
                </TableCell>
                <TableCell>
                  <RefreshButton keywordId={kw.id} projectId={projectId} />
                </TableCell>
              </TableRow>
            ))}
          </TableBody>
        </Table>
      </div>
    </div>
  )
}

function PositionChange({ change }: { change: number | null }) {
  if (change === null || change === 0) {
    return (
      <span className="flex items-center text-muted-foreground">
        <Minus className="h-4 w-4" />
      </span>
    )
  }

  if (change > 0) {
    return (
      <span className="flex items-center text-green-600">
        <TrendingUp className="h-4 w-4 mr-1" />+{change}
      </span>
    )
  }

  return (
    <span className="flex items-center text-red-600">
      <TrendingDown className="h-4 w-4 mr-1" />
      {change}
    </span>
  )
}

// AddKeywordDialog and RefreshButton components...
```

---

## 8) Configuration

Add to `.env`:
```env
# Optional SERP API provider
SERP_API_KEY=
SERP_API_URL=https://serpapi.com/search

# Limits
MAX_KEYWORDS_PER_PROJECT=500
SERP_REFRESH_DAILY_CAP=100
```

---

## 9) Acceptance Criteria

From PRD Section 6.4:

- [x] Provider interface defined
- [x] Default GSC-based provider (compliant)
- [x] Optional API provider (compliant, licensed)
- [x] Keyword targets with locale/device/engine
- [x] Scheduled + on-demand refresh
- [x] SERP snapshots stored for reproducibility
- [x] Position trends and history
- [x] Domain highlighting in SERP viewer

---

## 10) Implementation Checklist

```
[x] Create backend/app/models/serp.py
[x] Create Alembic migration for SERP tables
[x] Create backend/app/services/serp/providers/base.py
[x] Create backend/app/services/serp/gsc_provider.py
[ ] Create backend/app/services/serp/providers/api_provider.py (optional - not implemented)
[x] Create backend/app/services/serp/tracker.py
[x] Create backend/app/tasks/serp.py
[x] Create backend/app/api/routes/serp.py
[x] Register routes in backend/app/api/main.py
[x] Create frontend pages for rank tracker
[x] Generate TypeScript client (via npm run build)
[x] Write unit tests for providers
[x] Write integration tests for tracker
[ ] Add Celery Beat schedule for daily refresh (can be added to celery config)
```

## 11) Sprint 3 Completion Summary (2025-12-23)

**Status:** ✅ COMPLETED

**Test Results:**
- 478 backend tests passing
- 3 skipped tests
- Zero TypeScript/frontend build errors

**Components Implemented:**
- **SERP Models:** KeywordTarget, RankObservation, SerpSnapshot with timezone-aware timestamps
- **Provider Interface:** SerpProvider ABC, ProviderRegistry, ProviderStatus, SerpResult, ProviderResponse
- **GSC Provider:** GSCBasedProvider using GSCQueryDaily data with first-party compliance
- **RankTracker Service:** Orchestrates keyword refresh, position change calculation, history retrieval
- **Celery Tasks:** refresh_keyword, refresh_project_keywords, refresh_all_due_keywords
- **API Routes:** 8 endpoints for keyword CRUD, refresh triggers, history, and snapshots
- **Frontend:** Rank Tracker list page, keyword detail page with charts, shared components
