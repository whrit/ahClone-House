# Implementation 06: PPC, Transparency & Traffic Panel

> **PRD Reference:** Section 6.6 PPC + Transparency + Traffic Panel
> **Priority:** P3 (Valuable addition)
> **Sprint:** 5
> **Dependencies:** Projects module, Celery infrastructure, Google OAuth

---

## 1) Overview

This module provides paid search intelligence and traffic analysis:

- **PPC Reporting**: Google Ads account data for owned campaigns
- **SEO + PPC Overlap**: Match paid keywords with organic performance
- **Transparency Explorer**: Competitor ad visibility (via public datasets)
- **Traffic Panel**: Multi-source traffic data aggregation

---

## 2) Data Models

```python
# backend/app/models/ads.py
import uuid
from datetime import date, datetime
from sqlmodel import SQLModel, Field, Relationship, Column, JSON, Index


class AdsAccount(SQLModel, table=True):
    """Linked Google Ads account"""
    __tablename__ = "ads_accounts"

    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    project_id: uuid.UUID = Field(foreign_key="projects.id", unique=True, index=True)
    customer_id: str  # Google Ads customer ID
    descriptive_name: str | None = None
    currency_code: str = "USD"

    linked_at: datetime = Field(default_factory=datetime.utcnow)
    last_sync_at: datetime | None = None
    sync_status: str = "pending"


class AdsCampaignDaily(SQLModel, table=True):
    """Daily campaign performance"""
    __tablename__ = "ads_campaign_daily"
    __table_args__ = (
        Index("ix_ads_campaign_project_date", "project_id", "date"),
    )

    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    project_id: uuid.UUID = Field(foreign_key="projects.id", index=True)
    date: date = Field(index=True)

    campaign_id: str
    campaign_name: str
    campaign_status: str  # ENABLED, PAUSED, REMOVED

    # Metrics
    impressions: int = 0
    clicks: int = 0
    cost_micros: int = 0  # Cost in micros (divide by 1M for actual)
    conversions: float = 0
    conversion_value: float = 0


class AdsKeywordDaily(SQLModel, table=True):
    """Daily keyword performance"""
    __tablename__ = "ads_keyword_daily"
    __table_args__ = (
        Index("ix_ads_keyword_project_date", "project_id", "date"),
    )

    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    project_id: uuid.UUID = Field(foreign_key="projects.id", index=True)
    date: date = Field(index=True)

    campaign_id: str
    ad_group_id: str
    criterion_id: str

    keyword_text: str = Field(index=True)
    match_type: str  # EXACT, PHRASE, BROAD

    # Metrics
    impressions: int = 0
    clicks: int = 0
    cost_micros: int = 0
    conversions: float = 0
    average_cpc_micros: int = 0
    quality_score: int | None = None

    # Landing page
    final_url: str | None = None


class TransparencyCreative(SQLModel, table=True):
    """Competitor ad from transparency datasets"""
    __tablename__ = "transparency_creatives"

    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    source_key: str  # "google_ads_transparency", "meta_ad_library", etc.

    advertiser_id: str = Field(index=True)
    advertiser_name: str
    creative_id: str

    # Content
    headline: str | None = None
    description: str | None = None
    image_url: str | None = None
    landing_url: str | None = None

    # Date range seen
    first_seen: date
    last_seen: date

    # Targeting
    geo_targeting: list[str] = Field(default_factory=list, sa_column=Column(JSON))

    # Metadata
    metadata_json: dict = Field(default_factory=dict, sa_column=Column(JSON))


# Traffic Models
class TrafficDaily(SQLModel, table=True):
    """Daily traffic from various sources"""
    __tablename__ = "traffic_daily"
    __table_args__ = (
        Index("ix_traffic_project_date_source", "project_id", "date", "source_key"),
    )

    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    project_id: uuid.UUID = Field(foreign_key="projects.id", index=True)
    date: date = Field(index=True)
    source_key: str  # "ga4", "csv", "gsc", "crux"

    # Common metrics (not all sources have all)
    sessions: int | None = None
    users: int | None = None
    pageviews: int | None = None
    bounce_rate: float | None = None
    avg_session_duration: float | None = None

    # GSC-specific
    clicks: int | None = None
    impressions: int | None = None

    # CrUX-specific
    lcp_p75: float | None = None  # Largest Contentful Paint
    fid_p75: float | None = None  # First Input Delay
    cls_p75: float | None = None  # Cumulative Layout Shift

    # Dimensions
    dimensions_json: dict = Field(default_factory=dict, sa_column=Column(JSON))


# API Response Models
class CampaignRow(SQLModel):
    campaign_id: str
    campaign_name: str
    impressions: int
    clicks: int
    cost: float  # In currency units
    conversions: float
    ctr: float
    cpc: float


class CampaignsResponse(SQLModel):
    items: list[CampaignRow]
    total: int
    currency: str
    period_start: date
    period_end: date


class PaidKeywordRow(SQLModel):
    keyword_text: str
    match_type: str
    impressions: int
    clicks: int
    cost: float
    conversions: float
    cpc: float
    quality_score: int | None
    final_url: str | None


class OverlapRow(SQLModel):
    keyword: str
    # Paid metrics
    paid_clicks: int
    paid_cost: float
    paid_position: float | None  # avg position in ads
    # Organic metrics
    organic_clicks: int
    organic_impressions: int
    organic_position: float
    # Analysis
    overlap_type: str  # "both", "paid_only", "organic_only"
    opportunity_score: float


class OverlapResponse(SQLModel):
    items: list[OverlapRow]
    summary: dict  # total_overlap, paid_only_count, organic_only_count


class TrafficPanelRow(SQLModel):
    date: date
    ga4_sessions: int | None
    gsc_clicks: int | None
    estimated_traffic: int | None
    lcp: float | None
    cls: float | None


class TrafficPanelResponse(SQLModel):
    items: list[TrafficPanelRow]
    sources_available: list[str]
```

