# Contributing to SEO Platform

Thank you for your interest in contributing to the SEO Platform! This document provides guidelines and instructions for contributing.

## Code of Conduct

We are committed to providing a welcoming and inspiring community for all. Please read and follow our code of conduct in all your interactions with the community.

## Getting Started

### Prerequisites

- Python 3.10+ with uv
- Node.js 20+
- Docker and Docker Compose
- Git

### Setup Development Environment

#### Backend Setup

```bash
# Navigate to backend directory
cd backend

# Install Python dependencies using uv
uv sync

# Set up environment variables
cp ../.env.example .env

# Run database migrations
uv run alembic upgrade head

# Start development server
uv run fastapi run app/main.py --reload
```

#### Frontend Setup

```bash
# Navigate to frontend directory
cd frontend

# Install npm dependencies
npm ci

# Set up environment variables
cp .env.example .env.local

# Start development server
npm run dev
```

### Running Tests Locally

#### Backend Tests

```bash
cd backend

# Run all tests
uv run pytest -v

# Run with coverage
uv run pytest --cov=app --cov-report=html

# Run specific test file
uv run pytest tests/test_auth.py -v

# Run specific test
uv run pytest tests/test_auth.py::test_login -v
```

#### Frontend Tests

```bash
cd frontend

# Run Playwright tests
npm run test

# Run tests in headed mode (see browser)
npx playwright test --headed

# Run specific test file
npx playwright test tests/auth.spec.ts
```

## Development Workflow

### 1. Create a Feature Branch

```bash
git checkout -b feature/your-feature-name
```

Use one of these prefixes:
- `feature/` - New features
- `fix/` - Bug fixes
- `chore/` - Maintenance tasks
- `docs/` - Documentation updates
- `refactor/` - Code refactoring
- `perf/` - Performance improvements

### 2. Make Your Changes

#### Backend Changes

- Follow PEP 8 style guide (enforced by Ruff)
- Write type hints for all functions (enforced by mypy)
- Add tests for new functionality
- Update documentation as needed

#### Frontend Changes

- Follow the Biome style guide
- Use TypeScript, avoid any types
- Add tests for new components
- Update Storybook if applicable

### 3. Run Local Checks

Before pushing, run all quality checks locally:

#### Backend

```bash
cd backend

# Linting
uv run ruff check .

# Format code
uv run ruff format .

# Type checking
uv run mypy app --ignore-missing-imports

# Run tests
uv run pytest -v --cov=app

# Check coverage
uv run coverage report --fail-under=90
```

#### Frontend

```bash
cd frontend

# Linting and formatting
npx biome check --write ./

# Type checking
npx tsc --noEmit

# Run tests
npm run test

# Build
npm run build
```

### 4. Commit Your Changes

Use clear, descriptive commit messages:

```bash
# Good commit messages
git commit -m "feat: add user authentication with OAuth2"
git commit -m "fix: correct calculation of SEO score"
git commit -m "docs: update API documentation for /users endpoint"

# Avoid vague messages
git commit -m "fix stuff"
git commit -m "WIP"
```

Follow conventional commits:
- `feat:` - New feature
- `fix:` - Bug fix
- `docs:` - Documentation
- `style:` - Code style changes (formatting, semicolons, etc)
- `refactor:` - Code refactoring
- `perf:` - Performance improvements
- `test:` - Adding or updating tests
- `chore:` - Maintenance tasks

### 5. Push to Your Fork

```bash
git push origin feature/your-feature-name
```

### 6. Create a Pull Request

1. Go to the repository on GitHub
2. Click "New Pull Request"
3. Select your branch
4. Fill in the PR template with:
   - Description of changes
   - Type of change
   - Testing performed
   - Checklist completion

## Pull Request Requirements

All PRs must satisfy:

### Code Quality

- [ ] Linting passes (Ruff for Python, Biome for TypeScript)
- [ ] Type checking passes (mypy for Python, tsc for TypeScript)
- [ ] Code formatting is consistent
- [ ] No new warnings introduced

### Testing

- [ ] Unit tests added/updated
- [ ] All tests pass locally
- [ ] Integration tests pass (if applicable)
- [ ] Backend coverage >= 90%
- [ ] No test regressions

### Security

- [ ] No hardcoded secrets or credentials
- [ ] No SQL injection vulnerabilities
- [ ] No XSS vulnerabilities
- [ ] Dependencies are up to date
- [ ] No known vulnerabilities in dependencies

### Documentation

- [ ] Code is well-commented
- [ ] Complex logic is documented
- [ ] README updated if needed
- [ ] API documentation updated (if applicable)

### CI/CD

- [ ] All GitHub Actions workflows pass
- [ ] Security scans pass
- [ ] Code quality checks pass
- [ ] Coverage thresholds met

## Code Style Guide

### Backend (Python)

#### Imports

