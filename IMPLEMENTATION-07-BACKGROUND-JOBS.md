# Implementation 07: Background Jobs Infrastructure

> **PRD Reference:** Section 7 Non-Functional Requirements, Section 11 Background Jobs
> **Priority:** P0 (Foundation)
> **Sprint:** 0 (Parallel with Projects)
> **Dependencies:** None

---

## 1) Overview

The background jobs infrastructure enables all async processing in the platform:

- Site crawling and rendering
- GSC data ingestion
- SERP refreshes
- Link ingestion
- PPC syncing
- Scheduled tasks

**Stack:**
- **Celery** - Task queue and workers
- **Redis** - Message broker
- **Celery Beat** - Periodic task scheduler
- **Flower** - Optional monitoring UI

---

## 2) Docker Compose Setup

Add these services to `docker-compose.yml`:

```yaml
# docker-compose.yml additions

services:
  # ... existing services (db, backend, frontend) ...

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  worker:
    build:
      context: ./backend
      dockerfile: Dockerfile
    command: celery -A app.core.celery worker --loglevel=info --concurrency=4
    restart: unless-stopped
    depends_on:
      redis:
        condition: service_healthy
      db:
        condition: service_healthy
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=redis://redis:6379/0
      - CELERY_BROKER_URL=redis://redis:6379/0
      - CELERY_RESULT_BACKEND=redis://redis:6379/0
    volumes:
      - ./backend:/app
    networks:
      - app-network

  worker-render:
    build:
      context: ./backend
      dockerfile: Dockerfile.playwright
    command: celery -A app.core.celery worker --loglevel=info --concurrency=2 -Q render
    restart: unless-stopped
    depends_on:
      redis:
        condition: service_healthy
      db:
        condition: service_healthy
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=redis://redis:6379/0
      - CELERY_BROKER_URL=redis://redis:6379/0
      - CELERY_RESULT_BACKEND=redis://redis:6379/0
    volumes:
      - ./backend:/app
    networks:
      - app-network

  beat:
    build:
      context: ./backend
      dockerfile: Dockerfile
    command: celery -A app.core.celery beat --loglevel=info
    restart: unless-stopped
    depends_on:
      redis:
        condition: service_healthy
      db:
        condition: service_healthy
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=redis://redis:6379/0
      - CELERY_BROKER_URL=redis://redis:6379/0
      - CELERY_RESULT_BACKEND=redis://redis:6379/0
    volumes:
      - ./backend:/app
    networks:
      - app-network

  flower:
    image: mher/flower:2.0
    restart: unless-stopped
    depends_on:
      - redis
    environment:
      - CELERY_BROKER_URL=redis://redis:6379/0
      - FLOWER_PORT=5555
    ports:
      - "5555:5555"
    networks:
      - app-network

volumes:
  redis_data:
```

---

## 3) Playwright Worker Dockerfile

```dockerfile
# backend/Dockerfile.playwright
FROM python:3.12-slim

WORKDIR /app

# Install system dependencies for Playwright
RUN apt-get update && apt-get install -y \
    wget \
    gnupg \
    ca-certificates \
    libnss3 \
    libatk1.0-0 \
    libatk-bridge2.0-0 \
    libcups2 \
    libdrm2 \
    libxkbcommon0 \
    libxcomposite1 \
    libxdamage1 \
    libxrandr2 \
    libgbm1 \
    libasound2 \
    libpango-1.0-0 \
    libcairo2 \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen

# Install Playwright browsers
RUN playwright install chromium

COPY . .

CMD ["celery", "-A", "app.core.celery", "worker", "--loglevel=info", "-Q", "render"]
```

---

## 4) Celery Configuration