---

## 3) Google Ads Client

```python
# backend/app/services/ads/google_ads.py
from datetime import date, timedelta
from typing import Any

from google.ads.googleads.client import GoogleAdsClient
from google.ads.googleads.errors import GoogleAdsException

from app.core.config import settings


class GoogleAdsService:
    """Google Ads API client wrapper"""

    def __init__(self, refresh_token: str, customer_id: str):
        self.customer_id = customer_id

        # Configure client
        credentials = {
            "developer_token": settings.GOOGLE_ADS_DEVELOPER_TOKEN,
            "client_id": settings.GOOGLE_CLIENT_ID,
            "client_secret": settings.GOOGLE_CLIENT_SECRET,
            "refresh_token": refresh_token,
            "use_proto_plus": True,
        }

        self.client = GoogleAdsClient.load_from_dict(credentials)

    def list_accessible_customers(self) -> list[dict]:
        """List all customer accounts the user has access to"""
        customer_service = self.client.get_service("CustomerService")

        try:
            response = customer_service.list_accessible_customers()
            customers = []

            for resource_name in response.resource_names:
                customer_id = resource_name.split("/")[-1]
                # Get customer details
                details = self._get_customer_details(customer_id)
                if details:
                    customers.append(details)

            return customers
        except GoogleAdsException as e:
            raise Exception(f"Failed to list customers: {e}")

    def _get_customer_details(self, customer_id: str) -> dict | None:
        """Get details for a specific customer"""
        ga_service = self.client.get_service("GoogleAdsService")

        query = """
            SELECT
                customer.id,
                customer.descriptive_name,
                customer.currency_code,
                customer.time_zone
            FROM customer
            LIMIT 1
        """

        try:
            response = ga_service.search(customer_id=customer_id, query=query)
            for row in response:
                return {
                    "customer_id": str(row.customer.id),
                    "name": row.customer.descriptive_name,
                    "currency": row.customer.currency_code,
                    "timezone": row.customer.time_zone,
                }
        except Exception:
            return None

    def get_campaign_performance(
        self,
        start_date: date,
        end_date: date,
    ) -> list[dict]:
        """Get campaign performance data"""
        ga_service = self.client.get_service("GoogleAdsService")

        query = f"""
            SELECT
                campaign.id,
                campaign.name,
                campaign.status,
                segments.date,
                metrics.impressions,
                metrics.clicks,
                metrics.cost_micros,
                metrics.conversions,
                metrics.conversions_value
            FROM campaign
            WHERE segments.date BETWEEN '{start_date}' AND '{end_date}'
            ORDER BY segments.date
        """

        results = []
        try:
            response = ga_service.search(customer_id=self.customer_id, query=query)

            for row in response:
                results.append({
                    "date": row.segments.date,
                    "campaign_id": str(row.campaign.id),
                    "campaign_name": row.campaign.name,
                    "campaign_status": row.campaign.status.name,
                    "impressions": row.metrics.impressions,
                    "clicks": row.metrics.clicks,
                    "cost_micros": row.metrics.cost_micros,
                    "conversions": row.metrics.conversions,
                    "conversion_value": row.metrics.conversions_value,
                })
        except GoogleAdsException as e:
            raise Exception(f"Campaign query failed: {e}")

        return results

    def get_keyword_performance(
        self,
        start_date: date,
        end_date: date,
    ) -> list[dict]:
        """Get keyword-level performance data"""
        ga_service = self.client.get_service("GoogleAdsService")

        query = f"""
            SELECT
                campaign.id,
                ad_group.id,
                ad_group_criterion.criterion_id,
                ad_group_criterion.keyword.text,
                ad_group_criterion.keyword.match_type,
                ad_group_criterion.quality_info.quality_score,
                ad_group_criterion.final_urls,
                segments.date,
                metrics.impressions,
                metrics.clicks,
                metrics.cost_micros,
                metrics.conversions,
                metrics.average_cpc
            FROM keyword_view
            WHERE segments.date BETWEEN '{start_date}' AND '{end_date}'
                AND ad_group_criterion.status != 'REMOVED'
            ORDER BY metrics.clicks DESC
        """

        results = []
        try:
            response = ga_service.search(customer_id=self.customer_id, query=query)

            for row in response:
                final_url = None
                if row.ad_group_criterion.final_urls:
                    final_url = row.ad_group_criterion.final_urls[0]

                results.append({
                    "date": row.segments.date,
                    "campaign_id": str(row.campaign.id),
                    "ad_group_id": str(row.ad_group.id),
                    "criterion_id": str(row.ad_group_criterion.criterion_id),
                    "keyword_text": row.ad_group_criterion.keyword.text,
                    "match_type": row.ad_group_criterion.keyword.match_type.name,
                    "quality_score": row.ad_group_criterion.quality_info.quality_score,
                    "final_url": final_url,
                    "impressions": row.metrics.impressions,
                    "clicks": row.metrics.clicks,
                    "cost_micros": row.metrics.cost_micros,
                    "conversions": row.metrics.conversions,
                    "average_cpc_micros": row.metrics.average_cpc,
                })
        except GoogleAdsException as e:
            raise Exception(f"Keyword query failed: {e}")

        return results
```

