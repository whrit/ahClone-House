# Implementation 01: Projects Module

> **PRD Reference:** Section 6.1 Projects
> **Priority:** P0 (Foundation for all other modules)
> **Sprint:** 0

---

## 1) Overview

The Projects module is the foundational entity that all other features attach to. A Project represents a website/domain being analyzed with its configuration, schedules, and linked integrations.

**Key Responsibilities:**
- CRUD operations for projects
- Settings management (crawl config, schedules, retention)
- Serving as foreign key parent for Audits, GSC, SERP, Links, etc.
- Dashboard showing last sync times for each module

---

## 2) Data Models

### 2.1 Project Model

```python
# backend/app/models/project.py
import uuid
from datetime import datetime
from sqlmodel import SQLModel, Field, Relationship, Column, JSON
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from .user import User
    from .audit import AuditRun
    from .gsc import GSCProperty


class ProjectSettings(SQLModel):
    """Embedded settings object (stored as JSON)"""
    # Crawl settings
    max_pages: int = 1000
    max_depth: int = 10
    crawl_concurrency: int = 5
    user_agent: str = "SEOPlatformBot/1.0"
    respect_robots_txt: bool = True
    include_subdomains: bool = False
    strip_query_params: bool = False
    include_patterns: list[str] = []  # Regex patterns to include
    exclude_patterns: list[str] = []  # Regex patterns to exclude

    # Rendering settings
    enable_js_rendering: bool = False
    js_render_mode: str = "hybrid"  # "hybrid", "always", "never"
    max_render_pages: int = 100
    render_timeout_ms: int = 30000

    # Schedule settings
    audit_frequency: str = "weekly"  # "off", "daily", "weekly", "monthly"
    gsc_sync_frequency: str = "daily"  # "off", "daily"
    serp_refresh_frequency: str = "daily"  # "off", "daily", "weekly"

    # Retention settings
    keep_audit_runs: int = 10  # Number of runs to retain
    keep_serp_days: int = 90  # Days of SERP snapshots to retain


class ProjectBase(SQLModel):
    name: str = Field(max_length=255, index=True)
    seed_url: str = Field(max_length=2048)
    description: str | None = Field(default=None, max_length=1000)


class Project(ProjectBase, table=True):
    __tablename__ = "projects"

    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    settings: dict = Field(default_factory=dict, sa_column=Column(JSON))
    created_by_id: uuid.UUID = Field(foreign_key="user.id", index=True)
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)

    # Status tracking (updated by background jobs)
    last_audit_at: datetime | None = None
    last_gsc_sync_at: datetime | None = None
    last_serp_refresh_at: datetime | None = None
    last_links_snapshot_at: datetime | None = None
    last_ppc_sync_at: datetime | None = None

    # Relationships
    created_by: "User" = Relationship(back_populates="projects")
    audit_runs: list["AuditRun"] = Relationship(back_populates="project")
    gsc_property: "GSCProperty" | None = Relationship(back_populates="project")


class ProjectCreate(ProjectBase):
    settings: ProjectSettings | None = None


class ProjectUpdate(SQLModel):
    name: str | None = None
    seed_url: str | None = None
    description: str | None = None
    settings: ProjectSettings | None = None


class ProjectPublic(ProjectBase):
    id: uuid.UUID
    settings: dict
    created_at: datetime
    updated_at: datetime
    last_audit_at: datetime | None
    last_gsc_sync_at: datetime | None
    last_serp_refresh_at: datetime | None
    last_links_snapshot_at: datetime | None
    last_ppc_sync_at: datetime | None


class ProjectsPublic(SQLModel):
    items: list[ProjectPublic]
    total: int
```

### 2.2 User Model Update

```python
# backend/app/models/user.py (add relationship)
class User(UserBase, table=True):
    # ... existing fields ...

    # Add relationship
    projects: list["Project"] = Relationship(back_populates="created_by")
```

---

## 3) API Routes

### 3.1 Route Definitions

