# GitHub Configuration and CI/CD Documentation

Welcome to the Sprint 6 Hardening CI/CD implementation for the SEO Platform. This directory contains all GitHub Actions workflows, configuration files, and documentation.

## Quick Navigation

### For First-Time Users
Start here: [QUICK_START.md](./QUICK_START.md) (5-10 minutes)

### For Developers
- [CONTRIBUTING.md](./CONTRIBUTING.md) - Development guidelines and standards
- [CI_CD_GUIDE.md](./CI_CD_GUIDE.md) - Complete CI/CD operations guide

### For DevOps/Infrastructure
- [IMPLEMENTATION_SUMMARY.md](./IMPLEMENTATION_SUMMARY.md) - Technical details
- [SPRINT6_IMPLEMENTATION.md](./SPRINT6_IMPLEMENTATION.md) - Implementation report
- [CI_CD_GUIDE.md](./CI_CD_GUIDE.md) - Operational procedures

### For Project Managers
- [SPRINT6_IMPLEMENTATION.md](./SPRINT6_IMPLEMENTATION.md) - Project completion report

## Directory Structure

```
.github/
├── workflows/                    # GitHub Actions workflows
│   ├── ci.yml                   # Main CI pipeline (280 lines)
│   ├── deploy.yml               # Deployment pipeline (279 lines)
│   ├── code-quality.yml         # Code quality checks (159 lines)
│   ├── security.yml             # Security scanning (232 lines)
│   └── [other workflows...]
│
├── ISSUE_TEMPLATE/              # Issue templates
│   ├── bug_report.md            # Bug report template
│   ├── feature_request.md       # Feature request template
│   └── config.yml               # Template configuration
│
├── QUICK_START.md               # 5-minute quick reference
├── CI_CD_GUIDE.md               # Comprehensive operations guide (427 lines)
├── CONTRIBUTING.md              # Developer guidelines (520 lines)
├── IMPLEMENTATION_SUMMARY.md    # Implementation overview (350+ lines)
├── SPRINT6_IMPLEMENTATION.md    # Implementation report (380+ lines)
├── README.md                    # This file
├── pull_request_template.md     # PR submission template
└── [configuration files...]
```

## Key Files Overview

### Workflows (in `.github/workflows/`)

#### ci.yml - Continuous Integration
- **Purpose**: Run tests and checks on every PR and push
- **Trigger**: Push to main/develop, Pull requests
- **Duration**: ~10-15 minutes
- **Key Jobs**:
  - Backend: Linting, type checking, testing (pytest with 90% coverage)
  - Frontend: Linting, type checking, building, E2E testing
  - Security: Vulnerability scanning, secret detection
  - Dependencies: Outdated package checking
- **Status**: All checks must pass before merge

#### deploy.yml - Continuous Deployment
- **Purpose**: Build and deploy Docker images to staging and production
- **Trigger**: Push to main (staging), Version tags (production), Manual dispatch
- **Duration**: ~10-20 minutes
- **Key Jobs**:
  - Plan: Determine environment based on ref
  - Build: Docker image building and registry push
  - Deploy: Environment-specific deployment
  - Health: Post-deployment health checks
- **Environments**: staging and production

#### code-quality.yml - Code Quality Analysis
- **Purpose**: Detailed code quality metrics and analysis
- **Trigger**: Push to main/develop with code changes
- **Key Jobs**:
  - Backend quality analysis (Ruff, mypy)
  - Frontend quality analysis (Biome, tsc)
  - Duplicate code detection
  - Code metrics calculation

#### security.yml - Security Scanning
- **Purpose**: Comprehensive security vulnerability detection
- **Trigger**: Push, PR, Weekly schedule (Sundays)
- **Key Jobs**:
  - SAST with Trivy
  - Dependency scanning (Python and npm)
  - Secret detection with TruffleHog
  - Container image scanning
  - Code security checks

### Documentation Files

#### QUICK_START.md
**Best for**: Everyone, especially new team members
- 5-10 minute setup and command reference
- Common tasks and workflows
- Quick troubleshooting
- Read this first!

#### CI_CD_GUIDE.md
**Best for**: Developers and operations
- Comprehensive workflow documentation
- Detailed job descriptions
- Configuration and secrets setup
- Troubleshooting guide
- Best practices and monitoring

#### CONTRIBUTING.md
**Best for**: All developers
- Development environment setup
- Code style guides (Python and TypeScript)
- Testing guidelines
- Commit message standards
- PR requirements and process

#### IMPLEMENTATION_SUMMARY.md
**Best for**: Technical leads and operations
- Overview of all implementations
- Feature descriptions
- Configuration requirements
- Integration points
- Next steps and recommendations

#### SPRINT6_IMPLEMENTATION.md
**Best for**: Project stakeholders
- Implementation scope and status
- Success criteria and validation
- Performance metrics
- Deployment procedures
- Support resources

### Configuration Files

