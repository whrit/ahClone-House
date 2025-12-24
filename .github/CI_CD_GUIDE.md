# Sprint 6: CI/CD Pipeline Documentation

This document describes the automated CI/CD workflows and processes for the SEO platform.

## Overview

The CI/CD system is built on GitHub Actions and provides comprehensive automation for:

- Continuous Integration (CI) testing and quality checks
- Continuous Deployment (CD) to staging and production
- Security scanning and dependency management
- Code quality analysis and metrics
- Automated dependency updates via Dependabot

## Workflows

### 1. CI Workflow (`.github/workflows/ci.yml`)

**Trigger**: Push to `main` or `develop`, and Pull Requests

**Purpose**: Comprehensive code quality and testing on every change

#### Jobs:

1. **backend-lint**: Linting and type checking for Python code
   - Runs Ruff linting and formatting checks
   - Runs mypy type checking
   - Reports issues in the PR

2. **backend-test**: Backend testing with database and cache services
   - Spins up PostgreSQL 17 and Redis 7 services
   - Runs pytest with coverage reporting
   - Requires 90% minimum code coverage
   - Uploads coverage to Codecov

3. **frontend-lint**: Linting for TypeScript/React code
   - Runs Biome linting and formatting checks
   - Ensures consistent code style

4. **frontend-build**: Frontend compilation and build
   - Runs TypeScript type checking
   - Builds the frontend production bundle
   - Uploads build artifacts

5. **frontend-test**: Frontend E2E and unit tests
   - Installs Playwright browsers
   - Runs Playwright test suite
   - Uploads test reports and videos

6. **security-scan**: Security vulnerability scanning
   - Trivy filesystem scan for vulnerable dependencies
   - npm audit for frontend dependencies
   - Reports critical and high severity issues

7. **dependency-check**: Checks for outdated dependencies
   - Lists outdated backend packages
   - Lists outdated frontend packages
   - Informational only, does not fail the build

#### Caching Strategy

- Python dependencies: Cached using uv cache
- npm dependencies: Cached using npm ci
- Build artifacts: Retained for 1 day

#### Failed Job Handling

- Lint failures block PRs
- Test failures block PRs
- Security failures block PRs (CRITICAL and HIGH)
- Dependency outdated warnings are informational only

### 2. Deploy Workflow (`.github/workflows/deploy.yml`)

**Trigger**: Push tags `v*` for production, push to `main` for staging, or manual `workflow_dispatch`

**Purpose**: Automated deployment pipeline with environment-specific logic

#### Jobs:

1. **plan-deployment**: Determines deployment environment
   - Parses git refs and tags
   - Outputs environment (staging/production) and version
   - Validates version format for production (vX.Y.Z)

2. **build-images**: Docker image building and pushing
   - Sets up Docker Buildx for multi-platform builds
   - Builds backend and frontend Docker images
   - Pushes to Docker registry with version tags
   - Uses registry caching for faster builds

3. **deploy-staging**: Deploy to staging environment
   - Runs after successful build
   - Deploys on pushes to `main`
   - Runs smoke tests
   - Posts deployment status

4. **deploy-production**: Deploy to production environment
   - Runs after successful build
   - Requires version tag in format `vX.Y.Z`
   - Creates GitHub release
   - Runs health checks post-deployment
   - Posts deployment notifications

#### Deployment Strategy

- **Staging**: Automatic deployment on merge to main
- **Production**: Manual tag-triggered deployment
- **Manual Override**: Use workflow dispatch to deploy to either environment

#### Environment Configuration

See "Secrets Configuration" section below for required environment variables.

### 3. Code Quality Workflow (`.github/workflows/code-quality.yml`)

**Trigger**: Push to main/develop with code changes, Pull Requests

**Purpose**: In-depth code quality analysis and reporting

#### Jobs:

1. **backend-quality**: Python code quality analysis
   - Ruff linting with GitHub output format
   - Complexity checks
   - Type checking with mypy