```python
# backend/app/core/celery.py
from celery import Celery
from celery.schedules import crontab

from app.core.config import settings


def create_celery_app() -> Celery:
    """Create and configure Celery application"""
    app = Celery(
        "seo_platform",
        broker=settings.CELERY_BROKER_URL,
        backend=settings.CELERY_RESULT_BACKEND,
    )

    # Configure Celery
    app.conf.update(
        # Serialization
        task_serializer="json",
        accept_content=["json"],
        result_serializer="json",

        # Time limits
        task_time_limit=3600,  # 1 hour hard limit
        task_soft_time_limit=3300,  # 55 minutes soft limit

        # Retry settings
        task_acks_late=True,
        task_reject_on_worker_lost=True,

        # Queue routing
        task_routes={
            "app.tasks.audit.crawl_pages": {"queue": "default"},
            "app.tasks.audit.render_pages": {"queue": "render"},
            "app.tasks.audit.analyze_pages": {"queue": "default"},
            "app.tasks.gsc.*": {"queue": "default"},
            "app.tasks.serp.*": {"queue": "default"},
            "app.tasks.links.*": {"queue": "low"},
            "app.tasks.ads.*": {"queue": "default"},
        },

        # Default queue
        task_default_queue="default",

        # Worker settings
        worker_prefetch_multiplier=1,
        worker_concurrency=4,

        # Result settings
        result_expires=86400,  # 24 hours

        # Scheduled tasks
        beat_schedule={
            # GSC sync - daily at 2am
            "gsc-daily-sync": {
                "task": "app.tasks.gsc.sync_all_properties",
                "schedule": crontab(hour=2, minute=0),
            },
            # SERP refresh - daily at 3am
            "serp-daily-refresh": {
                "task": "app.tasks.serp.refresh_all_due_keywords",
                "schedule": crontab(hour=3, minute=0),
            },
            # Ads sync - daily at 4am
            "ads-daily-sync": {
                "task": "app.tasks.ads.sync_all_accounts",
                "schedule": crontab(hour=4, minute=0),
            },
            # Weekly audit (for scheduled projects)
            "audit-weekly": {
                "task": "app.tasks.audit.run_scheduled_audits",
                "schedule": crontab(day_of_week=0, hour=1, minute=0),
            },
            # Cleanup old data - daily at 5am
            "cleanup-old-data": {
                "task": "app.tasks.maintenance.cleanup_old_data",
                "schedule": crontab(hour=5, minute=0),
            },
        },

        # Timezone
        timezone="UTC",
    )

    # Auto-discover tasks
    app.autodiscover_tasks([
        "app.tasks.audit",
        "app.tasks.gsc",
        "app.tasks.serp",
        "app.tasks.links",
        "app.tasks.ads",
        "app.tasks.maintenance",
    ])

    return app


# Create the celery app instance
celery_app = create_celery_app()
```

---

## 5) Configuration Updates

```python
# backend/app/core/config.py additions
class Settings(BaseSettings):
    # ... existing settings ...

    # Redis/Celery
    REDIS_URL: str = "redis://localhost:6379/0"
    CELERY_BROKER_URL: str = "redis://localhost:6379/0"
    CELERY_RESULT_BACKEND: str = "redis://localhost:6379/0"

    # Task limits
    MAX_CRAWL_PAGES: int = 1000
    MAX_RENDER_PAGES: int = 100
    SERP_REFRESH_DAILY_CAP: int = 100
    MAX_KEYWORDS_PER_PROJECT: int = 500

    # Timeouts (seconds)
    CRAWL_TIMEOUT: int = 30
    RENDER_TIMEOUT: int = 30
    API_REQUEST_TIMEOUT: int = 60
```

---

## 6) Job Status Tracking