```python
# backend/app/api/routes/projects.py
import uuid
from typing import Any
from fastapi import APIRouter, Depends, HTTPException, Query
from sqlmodel import select, func

from app.api.deps import CurrentUser, SessionDep
from app.models.project import (
    Project,
    ProjectCreate,
    ProjectUpdate,
    ProjectPublic,
    ProjectsPublic,
    ProjectSettings,
)

router = APIRouter(prefix="/projects", tags=["projects"])


@router.get("/", response_model=ProjectsPublic)
def list_projects(
    session: SessionDep,
    current_user: CurrentUser,
    skip: int = Query(default=0, ge=0),
    limit: int = Query(default=20, le=100),
) -> Any:
    """List all projects for the current user."""
    count_statement = (
        select(func.count())
        .select_from(Project)
        .where(Project.created_by_id == current_user.id)
    )
    total = session.exec(count_statement).one()

    statement = (
        select(Project)
        .where(Project.created_by_id == current_user.id)
        .offset(skip)
        .limit(limit)
        .order_by(Project.created_at.desc())
    )
    projects = session.exec(statement).all()

    return ProjectsPublic(items=projects, total=total)


@router.post("/", response_model=ProjectPublic)
def create_project(
    session: SessionDep,
    current_user: CurrentUser,
    project_in: ProjectCreate,
) -> Any:
    """Create a new project."""
    # Validate seed_url is a valid URL
    if not project_in.seed_url.startswith(("http://", "https://")):
        raise HTTPException(
            status_code=400,
            detail="seed_url must be a valid HTTP(S) URL",
        )

    settings = project_in.settings or ProjectSettings()
    project = Project(
        name=project_in.name,
        seed_url=project_in.seed_url,
        description=project_in.description,
        settings=settings.model_dump(),
        created_by_id=current_user.id,
    )
    session.add(project)
    session.commit()
    session.refresh(project)
    return project


@router.get("/{project_id}", response_model=ProjectPublic)
def get_project(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
) -> Any:
    """Get a specific project."""
    project = session.get(Project, project_id)
    if not project:
        raise HTTPException(status_code=404, detail="Project not found")
    if project.created_by_id != current_user.id:
        raise HTTPException(status_code=403, detail="Not authorized")
    return project


@router.patch("/{project_id}", response_model=ProjectPublic)
def update_project(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
    project_in: ProjectUpdate,
) -> Any:
    """Update a project."""
    project = session.get(Project, project_id)
    if not project:
        raise HTTPException(status_code=404, detail="Project not found")
    if project.created_by_id != current_user.id:
        raise HTTPException(status_code=403, detail="Not authorized")

    update_data = project_in.model_dump(exclude_unset=True)
    if "settings" in update_data and update_data["settings"]:
        # Merge with existing settings
        current_settings = project.settings or {}
        current_settings.update(update_data["settings"].model_dump())
        update_data["settings"] = current_settings

    for field, value in update_data.items():
        setattr(project, field, value)

    project.updated_at = datetime.utcnow()
    session.add(project)
    session.commit()
    session.refresh(project)
    return project


@router.delete("/{project_id}")
def delete_project(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
) -> Any:
    """Delete a project and all related data."""
    project = session.get(Project, project_id)
    if not project:
        raise HTTPException(status_code=404, detail="Project not found")
    if project.created_by_id != current_user.id:
        raise HTTPException(status_code=403, detail="Not authorized")

    session.delete(project)
    session.commit()
    return {"message": "Project deleted"}


@router.get("/{project_id}/overview")
def get_project_overview(
    session: SessionDep,
    current_user: CurrentUser,
    project_id: uuid.UUID,
) -> Any:
    """Get project overview with stats from each module."""
    project = session.get(Project, project_id)
    if not project:
        raise HTTPException(status_code=404, detail="Project not found")
    if project.created_by_id != current_user.id:
        raise HTTPException(status_code=403, detail="Not authorized")

    # TODO: Query stats from each module
    return {
        "project": ProjectPublic.model_validate(project),
        "stats": {
            "audits": {
                "total_runs": 0,
                "last_run": project.last_audit_at,
                "critical_issues": 0,
                "high_issues": 0,
            },
            "keywords": {
                "total_queries": 0,
                "last_sync": project.last_gsc_sync_at,
                "opportunities": 0,
            },
            "rank_tracking": {
                "keywords_tracked": 0,
                "last_refresh": project.last_serp_refresh_at,
            },
            "backlinks": {
                "referring_domains": 0,
                "total_backlinks": 0,
                "last_snapshot": project.last_links_snapshot_at,
            },
            "ppc": {
                "campaigns": 0,
                "last_sync": project.last_ppc_sync_at,
            },
        },
    }
```

### 3.2 Register Routes

```python
# backend/app/api/main.py (add to existing)
from app.api.routes import projects

api_router.include_router(projects.router)
```

---

## 4) Database Migration

### 4.1 Alembic Migration

```python
# backend/app/alembic/versions/xxxx_add_projects.py
"""Add projects table

Revision ID: xxxx
Revises: previous_revision
Create Date: 2024-xx-xx

"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

revision = "xxxx"
down_revision = "previous_revision"
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.create_table(
        "projects",
        sa.Column("id", postgresql.UUID(as_uuid=True), primary_key=True),
        sa.Column("name", sa.String(255), nullable=False, index=True),
        sa.Column("seed_url", sa.String(2048), nullable=False),
        sa.Column("description", sa.String(1000), nullable=True),
        sa.Column("settings", postgresql.JSONB, nullable=False, default={}),
        sa.Column(
            "created_by_id",
            postgresql.UUID(as_uuid=True),
            sa.ForeignKey("user.id", ondelete="CASCADE"),
            nullable=False,
            index=True,
        ),
        sa.Column("created_at", sa.DateTime, nullable=False),
        sa.Column("updated_at", sa.DateTime, nullable=False),
        sa.Column("last_audit_at", sa.DateTime, nullable=True),
        sa.Column("last_gsc_sync_at", sa.DateTime, nullable=True),
        sa.Column("last_serp_refresh_at", sa.DateTime, nullable=True),
        sa.Column("last_links_snapshot_at", sa.DateTime, nullable=True),
        sa.Column("last_ppc_sync_at", sa.DateTime, nullable=True),
    )


def downgrade() -> None:
    op.drop_table("projects")
```

---

## 5) Frontend Implementation

### 5.1 Project List Page