---

## 4) SEO + PPC Overlap Analysis

```python
# backend/app/services/ads/overlap.py
import uuid
from datetime import date, timedelta
from dataclasses import dataclass

from sqlmodel import Session, select, func

from app.models.ads import AdsKeywordDaily
from app.models.gsc import GSCQueryDaily


@dataclass
class OverlapResult:
    keyword: str
    paid_clicks: int
    paid_cost: float
    paid_cpc: float
    organic_clicks: int
    organic_impressions: int
    organic_position: float
    overlap_type: str  # "both", "paid_only", "organic_only"
    opportunity_score: float


class OverlapAnalyzer:
    """Analyze overlap between paid and organic keywords"""

    def __init__(self, session: Session):
        self.session = session

    def compute_overlap(
        self,
        project_id: uuid.UUID,
        period_days: int = 28,
    ) -> list[OverlapResult]:
        """Compute SEO + PPC keyword overlap"""
        end_date = date.today() - timedelta(days=1)
        start_date = end_date - timedelta(days=period_days)

        # Get paid keywords aggregated
        paid_stmt = (
            select(
                AdsKeywordDaily.keyword_text,
                func.sum(AdsKeywordDaily.clicks).label("clicks"),
                func.sum(AdsKeywordDaily.cost_micros).label("cost_micros"),
                func.sum(AdsKeywordDaily.impressions).label("impressions"),
            )
            .where(AdsKeywordDaily.project_id == project_id)
            .where(AdsKeywordDaily.date >= start_date)
            .where(AdsKeywordDaily.date <= end_date)
            .group_by(AdsKeywordDaily.keyword_text)
        )
        paid_data = {
            row.keyword_text.lower(): {
                "clicks": row.clicks,
                "cost": row.cost_micros / 1_000_000,
                "impressions": row.impressions,
            }
            for row in self.session.exec(paid_stmt)
        }

        # Get organic keywords aggregated
        organic_stmt = (
            select(
                GSCQueryDaily.query,
                func.sum(GSCQueryDaily.clicks).label("clicks"),
                func.sum(GSCQueryDaily.impressions).label("impressions"),
                func.avg(GSCQueryDaily.position).label("position"),
            )
            .where(GSCQueryDaily.project_id == project_id)
            .where(GSCQueryDaily.date >= start_date)
            .where(GSCQueryDaily.date <= end_date)
            .group_by(GSCQueryDaily.query)
        )
        organic_data = {
            row.query.lower(): {
                "clicks": row.clicks,
                "impressions": row.impressions,
                "position": row.position,
            }
            for row in self.session.exec(organic_stmt)
        }

        # Compute overlap
        all_keywords = set(paid_data.keys()) | set(organic_data.keys())
        results = []

        for keyword in all_keywords:
            paid = paid_data.get(keyword, {})
            organic = organic_data.get(keyword, {})

            paid_clicks = paid.get("clicks", 0)
            paid_cost = paid.get("cost", 0)
            organic_clicks = organic.get("clicks", 0)
            organic_impressions = organic.get("impressions", 0)
            organic_position = organic.get("position", 0)

            # Determine overlap type
            if paid_clicks > 0 and organic_clicks > 0:
                overlap_type = "both"
            elif paid_clicks > 0:
                overlap_type = "paid_only"
            else:
                overlap_type = "organic_only"

            # Calculate opportunity score
            # High score = high paid spend + low organic performance = opportunity to reduce spend
            # Or: high organic potential + no paid coverage = opportunity to add paid
            if overlap_type == "both":
                # Could potentially reduce paid spend if organic is strong
                if organic_position < 5:
                    opportunity_score = paid_cost * (1 - (organic_position / 10))
                else:
                    opportunity_score = 0
            elif overlap_type == "paid_only":
                # Opportunity to build organic presence
                opportunity_score = paid_cost * 0.5
            else:
                # Organic only - opportunity to add paid if high value
                opportunity_score = organic_impressions * 0.001 if organic_position > 5 else 0

            results.append(OverlapResult(
                keyword=keyword,
                paid_clicks=paid_clicks,
                paid_cost=paid_cost,
                paid_cpc=paid_cost / paid_clicks if paid_clicks > 0 else 0,
                organic_clicks=organic_clicks,
                organic_impressions=organic_impressions,
                organic_position=organic_position,
                overlap_type=overlap_type,
                opportunity_score=opportunity_score,
            ))

        # Sort by opportunity score
        results.sort(key=lambda x: -x.opportunity_score)

        return results
```