```python
# backend/app/models/job.py
import uuid
from datetime import datetime
from enum import Enum
from sqlmodel import SQLModel, Field, Column, JSON


class JobStatus(str, Enum):
    QUEUED = "queued"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    CANCELLED = "cancelled"


class JobType(str, Enum):
    AUDIT = "audit"
    GSC_SYNC = "gsc_sync"
    GSC_BACKFILL = "gsc_backfill"
    SERP_REFRESH = "serp_refresh"
    LINKS_INGEST = "links_ingest"
    ADS_SYNC = "ads_sync"


class JobRun(SQLModel, table=True):
    """Track background job executions"""
    __tablename__ = "job_runs"

    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    job_type: JobType
    celery_task_id: str | None = None

    # References
    project_id: uuid.UUID | None = Field(foreign_key="projects.id", index=True)
    user_id: uuid.UUID | None = Field(foreign_key="user.id")

    # Status
    status: JobStatus = JobStatus.QUEUED
    progress_pct: int = 0
    progress_message: str | None = None

    # Timing
    queued_at: datetime = Field(default_factory=datetime.utcnow)
    started_at: datetime | None = None
    finished_at: datetime | None = None

    # Result
    result_json: dict | None = Field(default=None, sa_column=Column(JSON))
    error_message: str | None = None

    # Metadata
    config_json: dict | None = Field(default=None, sa_column=Column(JSON))


# API Response
class JobStatusResponse(SQLModel):
    id: uuid.UUID
    job_type: str
    status: JobStatus
    progress_pct: int
    progress_message: str | None
    started_at: datetime | None
    finished_at: datetime | None
    error_message: str | None
```

---

## 7) Base Task Class

```python
# backend/app/tasks/base.py
import uuid
from datetime import datetime
from typing import Any

from celery import Task
from sqlmodel import Session

from app.core.db import engine
from app.models.job import JobRun, JobStatus


class TrackedTask(Task):
    """Base task class with job status tracking"""

    # Set to True in subclass to enable tracking
    track_job = False
    job_type = None

    def before_start(self, task_id: str, args: tuple, kwargs: dict) -> None:
        """Called before task execution"""
        if not self.track_job:
            return

        job_run_id = kwargs.get("job_run_id")
        if job_run_id:
            with Session(engine) as session:
                job = session.get(JobRun, uuid.UUID(job_run_id))
                if job:
                    job.status = JobStatus.RUNNING
                    job.started_at = datetime.utcnow()
                    job.celery_task_id = task_id
                    session.commit()

    def on_success(self, retval: Any, task_id: str, args: tuple, kwargs: dict) -> None:
        """Called on successful completion"""
        if not self.track_job:
            return

        job_run_id = kwargs.get("job_run_id")
        if job_run_id:
            with Session(engine) as session:
                job = session.get(JobRun, uuid.UUID(job_run_id))
                if job:
                    job.status = JobStatus.COMPLETED
                    job.progress_pct = 100
                    job.finished_at = datetime.utcnow()
                    if isinstance(retval, dict):
                        job.result_json = retval
                    session.commit()

    def on_failure(
        self,
        exc: Exception,
        task_id: str,
        args: tuple,
        kwargs: dict,
        einfo: Any,
    ) -> None:
        """Called on task failure"""
        if not self.track_job:
            return

        job_run_id = kwargs.get("job_run_id")
        if job_run_id:
            with Session(engine) as session:
                job = session.get(JobRun, uuid.UUID(job_run_id))
                if job:
                    job.status = JobStatus.FAILED
                    job.finished_at = datetime.utcnow()
                    job.error_message = str(exc)
                    session.commit()

    def update_progress(
        self,
        job_run_id: str,
        progress_pct: int,
        message: str | None = None,
    ) -> None:
        """Update job progress"""
        with Session(engine) as session:
            job = session.get(JobRun, uuid.UUID(job_run_id))
            if job:
                job.progress_pct = progress_pct
                job.progress_message = message
                session.commit()
```

---

## 8) Example Task Implementation