```tsx
// frontend/src/routes/_layout/projects/index.tsx
import { createFileRoute, Link } from "@tanstack/react-router"
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query"
import { ProjectsService } from "@/client"
import { Button } from "@/components/ui/button"
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table"
import { Badge } from "@/components/ui/badge"
import { Plus, ExternalLink, MoreHorizontal } from "lucide-react"
import { formatDistanceToNow } from "date-fns"
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from "@/components/ui/dropdown-menu"

export const Route = createFileRoute("/_layout/projects/")({
  component: ProjectsPage,
})

function ProjectsPage() {
  const { data, isLoading } = useQuery({
    queryKey: ["projects"],
    queryFn: () => ProjectsService.listProjects(),
  })

  if (isLoading) {
    return <ProjectsPageSkeleton />
  }

  return (
    <div className="container mx-auto py-6">
      <div className="flex items-center justify-between mb-6">
        <div>
          <h1 className="text-2xl font-bold">Projects</h1>
          <p className="text-muted-foreground">
            Manage your SEO projects and tracked domains
          </p>
        </div>
        <Button asChild>
          <Link to="/projects/new">
            <Plus className="mr-2 h-4 w-4" />
            New Project
          </Link>
        </Button>
      </div>

      {data?.items.length === 0 ? (
        <EmptyState />
      ) : (
        <div className="border rounded-lg">
          <Table>
            <TableHeader>
              <TableRow>
                <TableHead>Project</TableHead>
                <TableHead>Last Audit</TableHead>
                <TableHead>Last GSC Sync</TableHead>
                <TableHead>Keywords Tracked</TableHead>
                <TableHead className="w-[50px]"></TableHead>
              </TableRow>
            </TableHeader>
            <TableBody>
              {data?.items.map((project) => (
                <TableRow key={project.id}>
                  <TableCell>
                    <Link
                      to={`/projects/${project.id}`}
                      className="font-medium hover:underline"
                    >
                      {project.name}
                    </Link>
                    <div className="text-sm text-muted-foreground flex items-center gap-1">
                      {project.seed_url}
                      <ExternalLink className="h-3 w-3" />
                    </div>
                  </TableCell>
                  <TableCell>
                    {project.last_audit_at ? (
                      <Badge variant="outline">
                        {formatDistanceToNow(new Date(project.last_audit_at), {
                          addSuffix: true,
                        })}
                      </Badge>
                    ) : (
                      <Badge variant="secondary">Never</Badge>
                    )}
                  </TableCell>
                  <TableCell>
                    {project.last_gsc_sync_at ? (
                      formatDistanceToNow(new Date(project.last_gsc_sync_at), {
                        addSuffix: true,
                      })
                    ) : (
                      <span className="text-muted-foreground">Not connected</span>
                    )}
                  </TableCell>
                  <TableCell>—</TableCell>
                  <TableCell>
                    <ProjectRowActions projectId={project.id} />
                  </TableCell>
                </TableRow>
              ))}
            </TableBody>
          </Table>
        </div>
      )}
    </div>
  )
}

function EmptyState() {
  return (
    <div className="flex flex-col items-center justify-center py-12 border rounded-lg bg-muted/50">
      <h3 className="text-lg font-medium mb-2">No projects yet</h3>
      <p className="text-muted-foreground mb-4">
        Create your first project to start tracking SEO performance
      </p>
      <Button asChild>
        <Link to="/projects/new">
          <Plus className="mr-2 h-4 w-4" />
          Create Project
        </Link>
      </Button>
    </div>
  )
}

function ProjectRowActions({ projectId }: { projectId: string }) {
  const queryClient = useQueryClient()

  const deleteMutation = useMutation({
    mutationFn: () => ProjectsService.deleteProject({ projectId }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["projects"] })
    },
  })

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="ghost" size="icon">
          <MoreHorizontal className="h-4 w-4" />
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent align="end">
        <DropdownMenuItem asChild>
          <Link to={`/projects/${projectId}/settings`}>Settings</Link>
        </DropdownMenuItem>
        <DropdownMenuItem
          className="text-destructive"
          onClick={() => deleteMutation.mutate()}
        >
          Delete
        </DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  )
}

function ProjectsPageSkeleton() {
  return (
    <div className="container mx-auto py-6">
      <div className="h-8 w-48 bg-muted rounded mb-6" />
      <div className="border rounded-lg p-4">
        <div className="space-y-4">
          {[1, 2, 3].map((i) => (
            <div key={i} className="h-16 bg-muted rounded" />
          ))}
        </div>
      </div>
    </div>
  )
}
```

### 5.2 Create Project Page