---

## 5) Traffic Panel Service

```python
# backend/app/services/traffic/panel.py
import uuid
from datetime import date, timedelta
from typing import Any

from sqlmodel import Session, select

from app.models.ads import TrafficDaily
from app.models.gsc import GSCQueryDaily


class TrafficPanelService:
    """Aggregate traffic from multiple sources"""

    def __init__(self, session: Session):
        self.session = session

    def get_panel_data(
        self,
        project_id: uuid.UUID,
        period_days: int = 28,
    ) -> list[dict]:
        """Get combined traffic panel data"""
        end_date = date.today() - timedelta(days=1)
        start_date = end_date - timedelta(days=period_days)

        # Get all dates in range
        dates = []
        current = start_date
        while current <= end_date:
            dates.append(current)
            current += timedelta(days=1)

        # Query each source
        ga4_data = self._get_ga4_data(project_id, start_date, end_date)
        gsc_data = self._get_gsc_clicks(project_id, start_date, end_date)
        crux_data = self._get_crux_data(project_id, start_date, end_date)

        # Combine by date
        results = []
        for d in dates:
            row = {
                "date": d,
                "ga4_sessions": ga4_data.get(d, {}).get("sessions"),
                "ga4_users": ga4_data.get(d, {}).get("users"),
                "gsc_clicks": gsc_data.get(d),
                "lcp": crux_data.get(d, {}).get("lcp_p75"),
                "cls": crux_data.get(d, {}).get("cls_p75"),
            }
            results.append(row)

        return results

    def _get_ga4_data(
        self,
        project_id: uuid.UUID,
        start_date: date,
        end_date: date,
    ) -> dict[date, dict]:
        """Get GA4 data by date"""
        stmt = (
            select(TrafficDaily)
            .where(TrafficDaily.project_id == project_id)
            .where(TrafficDaily.source_key == "ga4")
            .where(TrafficDaily.date >= start_date)
            .where(TrafficDaily.date <= end_date)
        )

        return {
            row.date: {
                "sessions": row.sessions,
                "users": row.users,
                "pageviews": row.pageviews,
            }
            for row in self.session.exec(stmt)
        }

    def _get_gsc_clicks(
        self,
        project_id: uuid.UUID,
        start_date: date,
        end_date: date,
    ) -> dict[date, int]:
        """Get GSC clicks by date"""
        from sqlmodel import func

        stmt = (
            select(
                GSCQueryDaily.date,
                func.sum(GSCQueryDaily.clicks).label("clicks"),
            )
            .where(GSCQueryDaily.project_id == project_id)
            .where(GSCQueryDaily.date >= start_date)
            .where(GSCQueryDaily.date <= end_date)
            .group_by(GSCQueryDaily.date)
        )

        return {row.date: row.clicks for row in self.session.exec(stmt)}

    def _get_crux_data(
        self,
        project_id: uuid.UUID,
        start_date: date,
        end_date: date,
    ) -> dict[date, dict]:
        """Get CrUX data by date"""
        stmt = (
            select(TrafficDaily)
            .where(TrafficDaily.project_id == project_id)
            .where(TrafficDaily.source_key == "crux")
            .where(TrafficDaily.date >= start_date)
            .where(TrafficDaily.date <= end_date)
        )

        return {
            row.date: {
                "lcp_p75": row.lcp_p75,
                "fid_p75": row.fid_p75,
                "cls_p75": row.cls_p75,
            }
            for row in self.session.exec(stmt)
        }

    def import_csv_data(
        self,
        project_id: uuid.UUID,
        csv_data: list[dict],
    ) -> int:
        """Import traffic data from CSV"""
        count = 0
        for row in csv_data:
            try:
                traffic = TrafficDaily(
                    project_id=project_id,
                    date=date.fromisoformat(row["date"]),
                    source_key="csv",
                    sessions=int(row.get("sessions", 0)) or None,
                    users=int(row.get("users", 0)) or None,
                    pageviews=int(row.get("pageviews", 0)) or None,
                )
                self.session.add(traffic)
                count += 1
            except Exception:
                continue

        self.session.commit()
        return count
```