```python
# backend/app/tasks/audit.py
import uuid
from datetime import datetime

from celery import shared_task
from sqlmodel import Session, select

from app.core.db import engine
from app.core.celery import celery_app
from app.models.audit import AuditRun, AuditStatus
from app.models.project import Project
from app.models.job import JobRun, JobStatus, JobType
from app.tasks.base import TrackedTask


@celery_app.task(bind=True, base=TrackedTask, track_job=True, job_type=JobType.AUDIT)
def run_audit_tracked(self, project_id: str, job_run_id: str) -> dict:
    """Run audit with job tracking"""
    with Session(engine) as session:
        project = session.get(Project, uuid.UUID(project_id))
        if not project:
            raise ValueError("Project not found")

        # Create audit run
        audit_run = AuditRun(
            project_id=project.id,
            config=project.settings or {},
            stats={},
        )
        session.add(audit_run)
        session.commit()
        session.refresh(audit_run)

        # Update progress
        self.update_progress(job_run_id, 10, "Starting crawl...")

        # Run crawl (simplified)
        # In real implementation, this would call crawler service
        self.update_progress(job_run_id, 50, "Crawling pages...")

        # Analyze
        self.update_progress(job_run_id, 80, "Analyzing issues...")

        # Complete
        audit_run.status = AuditStatus.COMPLETED
        audit_run.finished_at = datetime.utcnow()
        project.last_audit_at = datetime.utcnow()
        session.commit()

        return {
            "audit_run_id": str(audit_run.id),
            "pages_crawled": audit_run.stats.get("pages_crawled", 0),
        }


def start_audit_job(project_id: uuid.UUID, user_id: uuid.UUID) -> JobRun:
    """Helper to start an audit job with tracking"""
    with Session(engine) as session:
        # Create job run
        job = JobRun(
            job_type=JobType.AUDIT,
            project_id=project_id,
            user_id=user_id,
        )
        session.add(job)
        session.commit()
        session.refresh(job)

        # Queue task
        run_audit_tracked.delay(str(project_id), str(job.id))

        return job
```

---

## 9) Maintenance Tasks

```python
# backend/app/tasks/maintenance.py
from datetime import datetime, timedelta

from celery import shared_task
from sqlmodel import Session, select, delete

from app.core.db import engine
from app.models.audit import AuditRun
from app.models.serp import SerpSnapshot
from app.models.job import JobRun


@shared_task
def cleanup_old_data() -> dict:
    """Clean up old data based on retention policies"""
    with Session(engine) as session:
        now = datetime.utcnow()
        results = {}

        # Clean old job runs (keep 7 days)
        job_cutoff = now - timedelta(days=7)
        job_delete = delete(JobRun).where(JobRun.finished_at < job_cutoff)
        result = session.exec(job_delete)
        results["job_runs_deleted"] = result.rowcount

        # Clean old SERP snapshots (keep 90 days by default)
        serp_cutoff = now - timedelta(days=90)
        serp_delete = delete(SerpSnapshot).where(SerpSnapshot.fetched_at < serp_cutoff)
        result = session.exec(serp_delete)
        results["serp_snapshots_deleted"] = result.rowcount

        session.commit()
        return results


@shared_task
def health_check() -> dict:
    """Basic health check task"""
    with Session(engine) as session:
        # Test DB connection
        session.exec(select(1))

    return {
        "status": "healthy",
        "timestamp": datetime.utcnow().isoformat(),
    }
```

---

## 10) API for Job Status

```python
# backend/app/api/routes/jobs.py
import uuid
from typing import Any

from fastapi import APIRouter, Depends, HTTPException, Query
from sqlmodel import select

from app.api.deps import CurrentUser, SessionDep
from app.models.job import JobRun, JobStatusResponse


router = APIRouter(prefix="/jobs", tags=["jobs"])


@router.get("/{job_id}", response_model=JobStatusResponse)
def get_job_status(
    session: SessionDep,
    current_user: CurrentUser,
    job_id: uuid.UUID,
) -> Any:
    """Get job status"""
    job = session.get(JobRun, job_id)
    if not job:
        raise HTTPException(status_code=404, detail="Job not found")

    # Check authorization
    if job.user_id and job.user_id != current_user.id:
        raise HTTPException(status_code=403, detail="Not authorized")

    return job


@router.get("/project/{project_id}")
def list_project_jobs(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
    limit: int = Query(default=10, le=50),
) -> list[JobStatusResponse]:
    """List recent jobs for a project"""
    statement = (
        select(JobRun)
        .where(JobRun.project_id == project_id)
        .order_by(JobRun.queued_at.desc())
        .limit(limit)
    )
    jobs = session.exec(statement).all()
    return jobs


@router.post("/{job_id}/cancel")
def cancel_job(
    session: SessionDep,
    current_user: CurrentUser,
    job_id: uuid.UUID,
) -> dict:
    """Cancel a running job"""
    job = session.get(JobRun, job_id)
    if not job:
        raise HTTPException(status_code=404, detail="Job not found")

    if job.user_id and job.user_id != current_user.id:
        raise HTTPException(status_code=403, detail="Not authorized")

    if job.celery_task_id:
        from app.core.celery import celery_app
        celery_app.control.revoke(job.celery_task_id, terminate=True)

    job.status = "cancelled"
    session.commit()

    return {"message": "Job cancelled"}
```