```tsx
// frontend/src/routes/_layout/projects/new.tsx
import { createFileRoute, useNavigate } from "@tanstack/react-router"
import { useMutation, useQueryClient } from "@tanstack/react-query"
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { z } from "zod"
import { ProjectsService } from "@/client"
import { Button } from "@/components/ui/button"
import {
  Form,
  FormControl,
  FormDescription,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form"
import { Input } from "@/components/ui/input"
import { Textarea } from "@/components/ui/textarea"
import {
  Card,
  CardContent,
  CardDescription,
  CardHeader,
  CardTitle,
} from "@/components/ui/card"

export const Route = createFileRoute("/_layout/projects/new")({
  component: NewProjectPage,
})

const formSchema = z.object({
  name: z.string().min(1, "Name is required").max(255),
  seed_url: z.string().url("Must be a valid URL"),
  description: z.string().max(1000).optional(),
})

type FormValues = z.infer<typeof formSchema>

function NewProjectPage() {
  const navigate = useNavigate()
  const queryClient = useQueryClient()

  const form = useForm<FormValues>({
    resolver: zodResolver(formSchema),
    defaultValues: {
      name: "",
      seed_url: "",
      description: "",
    },
  })

  const mutation = useMutation({
    mutationFn: (values: FormValues) =>
      ProjectsService.createProject({ requestBody: values }),
    onSuccess: (project) => {
      queryClient.invalidateQueries({ queryKey: ["projects"] })
      navigate({ to: `/projects/${project.id}` })
    },
  })

  function onSubmit(values: FormValues) {
    mutation.mutate(values)
  }

  return (
    <div className="container max-w-2xl mx-auto py-6">
      <Card>
        <CardHeader>
          <CardTitle>Create Project</CardTitle>
          <CardDescription>
            Add a new website to track. You can configure crawl settings and
            connect integrations after creation.
          </CardDescription>
        </CardHeader>
        <CardContent>
          <Form {...form}>
            <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
              <FormField
                control={form.control}
                name="name"
                render={({ field }) => (
                  <FormItem>
                    <FormLabel>Project Name</FormLabel>
                    <FormControl>
                      <Input placeholder="My Website" {...field} />
                    </FormControl>
                    <FormDescription>
                      A friendly name to identify this project
                    </FormDescription>
                    <FormMessage />
                  </FormItem>
                )}
              />

              <FormField
                control={form.control}
                name="seed_url"
                render={({ field }) => (
                  <FormItem>
                    <FormLabel>Website URL</FormLabel>
                    <FormControl>
                      <Input placeholder="https://example.com" {...field} />
                    </FormControl>
                    <FormDescription>
                      The starting URL for crawling. Usually your homepage.
                    </FormDescription>
                    <FormMessage />
                  </FormItem>
                )}
              />

              <FormField
                control={form.control}
                name="description"
                render={({ field }) => (
                  <FormItem>
                    <FormLabel>Description (optional)</FormLabel>
                    <FormControl>
                      <Textarea
                        placeholder="Notes about this project..."
                        {...field}
                      />
                    </FormControl>
                    <FormMessage />
                  </FormItem>
                )}
              />

              <div className="flex gap-4">
                <Button type="submit" disabled={mutation.isPending}>
                  {mutation.isPending ? "Creating..." : "Create Project"}
                </Button>
                <Button
                  type="button"
                  variant="outline"
                  onClick={() => navigate({ to: "/projects" })}
                >
                  Cancel
                </Button>
              </div>
            </form>
          </Form>
        </CardContent>
      </Card>
    </div>
  )
}
```

### 5.3 Project Overview Page