---

## 6) Celery Tasks

```python
# backend/app/tasks/ads.py
import uuid
from datetime import date, timedelta, datetime

from celery import shared_task
from sqlmodel import Session, select, delete

from app.core.db import engine
from app.core.oauth.google import GoogleOAuthClient, IntegrationAccount
from app.models.ads import AdsAccount, AdsCampaignDaily, AdsKeywordDaily
from app.models.project import Project
from app.services.ads.google_ads import GoogleAdsService


@shared_task(bind=True)
def sync_ads_account(self, project_id: str) -> dict:
    """Sync Google Ads data for a project"""
    with Session(engine) as session:
        project = session.get(Project, uuid.UUID(project_id))
        if not project:
            return {"error": "Project not found"}

        ads_account = session.exec(
            select(AdsAccount).where(AdsAccount.project_id == project.id)
        ).first()

        if not ads_account:
            return {"error": "No Ads account linked"}

        # Get integration
        account = session.exec(
            select(IntegrationAccount).where(
                IntegrationAccount.user_id == project.created_by_id,
                IntegrationAccount.provider == "google_ads",
            )
        ).first()

        if not account:
            return {"error": "No Google Ads integration"}

        oauth = GoogleOAuthClient()
        refresh_token = oauth.decrypt_token(account.refresh_token_encrypted)

        # Create Ads client
        ads_service = GoogleAdsService(refresh_token, ads_account.customer_id)

        # Sync last 7 days
        end_date = date.today() - timedelta(days=1)
        start_date = end_date - timedelta(days=7)

        # Clear existing data for date range
        session.exec(
            delete(AdsCampaignDaily)
            .where(AdsCampaignDaily.project_id == project.id)
            .where(AdsCampaignDaily.date >= start_date)
            .where(AdsCampaignDaily.date <= end_date)
        )
        session.exec(
            delete(AdsKeywordDaily)
            .where(AdsKeywordDaily.project_id == project.id)
            .where(AdsKeywordDaily.date >= start_date)
            .where(AdsKeywordDaily.date <= end_date)
        )

        # Sync campaigns
        campaigns = ads_service.get_campaign_performance(start_date, end_date)
        for c in campaigns:
            record = AdsCampaignDaily(
                project_id=project.id,
                date=date.fromisoformat(c["date"]),
                campaign_id=c["campaign_id"],
                campaign_name=c["campaign_name"],
                campaign_status=c["campaign_status"],
                impressions=c["impressions"],
                clicks=c["clicks"],
                cost_micros=c["cost_micros"],
                conversions=c["conversions"],
                conversion_value=c["conversion_value"],
            )
            session.add(record)

        # Sync keywords
        keywords = ads_service.get_keyword_performance(start_date, end_date)
        for k in keywords:
            record = AdsKeywordDaily(
                project_id=project.id,
                date=date.fromisoformat(k["date"]),
                campaign_id=k["campaign_id"],
                ad_group_id=k["ad_group_id"],
                criterion_id=k["criterion_id"],
                keyword_text=k["keyword_text"],
                match_type=k["match_type"],
                impressions=k["impressions"],
                clicks=k["clicks"],
                cost_micros=k["cost_micros"],
                conversions=k["conversions"],
                average_cpc_micros=k["average_cpc_micros"],
                quality_score=k["quality_score"],
                final_url=k["final_url"],
            )
            session.add(record)

        # Update timestamps
        ads_account.last_sync_at = datetime.utcnow()
        ads_account.sync_status = "completed"
        project.last_ppc_sync_at = datetime.utcnow()

        session.commit()

        return {
            "campaigns": len(campaigns),
            "keywords": len(keywords),
        }
```

