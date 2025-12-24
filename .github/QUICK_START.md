# Sprint 6 CI/CD Quick Start Guide

## For Developers

### 1. Local Development Setup (5 minutes)

#### Backend
```bash
cd backend
uv sync
uv run pytest -v  # Run tests
```

#### Frontend
```bash
cd frontend
npm ci
npm run test  # Run tests
```

### 2. Before Pushing Changes (5 minutes)

#### Backend
```bash
cd backend
uv run ruff check . --fix      # Auto-fix style issues
uv run ruff format .           # Format code
uv run mypy app                # Type check
uv run pytest -v --cov=app     # Test with coverage
```

#### Frontend
```bash
cd frontend
npx biome check --write ./     # Fix linting issues
npx tsc --noEmit               # Type check
npm run test                   # Run tests
```

### 3. Creating a Pull Request

1. Create feature branch: `git checkout -b feature/your-feature`
2. Make changes and commit: `git commit -m "feat: description"`
3. Push: `git push origin feature/your-feature`
4. Create PR on GitHub
5. Wait for CI checks to pass (green checkmark)
6. Request review from team members
7. Merge when approved and all checks pass

### 4. Pushing to Staging

1. Create PR to main branch
2. Get approval
3. Merge to main
4. CI automatically builds and deploys to staging
5. Monitor deployment in Actions tab

### 5. Releasing to Production

1. Ensure all changes are in main
2. Create a tag: `git tag -a v1.2.3 -m "Release v1.2.3"`
3. Push tag: `git push origin v1.2.3`
4. CI automatically builds, tests, and deploys to production
5. Verify at production URL

## For DevOps/Infrastructure

### 1. Configure Repository Secrets

In GitHub Settings > Secrets and variables > Actions:

```
# Docker Registry
DOCKER_REGISTRY=your-registry.com
DOCKER_USERNAME=your-username
DOCKER_PASSWORD=your-token

# Staging
STAGING_DEPLOY_KEY=[paste SSH private key]
STAGING_DEPLOY_HOST=staging.example.com
STAGING_DEPLOY_USER=deploy
STAGING_URL=https://staging.example.com

# Production
PRODUCTION_DEPLOY_KEY=[paste SSH private key]
PRODUCTION_DEPLOY_HOST=prod.example.com
PRODUCTION_DEPLOY_USER=deploy
PRODUCTION_URL=https://example.com
```

### 2. Configure Branch Protection

In Settings > Branches > Add rule for main:

- [x] Require a pull request before merging
- [x] Require status checks to pass before merging
- [x] Require branches to be up to date before merging
- [x] Dismiss stale pull request approvals
- [x] Require code review (suggest 1-2 reviewers)

### 3. Enable Auto-Merge (Optional)

In Settings > Repository > Allow auto-merge

### 4. Configure Notifications

In Settings > Notifications:
- Workflow notifications: Enabled
- Email for failed workflows: Your email

## Workflow Reference

### What Runs on Every PR?

```
CI Pipeline:
├── backend-lint (Ruff + mypy)      ✓ Must pass
├── backend-test (pytest + coverage) ✓ Must pass (90%+)
├── frontend-lint (Biome)            ✓ Must pass
├── frontend-build (TypeScript + Vite) ✓ Must pass
├── frontend-test (Playwright)       ✓ Must pass
├── security-scan (Trivy + npm audit) ✓ Must pass (critical/high)
└── dependency-check (outdated)      ⓘ Informational
```

### What Happens on Merge to Main?

```
Deploy to Staging:
├── Build Docker images
├── Push to registry
├── Deploy to staging server
├── Run smoke tests
└── Post status
```

### What Happens on Version Tag?

```
Deploy to Production:
├── Build Docker images
├── Push to registry
├── Create GitHub release
├── Deploy to production
├── Run health checks
└── Post notification
```

## Debugging CI Failures

### If Lint Fails

```bash
# Backend
cd backend
uv run ruff check . --fix
uv run ruff format .

# Frontend
cd frontend
npx biome check --write ./
```

### If Type Check Fails

```bash
# Backend
cd backend
uv run mypy app --ignore-missing-imports

# Frontend
cd frontend
npx tsc --noEmit
```

### If Tests Fail

```bash
# Backend
cd backend
uv run pytest -v --tb=short

# Frontend
cd frontend
npm run test -- --headed  # See browser while testing
```

### If Security Scan Fails

Check the PR comment with detailed vulnerability info:
- Update the vulnerable package
- Or, if necessary, document exception
- Re-push to trigger new scan

## Useful Commands

### View Workflow Status
- GitHub Actions tab: https://github.com/your-repo/actions
- CLI: `gh workflow list` (requires gh CLI)

### Manually Trigger Deploy
- Go to Actions > Deploy
- Click "Run workflow"
- Select environment
- Click "Run workflow"

### View Logs
- Click workflow run name
- Click failing job
- Expand steps to see detailed logs

### Rerun Workflow
- Click workflow run
- Click "Re-run failed jobs" or "Re-run all jobs"

## Common Questions

**Q: My PR is blocked by failing CI, what do I do?**
A: Click on the failing check, read the error, fix locally, and push again.

**Q: How do I deploy a hotfix to production?**
A: Create a PR, get it merged to main, create a version tag (v1.2.3), push it.

**Q: Can I test my changes before creating a PR?**
A: Yes! Run all the checks locally as shown in "Before Pushing Changes" section.

**Q: How do I update a dependency?**
A: Use `uv update package-name` (backend) or `npm update package-name` (frontend).

**Q: My deployment failed, how do I rollback?**
A: Either deploy the previous tag, or fix the issue and create a new tag.

**Q: Can I skip CI checks?**
A: No, this is by design. All changes must pass all checks.

## Resources

- **Full CI/CD Documentation**: `.github/CI_CD_GUIDE.md`
- **Contributing Guidelines**: `.github/CONTRIBUTING.md`
- **Implementation Details**: `.github/IMPLEMENTATION_SUMMARY.md`

## Getting Help

1. Check the documentation files above
2. Look at the GitHub Actions logs for detailed error messages
3. Review similar PRs to see how others handled similar issues
4. Ask in team discussions or create an issue with details

---

**Ready to develop?** Start with step 1 above!