```tsx
// frontend/src/routes/_layout/projects/$projectId/index.tsx
import { createFileRoute, Link } from "@tanstack/react-router"
import { useQuery } from "@tanstack/react-query"
import { ProjectsService } from "@/client"
import { Button } from "@/components/ui/button"
import {
  Card,
  CardContent,
  CardDescription,
  CardHeader,
  CardTitle,
} from "@/components/ui/card"
import { Badge } from "@/components/ui/badge"
import {
  Search,
  Link2,
  TrendingUp,
  FileSearch,
  DollarSign,
  Settings,
  Play,
} from "lucide-react"
import { formatDistanceToNow } from "date-fns"

export const Route = createFileRoute("/_layout/projects/$projectId/")({
  component: ProjectOverviewPage,
})

function ProjectOverviewPage() {
  const { projectId } = Route.useParams()

  const { data, isLoading } = useQuery({
    queryKey: ["projects", projectId, "overview"],
    queryFn: () => ProjectsService.getProjectOverview({ projectId }),
  })

  if (isLoading) {
    return <div>Loading...</div>
  }

  const { project, stats } = data

  return (
    <div className="container mx-auto py-6">
      {/* Header */}
      <div className="flex items-center justify-between mb-6">
        <div>
          <h1 className="text-2xl font-bold">{project.name}</h1>
          <p className="text-muted-foreground">{project.seed_url}</p>
        </div>
        <div className="flex gap-2">
          <Button variant="outline" asChild>
            <Link to={`/projects/${projectId}/settings`}>
              <Settings className="mr-2 h-4 w-4" />
              Settings
            </Link>
          </Button>
        </div>
      </div>

      {/* Module Cards */}
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
        {/* Audits */}
        <ModuleCard
          title="Site Audits"
          description="Technical SEO issues and fixes"
          icon={<FileSearch className="h-5 w-5" />}
          href={`/projects/${projectId}/audits`}
          stats={[
            { label: "Last run", value: formatLastSync(stats.audits.last_run) },
            { label: "Critical", value: stats.audits.critical_issues },
            { label: "High", value: stats.audits.high_issues },
          ]}
          actionLabel="Run Audit"
          actionHref={`/projects/${projectId}/audits`}
        />

        {/* Keywords */}
        <ModuleCard
          title="Keywords"
          description="Search Console performance"
          icon={<Search className="h-5 w-5" />}
          href={`/projects/${projectId}/keywords`}
          stats={[
            { label: "Last sync", value: formatLastSync(stats.keywords.last_sync) },
            { label: "Queries", value: stats.keywords.total_queries },
            { label: "Opportunities", value: stats.keywords.opportunities },
          ]}
          actionLabel={stats.keywords.last_sync ? "View" : "Connect GSC"}
          actionHref={`/projects/${projectId}/keywords`}
        />

        {/* Rank Tracking */}
        <ModuleCard
          title="Rank Tracking"
          description="Keyword position monitoring"
          icon={<TrendingUp className="h-5 w-5" />}
          href={`/projects/${projectId}/rank-tracker`}
          stats={[
            {
              label: "Last refresh",
              value: formatLastSync(stats.rank_tracking.last_refresh),
            },
            { label: "Keywords", value: stats.rank_tracking.keywords_tracked },
          ]}
          actionLabel="Add Keywords"
          actionHref={`/projects/${projectId}/rank-tracker`}
        />

        {/* Backlinks */}
        <ModuleCard
          title="Backlinks"
          description="Link profile and competitors"
          icon={<Link2 className="h-5 w-5" />}
          href={`/projects/${projectId}/links`}
          stats={[
            {
              label: "Last snapshot",
              value: formatLastSync(stats.backlinks.last_snapshot),
            },
            { label: "Ref. domains", value: stats.backlinks.referring_domains },
            { label: "Total links", value: stats.backlinks.total_backlinks },
          ]}
          actionLabel="Explore"
          actionHref={`/projects/${projectId}/links`}
        />

        {/* PPC */}
        <ModuleCard
          title="PPC"
          description="Paid search performance"
          icon={<DollarSign className="h-5 w-5" />}
          href={`/projects/${projectId}/ppc`}
          stats={[
            { label: "Last sync", value: formatLastSync(stats.ppc.last_sync) },
            { label: "Campaigns", value: stats.ppc.campaigns },
          ]}
          actionLabel={stats.ppc.last_sync ? "View" : "Connect Ads"}
          actionHref={`/projects/${projectId}/ppc`}
        />
      </div>
    </div>
  )
}

interface ModuleCardProps {
  title: string
  description: string
  icon: React.ReactNode
  href: string
  stats: Array<{ label: string; value: string | number }>
  actionLabel: string
  actionHref: string
}

function ModuleCard({
  title,
  description,
  icon,
  href,
  stats,
  actionLabel,
  actionHref,
}: ModuleCardProps) {
  return (
    <Card>
      <CardHeader className="flex flex-row items-center justify-between pb-2">
        <div className="flex items-center gap-2">
          {icon}
          <CardTitle className="text-lg">{title}</CardTitle>
        </div>
      </CardHeader>
      <CardContent>
        <CardDescription className="mb-4">{description}</CardDescription>
        <div className="grid grid-cols-3 gap-2 mb-4">
          {stats.map((stat) => (
            <div key={stat.label}>
              <div className="text-xs text-muted-foreground">{stat.label}</div>
              <div className="font-medium">{stat.value}</div>
            </div>
          ))}
        </div>
        <Button asChild className="w-full">
          <Link to={actionHref}>
            <Play className="mr-2 h-4 w-4" />
            {actionLabel}
          </Link>
        </Button>
      </CardContent>
    </Card>
  )
}

function formatLastSync(date: string | null): string {
  if (!date) return "Never"
  return formatDistanceToNow(new Date(date), { addSuffix: true })
}
```

### 5.4 Project Settings Page