---

## 7) API Routes

```python
# backend/app/api/routes/ads.py
import uuid
from datetime import date, timedelta
from typing import Any

from fastapi import APIRouter, Depends, HTTPException, Query
from sqlmodel import select, func

from app.api.deps import CurrentUser, SessionDep
from app.models.project import Project
from app.models.ads import (
    AdsAccount,
    AdsCampaignDaily,
    AdsKeywordDaily,
    CampaignsResponse,
    CampaignRow,
    PaidKeywordRow,
    OverlapResponse,
    OverlapRow,
)
from app.core.oauth.google import GoogleOAuthClient, IntegrationAccount
from app.services.ads.google_ads import GoogleAdsService
from app.services.ads.overlap import OverlapAnalyzer
from app.tasks.ads import sync_ads_account

router = APIRouter(prefix="/projects/{project_id}/ads", tags=["ads"])


def get_project_or_404(
    session: SessionDep, project_id: uuid.UUID, current_user: CurrentUser
) -> Project:
    project = session.get(Project, project_id)
    if not project or project.created_by_id != current_user.id:
        raise HTTPException(status_code=404, detail="Project not found")
    return project


@router.get("/accounts")
async def list_available_accounts(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
) -> list[dict]:
    """List available Google Ads accounts"""
    project = get_project_or_404(session, project_id, current_user)

    account = session.exec(
        select(IntegrationAccount).where(
            IntegrationAccount.user_id == current_user.id,
            IntegrationAccount.provider == "google_ads",
        )
    ).first()

    if not account:
        raise HTTPException(status_code=400, detail="Google Ads not connected")

    oauth = GoogleOAuthClient()
    refresh_token = oauth.decrypt_token(account.refresh_token_encrypted)

    # Use a dummy customer ID to list accessible accounts
    ads_service = GoogleAdsService(refresh_token, "")
    return ads_service.list_accessible_customers()


@router.post("/link")
async def link_ads_account(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
    customer_id: str,
) -> dict:
    """Link a Google Ads account to the project"""
    project = get_project_or_404(session, project_id, current_user)

    # Check if already linked
    existing = session.exec(
        select(AdsAccount).where(AdsAccount.project_id == project.id)
    ).first()

    if existing:
        raise HTTPException(status_code=400, detail="Account already linked")

    ads_account = AdsAccount(
        project_id=project.id,
        customer_id=customer_id,
    )
    session.add(ads_account)
    session.commit()

    # Trigger initial sync
    sync_ads_account.delay(str(project.id))

    return {"message": "Account linked, sync started"}


@router.post("/sync")
def trigger_sync(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
) -> dict:
    """Trigger manual Ads sync"""
    project = get_project_or_404(session, project_id, current_user)

    sync_ads_account.delay(str(project.id))
    return {"message": "Sync started"}


@router.get("/campaigns", response_model=CampaignsResponse)
def get_campaigns(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
    period_days: int = Query(default=28, le=90),
) -> Any:
    """Get campaign performance"""
    project = get_project_or_404(session, project_id, current_user)

    ads_account = session.exec(
        select(AdsAccount).where(AdsAccount.project_id == project.id)
    ).first()

    if not ads_account:
        raise HTTPException(status_code=400, detail="No Ads account linked")

    end_date = date.today() - timedelta(days=1)
    start_date = end_date - timedelta(days=period_days)

    stmt = (
        select(
            AdsCampaignDaily.campaign_id,
            AdsCampaignDaily.campaign_name,
            func.sum(AdsCampaignDaily.impressions).label("impressions"),
            func.sum(AdsCampaignDaily.clicks).label("clicks"),
            func.sum(AdsCampaignDaily.cost_micros).label("cost_micros"),
            func.sum(AdsCampaignDaily.conversions).label("conversions"),
        )
        .where(AdsCampaignDaily.project_id == project.id)
        .where(AdsCampaignDaily.date >= start_date)
        .where(AdsCampaignDaily.date <= end_date)
        .group_by(AdsCampaignDaily.campaign_id, AdsCampaignDaily.campaign_name)
        .order_by(func.sum(AdsCampaignDaily.clicks).desc())
    )

    rows = session.exec(stmt).all()

    items = []
    for row in rows:
        clicks = row.clicks or 0
        cost = (row.cost_micros or 0) / 1_000_000
        impressions = row.impressions or 0

        items.append(CampaignRow(
            campaign_id=row.campaign_id,
            campaign_name=row.campaign_name,
            impressions=impressions,
            clicks=clicks,
            cost=cost,
            conversions=row.conversions or 0,
            ctr=clicks / impressions if impressions > 0 else 0,
            cpc=cost / clicks if clicks > 0 else 0,
        ))

    return CampaignsResponse(
        items=items,
        total=len(items),
        currency=ads_account.currency_code,
        period_start=start_date,
        period_end=end_date,
    )


@router.get("/seo-overlap", response_model=OverlapResponse)
def get_seo_overlap(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
    period_days: int = Query(default=28, le=90),
) -> Any:
    """Get SEO + PPC overlap analysis"""
    project = get_project_or_404(session, project_id, current_user)

    analyzer = OverlapAnalyzer(session)
    results = analyzer.compute_overlap(project.id, period_days)

    items = [
        OverlapRow(
            keyword=r.keyword,
            paid_clicks=r.paid_clicks,
            paid_cost=r.paid_cost,
            paid_position=None,
            organic_clicks=r.organic_clicks,
            organic_impressions=r.organic_impressions,
            organic_position=r.organic_position,
            overlap_type=r.overlap_type,
            opportunity_score=r.opportunity_score,
        )
        for r in results[:200]
    ]

    # Summary
    both_count = sum(1 for r in results if r.overlap_type == "both")
    paid_only = sum(1 for r in results if r.overlap_type == "paid_only")
    organic_only = sum(1 for r in results if r.overlap_type == "organic_only")

    return OverlapResponse(
        items=items,
        summary={
            "total_keywords": len(results),
            "overlap_count": both_count,
            "paid_only_count": paid_only,
            "organic_only_count": organic_only,
        },
    )
```