#### dependabot.yml
- **Purpose**: Automated dependency updates
- **Updates**: GitHub Actions (daily), Python/npm (weekly), Docker (weekly)
- **Features**: Grouped updates, conventional commits, custom labels

#### pull_request_template.md
- **Purpose**: Standardize PR submissions
- **Contents**: Description template, checklist, testing requirements

#### ISSUE_TEMPLATE/
- **bug_report.md**: Structured bug reporting
- **feature_request.md**: Structured feature proposals
- **config.yml**: Template configuration and contact links

## Getting Started

### Step 1: Read Documentation (5 minutes)
1. Read [QUICK_START.md](./QUICK_START.md)
2. Understand the workflow structure above
3. Review relevant documentation for your role

### Step 2: Configure Secrets (if infrastructure role)
Set these in GitHub Settings > Secrets:
- Docker registry credentials (3)
- Staging deployment secrets (4)
- Production deployment secrets (4)
- Codecov token (optional)

### Step 3: Test Locally (5 minutes)
```bash
# Backend
cd backend && uv sync && uv run ruff check . && uv run pytest -v

# Frontend
cd frontend && npm ci && npx biome check . && npm run build
```

### Step 4: Create Test PR
1. Create feature branch
2. Make small change
3. Push and create PR
4. Verify CI checks pass
5. Review logs if needed

### Step 5: Deploy
1. **To Staging**: Merge PR to main
2. **To Production**: Create version tag and push

## Common Tasks

### Run CI Checks Locally
```bash
# Backend
cd backend
uv sync
uv run ruff check . --fix
uv run ruff format .
uv run mypy app --ignore-missing-imports
uv run pytest -v --cov=app

# Frontend
cd frontend
npm ci
npx biome check --write ./
npx tsc --noEmit
npm run build
```

### Create a Production Release
```bash
git tag -a v1.2.3 -m "Release v1.2.3"
git push origin v1.2.3
# CI automatically builds, tests, and deploys
```

### Rerun a Workflow
Go to GitHub Actions tab, click workflow run, click "Re-run failed jobs" or "Re-run all jobs"

### View Workflow Logs
1. Go to GitHub Actions tab
2. Click workflow run name
3. Click failing job
4. Expand steps to see detailed logs

## Workflows Summary

| Workflow | Trigger | Duration | Purpose |
|----------|---------|----------|---------|
| CI | PR, push main/develop | 10-15 min | Test and check code quality |
| Deploy | Tag, push main, manual | 10-20 min | Build and deploy to environments |
| Code Quality | Push with changes | 5-10 min | Detailed quality analysis |
| Security | PR, push, weekly | 5-10 min | Vulnerability scanning |

## Troubleshooting

### My CI Failed, What Do I Do?
1. Click the failing check in PR
2. Read the error message
3. Fix the issue locally using commands above
4. Commit and push again
5. Check [CI_CD_GUIDE.md](./CI_CD_GUIDE.md) troubleshooting section

### How Do I Update a Dependency?
```bash
# Backend
cd backend && uv update package-name && git commit -am "chore: update package-name"

# Frontend
cd frontend && npm update package-name && git commit -am "chore: update package-name"
```

### My Deployment Failed
1. Check deployment logs in GitHub Actions
2. SSH into server and check service status
3. Review [CI_CD_GUIDE.md](./CI_CD_GUIDE.md) deployment section
4. Create issue if needed with error details

### How Do I Rollback?
Deploy the previous tag:
```bash
git tag -a v1.2.2 -m "Rollback to v1.2.2"
git push origin v1.2.2
```

## Support Resources

### Documentation Files
- **Quick Help**: [QUICK_START.md](./QUICK_START.md)
- **Full Guide**: [CI_CD_GUIDE.md](./CI_CD_GUIDE.md)
- **Contributing**: [CONTRIBUTING.md](./CONTRIBUTING.md)
- **Implementation**: [IMPLEMENTATION_SUMMARY.md](./IMPLEMENTATION_SUMMARY.md)

### External Resources
- [GitHub Actions Docs](https://docs.github.com/en/actions)
- [Ruff Documentation](https://docs.astral.sh/ruff/)
- [Biome Documentation](https://biomejs.dev/)
- [Docker Documentation](https://docs.docker.com/)

### Need Help?
1. Check the relevant documentation file
2. Review GitHub Actions logs for detailed errors
3. Search existing issues
4. Create a new issue with error details

## Key Metrics

- **Total Files Created**: 10
- **Total Files Modified**: 2
- **Workflow Code**: 950 lines
- **Documentation**: 1,928 lines
- **Implementation Status**: Complete
- **Production Ready**: Yes

## Implementation Status

✓ CI Pipeline - Complete
✓ Deploy Pipeline - Complete
✓ Code Quality - Complete
✓ Security Scanning - Complete
✓ Dependency Management - Complete
✓ Documentation - Complete
✓ Templates - Complete
✓ Team Ready - Yes

**Sprint 6: Hardening is complete and ready for production use.**

---

Last Updated: December 24, 2025
Status: Production Ready
For Questions: See documentation files above