```tsx
// frontend/src/routes/_layout/projects/$projectId/settings.tsx
import { createFileRoute, useNavigate } from "@tanstack/react-router"
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query"
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { z } from "zod"
import { ProjectsService } from "@/client"
import { Button } from "@/components/ui/button"
import {
  Form,
  FormControl,
  FormDescription,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form"
import { Input } from "@/components/ui/input"
import { Switch } from "@/components/ui/switch"
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select"
import {
  Card,
  CardContent,
  CardDescription,
  CardHeader,
  CardTitle,
} from "@/components/ui/card"
import { Tabs, TabsContent, TabsList, TabsTrigger } from "@/components/ui/tabs"
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
  AlertDialogTrigger,
} from "@/components/ui/alert-dialog"

export const Route = createFileRoute("/_layout/projects/$projectId/settings")({
  component: ProjectSettingsPage,
})

const settingsSchema = z.object({
  name: z.string().min(1).max(255),
  seed_url: z.string().url(),
  settings: z.object({
    max_pages: z.number().min(1).max(10000),
    max_depth: z.number().min(1).max(50),
    crawl_concurrency: z.number().min(1).max(20),
    respect_robots_txt: z.boolean(),
    enable_js_rendering: z.boolean(),
    js_render_mode: z.enum(["hybrid", "always", "never"]),
    max_render_pages: z.number().min(0).max(1000),
    audit_frequency: z.enum(["off", "daily", "weekly", "monthly"]),
    gsc_sync_frequency: z.enum(["off", "daily"]),
    serp_refresh_frequency: z.enum(["off", "daily", "weekly"]),
    keep_audit_runs: z.number().min(1).max(100),
    keep_serp_days: z.number().min(1).max(365),
  }),
})

type SettingsFormValues = z.infer<typeof settingsSchema>

function ProjectSettingsPage() {
  const { projectId } = Route.useParams()
  const navigate = useNavigate()
  const queryClient = useQueryClient()

  const { data: project, isLoading } = useQuery({
    queryKey: ["projects", projectId],
    queryFn: () => ProjectsService.getProject({ projectId }),
  })

  const updateMutation = useMutation({
    mutationFn: (values: SettingsFormValues) =>
      ProjectsService.updateProject({ projectId, requestBody: values }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["projects", projectId] })
    },
  })

  const deleteMutation = useMutation({
    mutationFn: () => ProjectsService.deleteProject({ projectId }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["projects"] })
      navigate({ to: "/projects" })
    },
  })

  const form = useForm<SettingsFormValues>({
    resolver: zodResolver(settingsSchema),
    values: project
      ? {
          name: project.name,
          seed_url: project.seed_url,
          settings: project.settings,
        }
      : undefined,
  })

  if (isLoading) {
    return <div>Loading...</div>
  }

  return (
    <div className="container max-w-4xl mx-auto py-6">
      <h1 className="text-2xl font-bold mb-6">Project Settings</h1>

      <Form {...form}>
        <form
          onSubmit={form.handleSubmit((values) => updateMutation.mutate(values))}
        >
          <Tabs defaultValue="general" className="space-y-6">
            <TabsList>
              <TabsTrigger value="general">General</TabsTrigger>
              <TabsTrigger value="crawl">Crawl Settings</TabsTrigger>
              <TabsTrigger value="rendering">JS Rendering</TabsTrigger>
              <TabsTrigger value="schedules">Schedules</TabsTrigger>
              <TabsTrigger value="danger">Danger Zone</TabsTrigger>
            </TabsList>

            {/* General Tab */}
            <TabsContent value="general">
              <Card>
                <CardHeader>
                  <CardTitle>General Settings</CardTitle>
                </CardHeader>
                <CardContent className="space-y-4">
                  <FormField
                    control={form.control}
                    name="name"
                    render={({ field }) => (
                      <FormItem>
                        <FormLabel>Project Name</FormLabel>
                        <FormControl>
                          <Input {...field} />
                        </FormControl>
                        <FormMessage />
                      </FormItem>
                    )}
                  />
                  <FormField
                    control={form.control}
                    name="seed_url"
                    render={({ field }) => (
                      <FormItem>
                        <FormLabel>Website URL</FormLabel>
                        <FormControl>
                          <Input {...field} />
                        </FormControl>
                        <FormMessage />
                      </FormItem>
                    )}
                  />
                </CardContent>
              </Card>
            </TabsContent>

            {/* Crawl Settings Tab */}
            <TabsContent value="crawl">
              <Card>
                <CardHeader>
                  <CardTitle>Crawl Configuration</CardTitle>
                  <CardDescription>
                    Control how the crawler navigates your site
                  </CardDescription>
                </CardHeader>
                <CardContent className="space-y-4">
                  <div className="grid grid-cols-2 gap-4">
                    <FormField
                      control={form.control}
                      name="settings.max_pages"
                      render={({ field }) => (
                        <FormItem>
                          <FormLabel>Max Pages</FormLabel>
                          <FormControl>
                            <Input
                              type="number"
                              {...field}
                              onChange={(e) =>
                                field.onChange(parseInt(e.target.value))
                              }
                            />
                          </FormControl>
                          <FormDescription>
                            Maximum pages to crawl per audit
                          </FormDescription>
                          <FormMessage />
                        </FormItem>
                      )}
                    />
                    <FormField
                      control={form.control}
                      name="settings.max_depth"
                      render={({ field }) => (
                        <FormItem>
                          <FormLabel>Max Depth</FormLabel>
                          <FormControl>
                            <Input
                              type="number"
                              {...field}
                              onChange={(e) =>
                                field.onChange(parseInt(e.target.value))
                              }
                            />
                          </FormControl>
                          <FormDescription>
                            Maximum link depth from seed URL
                          </FormDescription>
                          <FormMessage />
                        </FormItem>
                      )}
                    />
                  </div>
                  <FormField
                    control={form.control}
                    name="settings.respect_robots_txt"
                    render={({ field }) => (
                      <FormItem className="flex items-center justify-between rounded-lg border p-4">
                        <div>
                          <FormLabel>Respect robots.txt</FormLabel>
                          <FormDescription>
                            Follow robots.txt directives during crawl
                          </FormDescription>
                        </div>
                        <FormControl>
                          <Switch
                            checked={field.value}
                            onCheckedChange={field.onChange}
                          />
                        </FormControl>
                      </FormItem>
                    )}
                  />
                </CardContent>
              </Card>
            </TabsContent>

            {/* Rendering Tab */}
            <TabsContent value="rendering">
              <Card>
                <CardHeader>
                  <CardTitle>JavaScript Rendering</CardTitle>
                  <CardDescription>
                    Configure how JavaScript-heavy pages are processed
                  </CardDescription>
                </CardHeader>
                <CardContent className="space-y-4">
                  <FormField
                    control={form.control}
                    name="settings.enable_js_rendering"
                    render={({ field }) => (
                      <FormItem className="flex items-center justify-between rounded-lg border p-4">
                        <div>
                          <FormLabel>Enable JS Rendering</FormLabel>
                          <FormDescription>
                            Use Playwright to render JavaScript content
                          </FormDescription>
                        </div>
                        <FormControl>
                          <Switch
                            checked={field.value}
                            onCheckedChange={field.onChange}
                          />
                        </FormControl>
                      </FormItem>
                    )}
                  />
                  <FormField
                    control={form.control}
                    name="settings.js_render_mode"
                    render={({ field }) => (
                      <FormItem>
                        <FormLabel>Render Mode</FormLabel>
                        <Select
                          onValueChange={field.onChange}
                          value={field.value}
                        >
                          <FormControl>
                            <SelectTrigger>
                              <SelectValue />
                            </SelectTrigger>
                          </FormControl>
                          <SelectContent>
                            <SelectItem value="hybrid">
                              Hybrid (render when needed)
                            </SelectItem>
                            <SelectItem value="always">Always render</SelectItem>
                            <SelectItem value="never">Never render</SelectItem>
                          </SelectContent>
                        </Select>
                        <FormMessage />
                      </FormItem>
                    )}
                  />
                  <FormField
                    control={form.control}
                    name="settings.max_render_pages"
                    render={({ field }) => (
                      <FormItem>
                        <FormLabel>Max Rendered Pages</FormLabel>
                        <FormControl>
                          <Input
                            type="number"
                            {...field}
                            onChange={(e) =>
                              field.onChange(parseInt(e.target.value))
                            }
                          />
                        </FormControl>
                        <FormDescription>
                          Maximum pages to render per audit (budget control)
                        </FormDescription>
                        <FormMessage />
                      </FormItem>
                    )}
                  />
                </CardContent>
              </Card>
            </TabsContent>

            {/* Schedules Tab */}
            <TabsContent value="schedules">
              <Card>
                <CardHeader>
                  <CardTitle>Automated Schedules</CardTitle>
                  <CardDescription>
                    Configure when jobs run automatically
                  </CardDescription>
                </CardHeader>
                <CardContent className="space-y-4">
                  <FormField
                    control={form.control}
                    name="settings.audit_frequency"
                    render={({ field }) => (
                      <FormItem>
                        <FormLabel>Audit Frequency</FormLabel>
                        <Select
                          onValueChange={field.onChange}
                          value={field.value}
                        >
                          <FormControl>
                            <SelectTrigger>
                              <SelectValue />
                            </SelectTrigger>
                          </FormControl>
                          <SelectContent>
                            <SelectItem value="off">Off</SelectItem>
                            <SelectItem value="daily">Daily</SelectItem>
                            <SelectItem value="weekly">Weekly</SelectItem>
                            <SelectItem value="monthly">Monthly</SelectItem>
                          </SelectContent>
                        </Select>
                        <FormMessage />
                      </FormItem>
                    )}
                  />
                  <FormField
                    control={form.control}
                    name="settings.gsc_sync_frequency"
                    render={({ field }) => (
                      <FormItem>
                        <FormLabel>GSC Sync Frequency</FormLabel>
                        <Select
                          onValueChange={field.onChange}
                          value={field.value}
                        >
                          <FormControl>
                            <SelectTrigger>
                              <SelectValue />
                            </SelectTrigger>
                          </FormControl>
                          <SelectContent>
                            <SelectItem value="off">Off</SelectItem>
                            <SelectItem value="daily">Daily</SelectItem>
                          </SelectContent>
                        </Select>
                        <FormMessage />
                      </FormItem>
                    )}
                  />
                </CardContent>
              </Card>
            </TabsContent>

            {/* Danger Zone Tab */}
            <TabsContent value="danger">
              <Card className="border-destructive">
                <CardHeader>
                  <CardTitle className="text-destructive">Danger Zone</CardTitle>
                  <CardDescription>
                    Irreversible actions that affect your project
                  </CardDescription>
                </CardHeader>
                <CardContent>
                  <AlertDialog>
                    <AlertDialogTrigger asChild>
                      <Button variant="destructive">Delete Project</Button>
                    </AlertDialogTrigger>
                    <AlertDialogContent>
                      <AlertDialogHeader>
                        <AlertDialogTitle>Delete Project?</AlertDialogTitle>
                        <AlertDialogDescription>
                          This will permanently delete the project and all
                          associated data including audits, keywords, and
                          backlink data. This action cannot be undone.
                        </AlertDialogDescription>
                      </AlertDialogHeader>
                      <AlertDialogFooter>
                        <AlertDialogCancel>Cancel</AlertDialogCancel>
                        <AlertDialogAction
                          className="bg-destructive"
                          onClick={() => deleteMutation.mutate()}
                        >
                          Delete
                        </AlertDialogAction>
                      </AlertDialogFooter>
                    </AlertDialogContent>
                  </AlertDialog>
                </CardContent>
              </Card>
            </TabsContent>
          </Tabs>

          <div className="flex justify-end mt-6">
            <Button type="submit" disabled={updateMutation.isPending}>
              {updateMutation.isPending ? "Saving..." : "Save Changes"}
            </Button>
          </div>
        </form>
      </Form>
    </div>
  )
}
```

