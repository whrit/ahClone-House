# Developer Guide

Complete guide for developers working on the SEO Platform.

---

## Table of Contents

1. [Local Development Setup](#local-development-setup)
2. [Project Structure](#project-structure)
3. [Backend Development](#backend-development)
4. [Frontend Development](#frontend-development)
5. [Database Migrations](#database-migrations)
6. [Testing](#testing)
7. [Adding New Features](#adding-new-features)
8. [Code Style and Conventions](#code-style-and-conventions)
9. [Debugging](#debugging)
10. [Performance Optimization](#performance-optimization)

---

## Local Development Setup

### Prerequisites

Install the following tools:

| Tool | Version | Purpose | Installation |
|------|---------|---------|--------------|
| **Git** | Latest | Version control | [git-scm.com](https://git-scm.com/) |
| **Docker** | 24.0+ | Container runtime | [docker.com](https://www.docker.com/get-started) |
| **Docker Compose** | 2.x+ | Multi-container orchestration | Included with Docker Desktop |
| **Python** | 3.12+ | Backend development | [python.org](https://www.python.org/) |
| **uv** | Latest | Python package manager | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| **Node.js** | 20+ | Frontend development | [nodejs.org](https://nodejs.org/) |
| **npm** | 10+ | Node package manager | Included with Node.js |

### Clone the Repository

```bash
git clone https://github.com/yourusername/seo-platform.git
cd seo-platform
```

### Configure Environment

Copy the example environment file:

```bash
cp .env.example .env
```

Edit `.env` for local development (minimal config):

```bash
DOMAIN=localhost
FRONTEND_HOST=http://localhost:5173
ENVIRONMENT=local

SECRET_KEY=local-dev-secret-key-change-in-production
FIRST_SUPERUSER=admin@example.com
FIRST_SUPERUSER_PASSWORD=admin

POSTGRES_PASSWORD=postgres
```

### Start Development Stack

```bash
docker compose watch
```

This command:
- Starts all services (backend, frontend, database, Redis, Celery)
- Watches for code changes and auto-reloads
- Mounts source code as volumes for hot reloading

**Wait for services to be ready** (check logs):
```bash
docker compose logs -f
```

Look for:
- `backend_1 | Application startup complete`
- `worker_1 | celery@worker ready`
- `db_1 | database system is ready to accept connections`

### Verify Installation

Open these URLs in your browser:

- **Frontend**: http://localhost:5173
- **Backend API**: http://localhost:8000/docs
- **Adminer (DB UI)**: http://localhost:8080
- **Flower (Celery UI)**: http://localhost:5555
- **Traefik Dashboard**: http://localhost:8090
- **MailCatcher**: http://localhost:1080 (for email testing)

---

## Project Structure

### Repository Layout

```
seo-platform/
├── backend/                # FastAPI backend
│   ├── alembic.ini        # Alembic configuration
│   ├── app/
│   │   ├── alembic/       # Database migrations
│   │   ├── api/           # API endpoints
│   │   │   ├── deps.py    # Dependency injection
│   │   │   ├── main.py    # API router setup
│   │   │   └── routes/    # Route modules
│   │   ├── core/          # Core utilities
│   │   │   ├── cache.py       # Redis caching
│   │   │   ├── celery.py      # Celery app factory
│   │   │   ├── config.py      # Settings
│   │   │   ├── db.py          # Database session
│   │   │   ├── oauth/         # OAuth helpers
│   │   │   ├── rate_limit.py  # Rate limiting
│   │   │   ├── security.py    # JWT, password hashing
│   │   │   └── exceptions.py  # Custom exceptions
│   │   ├── email-templates/   # Email HTML templates
│   │   ├── models/        # SQLModel database models
│   │   │   ├── __init__.py    # Model registry
│   │   │   ├── user.py
│   │   │   ├── project.py
│   │   │   ├── audit.py
│   │   │   ├── gsc.py
│   │   │   ├── serp.py
│   │   │   ├── links.py
│   │   │   ├── ads.py
│   │   │   ├── job.py
│   │   │   └── integration.py
│   │   ├── services/      # Business logic
│   │   │   ├── audit/         # Audit engine
│   │   │   │   ├── crawler.py
│   │   │   │   ├── renderer.py
│   │   │   │   ├── analyzer.py
│   │   │   │   └── differ.py
│   │   │   ├── gsc/           # GSC integration
│   │   │   │   ├── client.py
│   │   │   │   ├── ingestor.py
│   │   │   │   ├── opportunities.py
│   │   │   │   └── clustering.py
│   │   │   ├── serp/          # SERP tracking
│   │   │   │   ├── providers/
│   │   │   │   │   ├── base.py
│   │   │   │   │   └── gsc_provider.py
│   │   │   │   └── tracker.py
│   │   │   ├── links/         # Backlinks
│   │   │   │   ├── commoncrawl.py
│   │   │   │   ├── aggregator.py
│   │   │   │   └── competitive.py
│   │   │   ├── ads/           # PPC
│   │   │   │   ├── google_ads.py
│   │   │   │   └── overlap.py
│   │   │   └── traffic/       # Traffic panel
│   │   │       └── panel.py
│   │   ├── tasks/         # Celery tasks
│   │   │   ├── __init__.py
│   │   │   ├── audit.py
│   │   │   ├── gsc.py
│   │   │   ├── serp.py
│   │   │   ├── links.py
│   │   │   ├── ads.py
│   │   │   └── maintenance.py
│   │   ├── crud.py        # CRUD utilities
│   │   ├── main.py        # FastAPI app setup
│   │   └── utils.py       # Shared utilities
│   ├── scripts/           # Utility scripts
│   │   ├── prestart.sh        # Pre-start migrations
│   │   ├── test.sh            # Run tests
│   │   └── lint.sh            # Linting
│   ├── tests/             # Test suite
│   │   ├── api/               # API endpoint tests
│   │   ├── crud/              # CRUD tests
│   │   ├── services/          # Service logic tests
│   │   ├── utils/             # Utility tests
│   │   └── conftest.py        # Pytest fixtures
│   ├── Dockerfile         # Backend Docker image
│   ├── pyproject.toml     # Python dependencies
│   └── uv.lock            # Locked dependencies
│
├── frontend/              # React frontend
│   ├── public/            # Static assets
│   ├── src/
│   │   ├── assets/        # Images, fonts, etc.
│   │   ├── client/        # Auto-generated API client
│   │   ├── components/    # Reusable components
│   │   │   ├── Common/
│   │   │   ├── Audits/
│   │   │   ├── Keywords/
│   │   │   ├── RankTracker/
│   │   │   ├── Links/
│   │   │   └── ui/        # shadcn/ui components
│   │   ├── hooks/         # Custom React hooks
│   │   ├── lib/           # Utilities
│   │   ├── routes/        # TanStack Router pages
│   │   │   ├── _layout/       # Authenticated layout
│   │   │   │   ├── index.tsx          # Dashboard
│   │   │   │   ├── projects/          # Projects
│   │   │   │   │   ├── index.tsx
│   │   │   │   │   ├── new.tsx
│   │   │   │   │   └── $projectId/    # Project detail
│   │   │   │   │       ├── audits/
│   │   │   │   │       ├── keywords/
│   │   │   │   │       ├── rank-tracker/
│   │   │   │   │       ├── links/
│   │   │   │   │       ├── ppc/
│   │   │   │   │       └── traffic/
│   │   │   └── login.tsx      # Login page
│   │   ├── main.tsx       # App entry point
│   │   └── routeTree.gen.ts   # Generated routes
│   ├── Dockerfile         # Frontend Docker image
│   ├── package.json       # Node dependencies
│   └── vite.config.ts     # Vite configuration
│
├── scripts/               # Project-level scripts
├── .github/               # GitHub Actions workflows
├── docker-compose.yml     # Development stack
├── docker-compose.override.yml  # Dev overrides
├── docker-compose.prod.yml      # Production stack
├── .env.example           # Environment template
└── README.md              # Project overview
```

### Key Directories

#### Backend (`backend/app/`)

- **`api/routes/`**: API endpoint definitions (one file per resource)
- **`models/`**: SQLModel database schemas (ORM + Pydantic validation)
- **`services/`**: Business logic (isolated from API layer)
- **`tasks/`**: Celery background tasks
- **`core/`**: Configuration, security, utilities

#### Frontend (`frontend/src/`)

- **`routes/`**: Page components (file-based routing)
- **`components/`**: Reusable UI components
- **`client/`**: Auto-generated TypeScript API client
- **`hooks/`**: Custom React hooks for state and effects

---

## Backend Development

### Python Environment Setup

#### Using uv (Recommended)

```bash
cd backend

# Install dependencies
uv sync

# Activate virtual environment
source .venv/bin/activate

# Verify Python version
python --version  # Should be 3.12+
```

#### Using pip (Alternative)

```bash
cd backend

# Create virtual environment
python3.12 -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt
```

### Running Backend Locally (Without Docker)

Useful for debugging with IDE breakpoints:

```bash
cd backend

# Ensure database and Redis are running (via Docker Compose)
docker compose up -d db redis

# Set environment variables
export POSTGRES_SERVER=localhost
export REDIS_URL=redis://localhost:6379/0

# Run FastAPI development server
fastapi dev app/main.py
```

Access at: http://localhost:8000

### Backend Project Structure

#### API Routes (`app/api/routes/`)

Each route file handles one resource:

```python
# app/api/routes/projects.py
from fastapi import APIRouter, Depends
from sqlmodel import Session

from app.api.deps import get_current_user, get_db
from app.models import Project, User
import app.crud as crud

router = APIRouter()

@router.get("/", response_model=list[Project])
def list_projects(
    *,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
    skip: int = 0,
    limit: int = 100,
) -> list[Project]:
    """List all projects for current user."""
    return crud.project.get_multi_by_owner(
        db, owner_id=current_user.id, skip=skip, limit=limit
    )
```

**Key Patterns**:
- Use dependency injection for `db` session and `current_user`
- Return Pydantic models for automatic validation and serialization
- Keep route handlers thin (delegate to CRUD/services)

#### Models (`app/models/`)

SQLModel models serve dual purpose: ORM + Pydantic validation.

```python
# app/models/project.py
from sqlmodel import Field, Relationship, SQLModel
import uuid
from datetime import datetime

class ProjectBase(SQLModel):
    """Shared properties"""
    name: str = Field(index=True, min_length=1, max_length=255)
    domain: str = Field(index=True, min_length=1, max_length=255)
    description: str | None = Field(default=None, max_length=1000)

class Project(ProjectBase, table=True):
    """Database model"""
    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    owner_id: uuid.UUID = Field(foreign_key="user.id", index=True)
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)

    # Relationships
    owner: "User" = Relationship(back_populates="projects")
    audits: list["AuditRun"] = Relationship(back_populates="project")

class ProjectCreate(ProjectBase):
    """Properties to receive on creation"""
    pass

class ProjectUpdate(SQLModel):
    """Properties to receive on update (all optional)"""
    name: str | None = None
    domain: str | None = None
    description: str | None = None

class ProjectPublic(ProjectBase):
    """Properties to return to client"""
    id: uuid.UUID
    owner_id: uuid.UUID
    created_at: datetime
    updated_at: datetime
```

**Best Practices**:
- Use `ProjectBase` for shared properties
- `Project` is the database table
- `ProjectCreate` for POST requests
- `ProjectUpdate` for PATCH requests (all fields optional)
- `ProjectPublic` for API responses

#### Services (`app/services/`)

Encapsulate business logic:

```python
# app/services/audit/crawler.py
import httpx
from urllib.parse import urljoin, urlparse

class Crawler:
    def __init__(self, start_url: str, max_pages: int = 1000):
        self.start_url = start_url
        self.max_pages = max_pages
        self.visited = set()
        self.to_visit = [start_url]

    async def crawl(self) -> list[dict]:
        """Crawl site and return page data."""
        pages = []

        async with httpx.AsyncClient() as client:
            while self.to_visit and len(pages) < self.max_pages:
                url = self.to_visit.pop(0)
                if url in self.visited:
                    continue

                page_data = await self._fetch_page(client, url)
                pages.append(page_data)
                self.visited.add(url)

                # Extract links
                links = self._extract_links(page_data["html"], url)
                self.to_visit.extend(links)

        return pages

    async def _fetch_page(self, client: httpx.AsyncClient, url: str) -> dict:
        """Fetch single page."""
        # Implementation...
        pass
```

**Key Principles**:
- Services are independent of FastAPI
- Can be unit tested without HTTP layer
- Use async/await for I/O operations
- Accept dependencies via constructor (testable)

#### Celery Tasks (`app/tasks/`)

Background job definitions:

```python
# app/tasks/audit.py
from app.core.celery import celery_app
from app.core.db import SessionLocal
from app.services.audit.crawler import Crawler
from app.models import AuditRun
import uuid

@celery_app.task(bind=True)
def run_audit(self, audit_id: uuid.UUID):
    """Background task to run audit."""
    db = SessionLocal()

    try:
        audit = db.get(AuditRun, audit_id)
        if not audit:
            return

        # Update status
        audit.status = "running"
        db.commit()

        # Run crawler
        crawler = Crawler(
            start_url=f"https://{audit.project.domain}",
            max_pages=audit.max_pages
        )
        pages = await crawler.crawl()

        # Store results
        for page_data in pages:
            # Save to database...
            pass

        # Update status
        audit.status = "completed"
        db.commit()

    except Exception as e:
        audit.status = "failed"
        audit.error_message = str(e)
        db.commit()
        raise
    finally:
        db.close()
```

**Task Best Practices**:
- Use `bind=True` to access task context (for retries, progress)
- Always handle exceptions and update job status
- Close database sessions in `finally`
- Use task retries for transient failures
- Report progress with `self.update_state()`

### Database Sessions

Use dependency injection for database sessions:

```python
from sqlmodel import Session
from app.core.db import engine

def get_db():
    """Dependency for database session."""
    with Session(engine) as session:
        yield session
```

**In routes**:
```python
@router.get("/")
def get_items(db: Session = Depends(get_db)):
    return db.exec(select(Item)).all()
```

### Authentication

Protected routes use `get_current_user` dependency:

```python
from app.api.deps import get_current_user
from app.models import User

@router.get("/me")
def read_user_me(current_user: User = Depends(get_current_user)):
    return current_user
```

**Superuser-only routes**:
```python
from app.api.deps import get_current_superuser

@router.get("/admin")
def admin_only(current_user: User = Depends(get_current_superuser)):
    return {"message": "Admin access"}
```

---

## Frontend Development

### Node Environment Setup

```bash
cd frontend

# Install dependencies
npm install

# Verify Node version
node --version  # Should be 20+
```

### Running Frontend Locally (Without Docker)

```bash
cd frontend

# Start development server
npm run dev

# Access at http://localhost:5173
```

**Hot Module Replacement** (HMR):
- Changes to `.tsx` files auto-reload
- Preserves React component state
- Instant feedback loop

### Generating API Client

The frontend uses an auto-generated TypeScript client:

```bash
cd frontend

# Generate client from backend OpenAPI schema
npm run generate-client
```

**When to regenerate**:
- After adding/modifying backend routes
- After changing request/response models
- After pulling latest backend changes

**Generated files**: `frontend/src/client/`

### Frontend Project Structure

#### Routes (`src/routes/`)

File-based routing with TanStack Router:

```tsx
// src/routes/_layout/projects/index.tsx
import { createFileRoute } from '@tanstack/react-router'
import { ProjectsService } from '@/client'
import { ProjectList } from '@/components/Projects/ProjectList'

export const Route = createFileRoute('/_layout/projects/')({
  component: ProjectsPage,
  loader: async () => {
    // Fetch data before rendering
    const projects = await ProjectsService.readProjects()
    return { projects }
  },
})

function ProjectsPage() {
  const { projects } = Route.useLoaderData()

  return (
    <div>
      <h1>Projects</h1>
      <ProjectList projects={projects} />
    </div>
  )
}
```

**Route patterns**:
- `_layout/` = authenticated layout (sidebar, header)
- `$projectId` = dynamic parameter
- `index.tsx` = default route for directory

#### Components (`src/components/`)

Reusable UI components:

```tsx
// src/components/Projects/ProjectCard.tsx
import { Card, CardHeader, CardTitle, CardDescription } from '@/components/ui/card'
import { Project } from '@/client'

interface ProjectCardProps {
  project: Project
  onClick: (id: string) => void
}

export function ProjectCard({ project, onClick }: ProjectCardProps) {
  return (
    <Card
      className="cursor-pointer hover:shadow-lg transition-shadow"
      onClick={() => onClick(project.id)}
    >
      <CardHeader>
        <CardTitle>{project.name}</CardTitle>
        <CardDescription>{project.domain}</CardDescription>
      </CardHeader>
    </Card>
  )
}
```

**Component Best Practices**:
- Use TypeScript for all components
- Define prop interfaces
- Use shadcn/ui components for consistency
- Avoid inline styles (use Tailwind classes)
- Extract logic to custom hooks

#### Custom Hooks (`src/hooks/`)

Encapsulate reusable logic:

```tsx
// src/hooks/useProjects.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { ProjectsService, ProjectCreate } from '@/client'

export function useProjects() {
  const queryClient = useQueryClient()

  const { data: projects, isLoading } = useQuery({
    queryKey: ['projects'],
    queryFn: () => ProjectsService.readProjects(),
  })

  const createProject = useMutation({
    mutationFn: (data: ProjectCreate) => ProjectsService.createProject({ requestBody: data }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['projects'] })
    },
  })

  return { projects, isLoading, createProject }
}
```

**Usage in components**:
```tsx
function ProjectsList() {
  const { projects, isLoading, createProject } = useProjects()

  if (isLoading) return <Spinner />

  return (
    <div>
      {projects.map(project => (
        <ProjectCard key={project.id} project={project} />
      ))}
    </div>
  )
}
```

### State Management

#### React Query (TanStack Query)

Used for server state:

**Query (read)**:
```tsx
const { data, isLoading, error } = useQuery({
  queryKey: ['projects', projectId],
  queryFn: () => ProjectsService.readProject({ id: projectId }),
})
```

**Mutation (write)**:
```tsx
const mutation = useMutation({
  mutationFn: (data) => ProjectsService.updateProject({ id, requestBody: data }),
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['projects', id] })
  },
})

mutation.mutate({ name: 'New Name' })
```

#### Local State

Use `useState` for component-local state:

```tsx
const [isOpen, setIsOpen] = useState(false)
const [selectedTab, setSelectedTab] = useState('issues')
```

### Styling

Using **Tailwind CSS** + **shadcn/ui**:

```tsx
<div className="flex items-center justify-between p-4 rounded-lg border border-gray-200 dark:border-gray-800">
  <h2 className="text-2xl font-bold">Title</h2>
  <Button variant="outline" size="sm">Action</Button>
</div>
```

**Dark Mode**:
- Automatically supported via `dark:` prefix
- Theme toggle in user settings
- Persisted to localStorage

---

## Database Migrations

### Alembic Workflow

#### Create Migration

After modifying models:

```bash
cd backend

# Auto-generate migration from model changes
docker compose exec backend alembic revision --autogenerate -m "Add project settings table"

# Migration file created in app/alembic/versions/
```

#### Review Migration

```bash
# Check generated migration
cat backend/app/alembic/versions/xxxx_add_project_settings_table.py
```

**Manual adjustments may be needed** for:
- Index creation
- Data migrations
- Complex constraint changes

#### Apply Migration

```bash
# Apply to local database
docker compose exec backend alembic upgrade head

# Check current revision
docker compose exec backend alembic current

# View migration history
docker compose exec backend alembic history
```

#### Rollback Migration

```bash
# Downgrade one revision
docker compose exec backend alembic downgrade -1

# Downgrade to specific revision
docker compose exec backend alembic downgrade <revision_id>
```

### Migration Best Practices

1. **Always review auto-generated migrations** before committing
2. **Test migrations on a copy of production data** before deploying
3. **Make migrations reversible** (implement `downgrade()`)
4. **Avoid breaking changes** in migrations (use multi-step approach)
5. **Add indexes for foreign keys** manually if Alembic misses them

---

## Testing

### Backend Tests

#### Running Tests

```bash
# Run all tests
docker compose exec backend pytest

# Run specific test file
docker compose exec backend pytest tests/api/test_projects.py

# Run with coverage
docker compose exec backend pytest --cov=app --cov-report=html

# Run in watch mode (auto-rerun on changes)
docker compose exec backend pytest-watch
```

#### Test Structure

```python
# tests/api/test_projects.py
import pytest
from fastapi.testclient import TestClient
from sqlmodel import Session

from app.main import app
from app.models import Project, User

client = TestClient(app)

def test_create_project(
    db: Session,
    client: TestClient,
    normal_user_token_headers: dict
):
    """Test creating a new project."""
    data = {
        "name": "Test Project",
        "domain": "example.com",
        "description": "Test description"
    }

    response = client.post(
        "/api/v1/projects/",
        headers=normal_user_token_headers,
        json=data,
    )

    assert response.status_code == 201
    content = response.json()
    assert content["name"] == data["name"]
    assert content["domain"] == data["domain"]
    assert "id" in content
```

#### Fixtures (`tests/conftest.py`)

Common test fixtures:

```python
import pytest
from sqlmodel import Session, create_engine
from app.core.db import get_db

@pytest.fixture(scope="session")
def db_engine():
    """Create test database engine."""
    engine = create_engine("postgresql://test:test@localhost/test_db")
    yield engine
    engine.dispose()

@pytest.fixture
def db(db_engine):
    """Create test database session."""
    with Session(db_engine) as session:
        yield session
        session.rollback()

@pytest.fixture
def normal_user_token_headers(client, db):
    """Get auth headers for normal user."""
    # Create test user and return auth headers
    pass
```

#### Testing Patterns

**API Endpoint Tests**:
```python
def test_read_project_not_found(client, normal_user_token_headers):
    response = client.get(
        f"/api/v1/projects/{uuid.uuid4()}",
        headers=normal_user_token_headers,
    )
    assert response.status_code == 404
```

**Service Tests** (no HTTP):
```python
from app.services.audit.analyzer import Analyzer

def test_analyzer_detects_missing_title():
    html = "<html><body>No title</body></html>"
    issues = Analyzer.analyze_page(html, "https://example.com")

    assert any(issue.type == "MISSING_TITLE" for issue in issues)
```

**Async Tests**:
```python
import pytest

@pytest.mark.asyncio
async def test_crawler_fetches_pages():
    from app.services.audit.crawler import Crawler

    crawler = Crawler("https://example.com", max_pages=10)
    pages = await crawler.crawl()

    assert len(pages) > 0
    assert pages[0]["url"] == "https://example.com"
```

### Frontend Tests

#### Running Tests

```bash
cd frontend

# Run unit tests
npm test

# Run E2E tests (Playwright)
npm run test:e2e

# Run with UI
npm run test:e2e:ui
```

#### Component Tests (Vitest)

```tsx
// components/Projects/ProjectCard.test.tsx
import { render, screen } from '@testing-library/react'
import { ProjectCard } from './ProjectCard'

describe('ProjectCard', () => {
  it('renders project name and domain', () => {
    const project = {
      id: '123',
      name: 'Test Project',
      domain: 'example.com',
    }

    render(<ProjectCard project={project} onClick={() => {}} />)

    expect(screen.getByText('Test Project')).toBeInTheDocument()
    expect(screen.getByText('example.com')).toBeInTheDocument()
  })

  it('calls onClick when clicked', () => {
    const onClick = vi.fn()
    const project = { id: '123', name: 'Test', domain: 'example.com' }

    render(<ProjectCard project={project} onClick={onClick} />)
    screen.getByText('Test').click()

    expect(onClick).toHaveBeenCalledWith('123')
  })
})
```

#### E2E Tests (Playwright)

```typescript
// tests/e2e/projects.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Projects', () => {
  test('create new project', async ({ page }) => {
    await page.goto('http://localhost:5173')

    // Login
    await page.fill('input[name="email"]', 'admin@example.com')
    await page.fill('input[name="password"]', 'admin')
    await page.click('button[type="submit"]')

    // Navigate to projects
    await page.click('text=Projects')

    // Create project
    await page.click('text=New Project')
    await page.fill('input[name="name"]', 'E2E Test Project')
    await page.fill('input[name="domain"]', 'e2e-test.com')
    await page.click('button[type="submit"]')

    // Verify
    await expect(page.locator('text=E2E Test Project')).toBeVisible()
  })
})
```

---

## Adding New Features

### Example: Adding a New API Endpoint

Let's add a feature to export audit issues as PDF.

#### Step 1: Add Model (if needed)

```python
# backend/app/models/audit.py (existing)
class AuditRun(SQLModel, table=True):
    # ...existing fields...
    pdf_export_url: str | None = None
```

#### Step 2: Create Service

```python
# backend/app/services/audit/exporter.py
from reportlab.lib.pagesizes import letter
from reportlab.pdfgen import canvas

class PDFExporter:
    def export_issues(self, audit_id: uuid.UUID) -> bytes:
        """Export audit issues as PDF."""
        # Fetch issues from database
        # Generate PDF
        # Return bytes
        pass
```

#### Step 3: Add API Route

```python
# backend/app/api/routes/audits.py
from fastapi.responses import StreamingResponse
from app.services.audit.exporter import PDFExporter

@router.get("/{audit_id}/export/pdf")
def export_audit_pdf(
    audit_id: uuid.UUID,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    """Export audit issues as PDF."""
    audit = db.get(AuditRun, audit_id)
    if not audit:
        raise HTTPException(status_code=404)

    # Check authorization
    if audit.project.owner_id != current_user.id:
        raise HTTPException(status_code=403)

    # Generate PDF
    exporter = PDFExporter()
    pdf_bytes = exporter.export_issues(audit_id)

    return StreamingResponse(
        io.BytesIO(pdf_bytes),
        media_type="application/pdf",
        headers={"Content-Disposition": f"attachment; filename=audit_{audit_id}.pdf"}
    )
```

#### Step 4: Add Tests

```python
# backend/tests/api/test_audits.py
def test_export_audit_pdf(client, normal_user_token_headers, db):
    # Create test audit
    audit = create_test_audit(db)

    # Export
    response = client.get(
        f"/api/v1/audits/{audit.id}/export/pdf",
        headers=normal_user_token_headers,
    )

    assert response.status_code == 200
    assert response.headers["content-type"] == "application/pdf"
```

#### Step 5: Regenerate Frontend Client

```bash
cd frontend
npm run generate-client
```

#### Step 6: Add Frontend UI

```tsx
// components/Audits/AuditActions.tsx
import { AuditsService } from '@/client'

export function AuditActions({ auditId }: { auditId: string }) {
  const handleExportPDF = async () => {
    const blob = await AuditsService.exportAuditPdf({ auditId })

    // Download file
    const url = window.URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url
    a.download = `audit_${auditId}.pdf`
    a.click()
  }

  return (
    <Button onClick={handleExportPDF}>
      Export PDF
    </Button>
  )
}
```

#### Step 7: Create Migration

```bash
docker compose exec backend alembic revision --autogenerate -m "Add pdf_export_url to audit_run"
docker compose exec backend alembic upgrade head
```

#### Step 8: Test End-to-End

1. Start stack: `docker compose watch`
2. Create audit via UI
3. Click "Export PDF"
4. Verify PDF downloads correctly

---

## Code Style and Conventions

### Backend (Python)

#### Code Formatting

Using **Ruff** for linting and formatting:

```bash
# Auto-format code
docker compose exec backend ruff format .

# Check for issues
docker compose exec backend ruff check .

# Auto-fix issues
docker compose exec backend ruff check --fix .
```

#### Type Checking

Using **mypy**:

```bash
docker compose exec backend mypy app
```

**Must pass with zero errors** before committing.

#### Naming Conventions

- **Functions/methods**: `snake_case`
- **Classes**: `PascalCase`
- **Constants**: `UPPER_CASE`
- **Private functions**: `_leading_underscore`

#### Import Order

```python
# 1. Standard library
import os
import sys
from datetime import datetime

# 2. Third-party
from fastapi import APIRouter, Depends
from sqlmodel import Session, select

# 3. Local
from app.api.deps import get_db
from app.models import Project
import app.crud as crud
```

### Frontend (TypeScript/React)

#### Code Formatting

Using **Prettier** and **ESLint**:

```bash
# Auto-format code
npm run format

# Check for issues
npm run lint

# Auto-fix issues
npm run lint:fix
```

#### Naming Conventions

- **Components**: `PascalCase` (e.g., `ProjectCard`)
- **Hooks**: `camelCase` with `use` prefix (e.g., `useProjects`)
- **Files**: Match component name (e.g., `ProjectCard.tsx`)
- **Props interfaces**: `ComponentNameProps` (e.g., `ProjectCardProps`)

#### Component Structure

```tsx
// 1. Imports
import { useState } from 'react'
import { Button } from '@/components/ui/button'

// 2. Types
interface MyComponentProps {
  title: string
  onSave: (data: string) => void
}

// 3. Component
export function MyComponent({ title, onSave }: MyComponentProps) {
  // 3a. Hooks
  const [value, setValue] = useState('')

  // 3b. Handlers
  const handleSubmit = () => {
    onSave(value)
  }

  // 3c. Render
  return (
    <div>
      <h1>{title}</h1>
      <input value={value} onChange={(e) => setValue(e.target.value)} />
      <Button onClick={handleSubmit}>Save</Button>
    </div>
  )
}
```

### Pre-commit Hooks

Install pre-commit hooks:

```bash
# Using uv
uv run pre-commit install

# Manual install
pip install pre-commit
pre-commit install
```

**What runs on commit**:
- Ruff (Python linting)
- mypy (type checking)
- ESLint (TypeScript linting)
- Prettier (formatting)
- YAML/TOML validation

### Git Commit Messages

Follow conventional commits:

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types**:
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation
- `style`: Formatting
- `refactor`: Code restructuring
- `test`: Adding tests
- `chore`: Maintenance

**Examples**:
```
feat(audits): add PDF export endpoint

Implements PDF generation for audit issues using ReportLab.
Includes download endpoint and frontend button.

Closes #123
```

---

## Debugging

### Backend Debugging

#### VS Code Debugger

Create `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Python: FastAPI",
      "type": "python",
      "request": "launch",
      "module": "uvicorn",
      "args": ["app.main:app", "--reload", "--host", "0.0.0.0", "--port", "8000"],
      "jinja": true,
      "justMyCode": false
    }
  ]
}
```

**Set breakpoints** in code, then press F5 to start debugging.

#### Print Debugging

```python
import logging

logger = logging.getLogger(__name__)

@router.get("/debug")
def debug_endpoint():
    logger.debug("Debug message")
    logger.info("Info message")
    logger.warning("Warning message")
    logger.error("Error message")
    return {"status": "ok"}
```

**View logs**:
```bash
docker compose logs -f backend
```

### Frontend Debugging

#### Browser DevTools

- **Console**: `console.log()`, `console.error()`
- **Network tab**: Inspect API requests
- **React DevTools**: Component hierarchy and props

#### React Query DevTools

Auto-enabled in development:

```tsx
// Floating panel in bottom-right shows:
// - Active queries
// - Cache state
// - Mutations
```

### Celery Task Debugging

#### View Task Logs

```bash
docker compose logs -f worker
```

#### Flower UI

Access http://localhost:5555:

- View active tasks
- Task history
- Task details (args, kwargs, result)
- Worker status

#### Manual Task Execution

```python
# In Python shell
from app.tasks.audit import run_audit
import uuid

audit_id = uuid.UUID("...")
result = run_audit.apply(args=[audit_id])
print(result.get())
```

---

## Performance Optimization

### Backend Performance

#### Database Query Optimization

**Use eager loading** to avoid N+1 queries:

```python
from sqlmodel import select
from sqlalchemy.orm import selectinload

# Bad: N+1 queries
projects = db.exec(select(Project)).all()
for project in projects:
    print(project.owner.email)  # Separate query for each project

# Good: 2 queries total
statement = select(Project).options(selectinload(Project.owner))
projects = db.exec(statement).all()
for project in projects:
    print(project.owner.email)  # No additional query
```

**Use indexes** for frequently queried columns:

```python
class Project(SQLModel, table=True):
    domain: str = Field(index=True)  # Index for lookups
    owner_id: uuid.UUID = Field(foreign_key="user.id", index=True)
```

#### Caching

Use Redis caching for expensive queries:

```python
from app.core.cache import cached

@cached(ttl=3600)  # Cache for 1 hour
def get_project_stats(project_id: uuid.UUID) -> dict:
    # Expensive computation
    return {...}
```

#### Async I/O

Use async for I/O-bound operations:

```python
import httpx

async def fetch_multiple_urls(urls: list[str]) -> list[dict]:
    async with httpx.AsyncClient() as client:
        tasks = [client.get(url) for url in urls]
        responses = await asyncio.gather(*tasks)
        return [r.json() for r in responses]
```

### Frontend Performance

#### Code Splitting

Use lazy loading for routes:

```tsx
import { lazy } from 'react'

const ProjectDetail = lazy(() => import('./routes/_layout/projects/$projectId'))
```

#### Memoization

Prevent unnecessary re-renders:

```tsx
import { useMemo, useCallback } from 'react'

function ExpensiveComponent({ items }) {
  // Memoize computation
  const total = useMemo(() => {
    return items.reduce((sum, item) => sum + item.price, 0)
  }, [items])

  // Memoize callback
  const handleClick = useCallback((id) => {
    console.log(id)
  }, [])

  return <div>{total}</div>
}
```

#### Virtual Scrolling

For large lists:

```tsx
import { useVirtualizer } from '@tanstack/react-virtual'

function LargeList({ items }) {
  const parentRef = useRef()

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
  })

  return (
    <div ref={parentRef} style={{ height: '400px', overflow: 'auto' }}>
      <div style={{ height: `${virtualizer.getTotalSize()}px` }}>
        {virtualizer.getVirtualItems().map((virtualItem) => (
          <div key={virtualItem.index}>
            {items[virtualItem.index].name}
          </div>
        ))}
      </div>
    </div>
  )
}
```

---

## Additional Resources

- **FastAPI Docs**: https://fastapi.tiangolo.com/
- **SQLModel Docs**: https://sqlmodel.tiangolo.com/
- **React Docs**: https://react.dev/
- **TanStack Router**: https://tanstack.com/router/
- **shadcn/ui**: https://ui.shadcn.com/
- **Celery Docs**: https://docs.celeryq.dev/

---

**Happy coding!** This guide should help you navigate the codebase and contribute effectively to the SEO Platform.