```python
# Standard library
import os
from typing import Optional

# Third party
from fastapi import FastAPI
from sqlmodel import SQLModel

# Local
from app.core.config import settings
```

#### Type Hints

```python
# Always use type hints
def calculate_score(value: int) -> float:
    return value * 1.5

# For optional values
def get_user(user_id: int) -> Optional[User]:
    return db.get_user(user_id)

# For complex types
from typing import Dict, List
def process_data(data: List[Dict[str, str]]) -> bool:
    return True
```

#### Naming Conventions

```python
# Constants
MAX_RETRIES = 3
API_TIMEOUT = 30

# Classes
class UserManager:
    pass

# Functions
def get_user_by_id(user_id: int):
    pass

# Private functions
def _internal_helper():
    pass
```

### Frontend (TypeScript)

#### Imports

```typescript
// React and external libraries
import React, { useState } from 'react';
import { useQuery } from '@tanstack/react-query';

// Internal components
import { Button } from '@/components/ui/button';

// Utilities
import { formatDate } from '@/lib/utils';
```

#### Type Definitions

```typescript
// Interfaces for components
interface UserCardProps {
  user: User;
  onSelect?: (userId: number) => void;
}

// Interfaces for API responses
interface ApiResponse<T> {
  data: T;
  status: number;
}

// Avoid using 'any'
// ❌ function processData(data: any) {}
// ✅ function processData(data: unknown) {}
// ✅ function processData<T>(data: T) {}
```

#### Component Structure

```typescript
// Use functional components
export const UserCard: React.FC<UserCardProps> = ({ user, onSelect }) => {
  const [isLoading, setIsLoading] = useState(false);

  return (
    <div className="user-card">
      {/* Component JSX */}
    </div>
  );
};
```

## Testing Guidelines

### Backend Testing

```python
# Use descriptive test names
def test_user_creation_with_valid_email():
    # Arrange
    user_data = {"email": "test@example.com", "password": "secure"}

    # Act
    user = create_user(user_data)

    # Assert
    assert user.email == "test@example.com"
    assert user.is_active is True
```

### Frontend Testing

```typescript
// Use Playwright for E2E tests
test('should display user profile', async ({ page }) => {
  // Navigate to page
  await page.goto('/profile');

  // Verify elements
  await expect(page.locator('[data-test="user-name"]')).toBeVisible();

  // Interact
  await page.click('[data-test="edit-button"]');
});
```

## Commit Message Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

Example:

```
feat(auth): implement OAuth2 authentication

- Add Google OAuth provider
- Implement token refresh flow
- Add session management

Closes #123
```

## Reporting Issues

### Security Issues

Please do NOT create a public GitHub issue. Instead, email security@example.com with:
- Description of the vulnerability
- Affected versions
- Steps to reproduce
- Suggested fix (if available)

### Bug Reports

Include:
- Clear description of the bug
- Steps to reproduce
- Expected vs actual behavior
- Screenshots (if applicable)
- System information (OS, Python/Node version, etc)

### Feature Requests

Include:
- Clear description of the feature
- Use case and benefits
- Proposed implementation (if applicable)
- Any related issues or discussions

## Performance Guidelines

### Backend

- Avoid N+1 queries (use eager loading)
- Cache expensive operations
- Use database indexes appropriately
- Monitor query performance
- Limit API response sizes

### Frontend

- Code-split large bundles
- Lazy load images
- Minimize re-renders with React.memo
- Use virtualization for large lists
- Monitor Core Web Vitals

## Documentation

### Code Comments

```python
# Good: Explains why, not what
# We cache the result because API calls are expensive
cached_result = get_cached_or_fetch(key)

# Bad: Obvious from code
# Get the user by ID
user = db.get_user(user_id)
```

### Function Documentation

```python
def calculate_seo_score(content: str, keywords: List[str]) -> float:
    """
    Calculate SEO score based on content analysis.

    Args:
        content: The page content to analyze
        keywords: List of target keywords

    Returns:
        Score between 0 and 100

    Raises:
        ValueError: If content is empty

    Example:
        >>> score = calculate_seo_score("Hello world", ["hello"])
        >>> score > 0
        True
    """
    ...
```

## Release Process

1. Ensure all changes are in main branch
2. Create a git tag: `git tag -a v1.2.3 -m "Release v1.2.3"`
3. Push tag: `git push origin v1.2.3`
4. GitHub Actions automatically creates release and deploys
5. Verify deployment succeeded
6. Update changelog if needed

## Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Ruff Documentation](https://docs.astral.sh/ruff/)
- [Biome Documentation](https://biomejs.dev/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [React Documentation](https://react.dev/)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/)

## Questions?

- Check existing issues and discussions
- Read the CI/CD guide at `.github/CI_CD_GUIDE.md`
- Create a new discussion for questions

Thank you for contributing!