---

## 6) Unit Tests

### 6.1 Backend Tests

```python
# backend/tests/api/test_projects.py
import pytest
from fastapi.testclient import TestClient
from sqlmodel import Session

from app.models.project import Project, ProjectCreate


def test_create_project(client: TestClient, normal_user_token_headers: dict):
    data = {
        "name": "Test Project",
        "seed_url": "https://example.com",
        "description": "A test project",
    }
    response = client.post(
        "/api/v1/projects/",
        headers=normal_user_token_headers,
        json=data,
    )
    assert response.status_code == 200
    content = response.json()
    assert content["name"] == data["name"]
    assert content["seed_url"] == data["seed_url"]
    assert "id" in content
    assert "settings" in content


def test_create_project_invalid_url(client: TestClient, normal_user_token_headers: dict):
    data = {
        "name": "Test Project",
        "seed_url": "not-a-valid-url",
    }
    response = client.post(
        "/api/v1/projects/",
        headers=normal_user_token_headers,
        json=data,
    )
    assert response.status_code == 400


def test_list_projects(
    client: TestClient, normal_user_token_headers: dict, db: Session
):
    # Create a project first
    response = client.post(
        "/api/v1/projects/",
        headers=normal_user_token_headers,
        json={"name": "Project 1", "seed_url": "https://example.com"},
    )
    assert response.status_code == 200

    # List projects
    response = client.get(
        "/api/v1/projects/",
        headers=normal_user_token_headers,
    )
    assert response.status_code == 200
    content = response.json()
    assert content["total"] >= 1
    assert len(content["items"]) >= 1


def test_get_project(client: TestClient, normal_user_token_headers: dict):
    # Create
    create_response = client.post(
        "/api/v1/projects/",
        headers=normal_user_token_headers,
        json={"name": "Get Test", "seed_url": "https://example.com"},
    )
    project_id = create_response.json()["id"]

    # Get
    response = client.get(
        f"/api/v1/projects/{project_id}",
        headers=normal_user_token_headers,
    )
    assert response.status_code == 200
    assert response.json()["id"] == project_id


def test_update_project(client: TestClient, normal_user_token_headers: dict):
    # Create
    create_response = client.post(
        "/api/v1/projects/",
        headers=normal_user_token_headers,
        json={"name": "Update Test", "seed_url": "https://example.com"},
    )
    project_id = create_response.json()["id"]

    # Update
    response = client.patch(
        f"/api/v1/projects/{project_id}",
        headers=normal_user_token_headers,
        json={"name": "Updated Name"},
    )
    assert response.status_code == 200
    assert response.json()["name"] == "Updated Name"


def test_delete_project(client: TestClient, normal_user_token_headers: dict):
    # Create
    create_response = client.post(
        "/api/v1/projects/",
        headers=normal_user_token_headers,
        json={"name": "Delete Test", "seed_url": "https://example.com"},
    )
    project_id = create_response.json()["id"]

    # Delete
    response = client.delete(
        f"/api/v1/projects/{project_id}",
        headers=normal_user_token_headers,
    )
    assert response.status_code == 200

    # Verify deleted
    response = client.get(
        f"/api/v1/projects/{project_id}",
        headers=normal_user_token_headers,
    )
    assert response.status_code == 404


def test_project_authorization(
    client: TestClient,
    normal_user_token_headers: dict,
    superuser_token_headers: dict,
):
    # Create project as normal user
    create_response = client.post(
        "/api/v1/projects/",
        headers=normal_user_token_headers,
        json={"name": "Auth Test", "seed_url": "https://example.com"},
    )
    project_id = create_response.json()["id"]

    # Try to access as different user (superuser here, but concept applies)
    # In real test, use a different normal user
    response = client.get(
        f"/api/v1/projects/{project_id}",
        headers=normal_user_token_headers,  # Same user - should work
    )
    assert response.status_code == 200
```