---

## 8) Acceptance Criteria

From PRD Section 6.6:

- [x] Google Ads connector (OAuth)
- [x] Ingest campaigns, ad groups, keywords, metrics
- [x] Landing page mapping (final URL)
- [x] Dashboard: top campaigns, top paid keywords
- [x] SEO + PPC overlap analysis
- [x] Traffic panel: GSC clicks + GA4/CSV + CrUX

---

## 9) Implementation Checklist

```
[x] Create backend/app/models/ads.py
[x] Create Alembic migration for ads tables
[x] Create backend/app/services/ads/google_ads.py
[x] Create backend/app/services/ads/overlap.py
[x] Create backend/app/services/traffic/panel.py
[x] Create backend/app/tasks/ads.py
[x] Create backend/app/api/routes/ads.py
[x] Create backend/app/api/routes/traffic.py
[x] Register routes
[x] Add google-ads-python to dependencies
[x] Create frontend PPC pages
[x] Create frontend traffic panel page
[x] Write integration tests
```

---

## 11) Completion Summary (2025-12-24)

**Sprint 5: PPC + Traffic** has been fully implemented using TDD with **713 backend tests passing**.

### Backend Components:
- **Models** (`backend/app/models/ads.py`): AdsAccount, AdsCampaignDaily, AdsKeywordDaily, TransparencyCreative, TrafficDaily + API response models (30 tests)
- **Google Ads Client** (`backend/app/services/ads/google_ads.py`): OAuth integration, campaign/keyword performance queries (12 tests)
- **Overlap Analyzer** (`backend/app/services/ads/overlap.py`): SEO+PPC keyword overlap with opportunity scoring (20 tests)
- **Traffic Panel** (`backend/app/services/traffic/panel.py`): Multi-source aggregation (GA4, GSC, CrUX, CSV) (14 tests)
- **Celery Tasks** (`backend/app/tasks/ads.py`): sync_ads_account task (8 tests)
- **Ads API** (`backend/app/api/routes/ads.py`): 6 endpoints - accounts, link, sync, campaigns, keywords, seo-overlap (23 tests)
- **Traffic API** (`backend/app/api/routes/traffic.py`): 3 endpoints - panel, import-csv, sources (16 tests)