---

## 11) Frontend Job Polling

```tsx
// frontend/src/hooks/useJobStatus.ts
import { useQuery } from "@tanstack/react-query"
import { JobsService } from "@/client"

export function useJobStatus(jobId: string | null) {
  return useQuery({
    queryKey: ["jobs", jobId],
    queryFn: () => JobsService.getJobStatus({ jobId: jobId! }),
    enabled: !!jobId,
    refetchInterval: (data) => {
      // Stop polling when job is done
      if (data?.status === "completed" || data?.status === "failed") {
        return false
      }
      return 2000 // Poll every 2 seconds
    },
  })
}


// Usage in component:
// const { data: job } = useJobStatus(currentJobId)
// if (job?.status === "running") {
//   return <Progress value={job.progress_pct} />
// }
```

---

## 12) Monitoring & Observability

### Flower Dashboard

Access at `http://localhost:5555` to see:
- Active workers
- Task queues
- Task history
- Worker stats

### Logging

```python
# backend/app/core/logging.py
import logging
import structlog

def configure_logging():
    """Configure structured logging"""
    structlog.configure(
        processors=[
            structlog.stdlib.filter_by_level,
            structlog.stdlib.add_logger_name,
            structlog.stdlib.add_log_level,
            structlog.stdlib.PositionalArgumentsFormatter(),
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.processors.UnicodeDecoder(),
            structlog.processors.JSONRenderer(),
        ],
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
        wrapper_class=structlog.stdlib.BoundLogger,
        cache_logger_on_first_use=True,
    )

    # Configure Celery logging
    logging.getLogger("celery").setLevel(logging.INFO)
```

---

## 13) Acceptance Criteria

From PRD Section 7 & 11:

- [x] Celery workers with Redis broker
- [x] Celery Beat for scheduled tasks
- [x] Task routing by queue
- [x] Retry with exponential backoff
- [x] Progress tracking for long jobs
- [x] Failure visibility
- [x] Flower monitoring (optional)

---

## 14) Implementation Checklist

```
[ ] Add Redis to docker-compose.yml
[ ] Add worker service to docker-compose.yml
[ ] Add beat service to docker-compose.yml
[ ] Create backend/Dockerfile.playwright
[ ] Create backend/app/core/celery.py
[ ] Create backend/app/models/job.py
[ ] Create Alembic migration for job_runs table
[ ] Create backend/app/tasks/base.py
[ ] Create backend/app/tasks/maintenance.py
[ ] Create backend/app/api/routes/jobs.py
[ ] Update backend/app/core/config.py with Redis settings
[ ] Add celery to pyproject.toml dependencies
[ ] Add redis to pyproject.toml dependencies
[ ] Create frontend job polling hook
[ ] Test worker startup
[ ] Test beat scheduler
[ ] Test task execution and tracking
```

---

## 15) Commands Reference

```bash
# Start all services
docker-compose up -d

# View worker logs
docker-compose logs -f worker

# View beat logs
docker-compose logs -f beat

# Scale workers
docker-compose up -d --scale worker=3

# Run a task manually (from backend container)
docker-compose exec backend python -c "
from app.tasks.maintenance import health_check
result = health_check.delay()
print(result.get())
"

# Check queue lengths
docker-compose exec redis redis-cli LLEN celery

# Purge all pending tasks
docker-compose exec backend celery -A app.core.celery purge
```