---

## 7) Acceptance Criteria

From PRD Section 6.1:

- [x] **CRUD Projects** - Create, read, update, delete operations
- [x] **Settings storage** - JSON settings field with typed schema
- [x] **Seed URL validation** - Must be valid HTTP(S) URL
- [x] **User authorization** - Users can only access their own projects
- [x] **Status tracking** - Last audit, GSC sync, SERP refresh, links snapshot timestamps
- [x] **Project list shows sync times** - Display in list and overview pages

---

## 8) Implementation Checklist

```
[ ] Create backend/app/models/project.py with all models
[ ] Create backend/app/api/routes/projects.py with CRUD endpoints
[ ] Register routes in backend/app/api/main.py
[ ] Create Alembic migration for projects table
[ ] Run migration: alembic upgrade head
[ ] Add User.projects relationship to user model
[ ] Create frontend routes for projects (list, new, overview, settings)
[ ] Generate TypeScript client: npm run generate-client
[ ] Write unit tests for API endpoints
[ ] Write E2E tests for frontend flows
```

---

## 9) Dependencies

**Before this module:**
- None (first module to implement)

**After this module:**
- All other modules depend on Project as parent entity
- Celery infrastructure (Sprint 0, parallel track)

**External dependencies:**
- None (uses existing stack)