2. **frontend-quality**: TypeScript code quality analysis
   - Biome linting and formatting
   - TypeScript compilation
   - Quality metrics reporting

3. **duplicate-detection**: Finds duplicate code patterns
   - Scans for code duplication
   - Patterns reported in PR

4. **code-metrics**: Calculates code statistics
   - Line count by language
   - File count metrics
   - Reports to job summary

### 4. Security Workflow (`.github/workflows/security.yml`)

**Trigger**: Push to main/develop, Pull Requests, Weekly schedule

**Purpose**: Security scanning and vulnerability detection

#### Jobs:

1. **sast**: Static Application Security Testing
   - Trivy filesystem scan
   - Produces SARIF reports
   - Uploads to GitHub Security tab

2. **dependency-scan-backend**: Python vulnerability scanning
   - Checks for known vulnerabilities
   - Lists dependencies

3. **dependency-scan-frontend**: npm vulnerability scanning
   - npm audit for security issues
   - License compliance check

4. **secret-scan**: Detects hardcoded secrets
   - TruffleHog secret detection
   - Verified secrets only
   - Full repo scan on all commits

5. **code-security**: Application code security checks
   - Ruff security rules (S prefix)
   - Manual pattern matching for secrets
   - Reports issues

6. **container-scan**: Docker image vulnerability scanning
   - Scans built Docker images
   - Reports vulnerabilities

#### Security Policies

- CRITICAL and HIGH severity issues: Must be fixed before merge
- MEDIUM severity: Should be fixed, but not blocking
- LOW severity: Informational only

### 5. Dependabot Configuration (`.github/dependabot.yml`)

**Purpose**: Automated dependency updates

#### Update Schedule:

- **GitHub Actions**: Daily at 03:00 UTC
- **Backend (Python)**: Weekly Monday at 03:00 UTC
  - Grouped: Development, Testing, Core dependencies
- **Frontend (npm)**: Weekly Monday at 04:00 UTC
  - Grouped: Development, Testing, UI, Core dependencies
- **Docker**: Weekly Tuesday at 03:00 UTC
- **Docker Compose**: Weekly Tuesday at 03:00 UTC

#### Features:

- Automatic PR creation for updates
- Grouped by dependency type
- Auto-merge enabled for patch versions (can be configured)
- Labeled for easy filtering
- Assigned to @beckett for review

## Secrets Configuration

The following secrets must be configured in your GitHub repository settings:

### Docker Registry (for deployment)

```
DOCKER_REGISTRY     - Docker registry URL
DOCKER_USERNAME     - Docker registry username
DOCKER_PASSWORD     - Docker registry password or token
```

### Staging Environment

```
STAGING_DEPLOY_KEY  - SSH key for staging deployment
STAGING_DEPLOY_HOST - Staging server hostname
STAGING_DEPLOY_USER - SSH user for staging server
STAGING_URL         - URL of staging environment
```

### Production Environment

```
PRODUCTION_DEPLOY_KEY  - SSH key for production deployment
PRODUCTION_DEPLOY_HOST - Production server hostname
PRODUCTION_DEPLOY_USER - SSH user for production server
PRODUCTION_URL         - URL of production environment
```

### Code Quality

```
CODECOV_TOKEN  - Codecov.io token for coverage reports
```

## Environment Variables

### Backend (via secrets)

```
DATABASE_URL           - PostgreSQL connection string
REDIS_URL             - Redis connection string
GOOGLE_CLIENT_ID      - Google OAuth client ID
GOOGLE_CLIENT_SECRET  - Google OAuth client secret
TOKEN_ENCRYPTION_KEY  - Fernet encryption key for tokens
```

### Frontend (via secrets)

```
VITE_API_URL          - Backend API URL
VITE_ENVIRONMENT      - Environment name (staging/production)
```

## Pull Request Requirements

All PRs must pass before merging:

- [x] Lint checks pass (backend and frontend)
- [x] Type checking passes
- [x] All tests pass (backend coverage >= 90%, frontend E2E)
- [x] Security scans pass (no CRITICAL/HIGH issues)
- [x] No new warnings introduced

### PR Checklist Template

Use the `.github/pull_request_template.md` template to ensure all checks are complete.

## Deployment Process

### Staging Deployment

1. Create a PR from feature branch to `develop`
2. All CI checks must pass
3. Merge to `develop`
4. Merge `develop` to `main`
5. Automatic deployment to staging on push to main

### Production Deployment

1. Ensure all changes are in `main`
2. Create a git tag: `git tag -a v1.2.3 -m "Release v1.2.3"`
3. Push tag: `git push origin v1.2.3`
4. Deploy workflow automatically triggers
5. Docker images built and pushed
6. Production deployment executed
7. Health checks run
8. GitHub release created

### Manual Override

To manually trigger a deployment:

1. Go to Actions -> Deploy
2. Click "Run workflow"
3. Select environment (staging/production)
4. Click "Run workflow"

## Local Development Setup

### Backend

```bash
cd backend

# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Install dependencies
uv sync

# Run linting
uv run ruff check .

# Run type checking
uv run mypy app --ignore-missing-imports

# Run tests
uv run pytest -v
```

### Frontend

```bash
cd frontend

# Install dependencies
npm ci

# Run linting
npx biome check --write ./

# Run type checking
npx tsc --noEmit

# Build
npm run build

# Run tests
npm run test
```

## Troubleshooting

### CI Pipeline Failures

1. **Lint Failures**: Run `uv run ruff check . --fix` (backend) or `npx biome check --write ./` (frontend)
2. **Type Failures**: Check mypy/tsc output, fix type annotations
3. **Test Failures**: Run tests locally, debug, commit fix
4. **Coverage Failures**: Add tests to reach 90% coverage threshold

### Deployment Issues

1. **Docker Build Failures**: Check Dockerfile syntax, ensure all dependencies are listed
2. **Deployment Script Failures**: SSH into server, check logs, fix infrastructure issues
3. **Health Check Failures**: Verify service is running, check logs on server

### Security Scan Issues

1. **Vulnerability Found**: Check CVE details, patch dependency, or document exception
2. **Secret Detected**: Remove the secret, rotate if it was leaked, commit clean version
3. **Policy Violation**: Update configuration or document exception

## Monitoring and Alerting

### GitHub Notifications

- Workflow failures are reported in PR comments
- Push-based workflow failures notify repository owner
- Check GitHub Actions dashboard for detailed logs

### CI Status Badge

Add to README:

```markdown
![CI Status](https://github.com/ahClone/workflows/CI/badge.svg)
```

## Best Practices

1. **Keep workflows focused**: Each workflow has a single responsibility
2. **Use concurrency control**: Prevents duplicate runs and saves resources
3. **Cache dependencies**: Reduces build time significantly
4. **Fail fast**: Lint before tests, quick checks before slow ones
5. **Meaningful commit messages**: Required for clear history
6. **Review security alerts**: Address vulnerabilities promptly
7. **Monitor build metrics**: Track CI performance over time

## Maintenance

### Regular Updates

- Review and update action versions quarterly
- Update Python and Node versions as needed
- Review security policies quarterly
- Archive completed releases monthly

### Performance Optimization

- Monitor job execution times
- Optimize cache strategies
- Consider parallel job execution
- Use self-hosted runners for large builds (if available)

## References

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Ruff Documentation](https://docs.astral.sh/ruff/)
- [Biome Documentation](https://biomejs.dev/)
- [mypy Documentation](https://mypy.readthedocs.io/)
- [Playwright Documentation](https://playwright.dev/)
- [Trivy Documentation](https://aquasecurity.github.io/trivy/)

## Support

For issues or questions about the CI/CD pipeline:

1. Check the workflow logs in GitHub Actions
2. Review this documentation
3. Check the troubleshooting section
4. Create an issue with the `ci-cd` label