### Frontend Components:
- **PPC Dashboard** (`frontend/src/routes/_layout/projects/$projectId/ppc/index.tsx`): Campaign metrics, sync button
- **SEO+PPC Overlap** (`frontend/src/routes/_layout/projects/$projectId/ppc/overlap.tsx`): Keyword overlap analysis
- **Ad Transparency** (`frontend/src/routes/_layout/projects/$projectId/ppc/transparency.tsx`): Coming soon placeholder
- **Traffic Panel** (`frontend/src/routes/_layout/projects/$projectId/traffic/index.tsx`): Multi-source dashboard
- **Traffic Components**: TrafficChart.tsx, CsvImportModal.tsx

### Key Features:
- Google Ads OAuth integration with encrypted token storage
- Campaign and keyword-level performance tracking
- SEO+PPC keyword overlap analysis with opportunity scoring
- Multi-source traffic aggregation (GA4, GSC, CrUX, CSV import)
- Core Web Vitals display (LCP, CLS)
- Automated data sync via Celery tasks

---

## 10) Configuration

Add to `.env`:
```env
GOOGLE_ADS_DEVELOPER_TOKEN=your-developer-token
```

Note: Google Ads API requires an approved developer token. Apply at:
https://developers.google.com/google-ads/api/docs/first-call/dev-token
