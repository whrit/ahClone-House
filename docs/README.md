# SEO Platform Documentation

Welcome to the SEO Platform MVP documentation. This comprehensive guide will help you understand, use, develop, and deploy the platform.

## Table of Contents

### Getting Started
- [Quick Start Guide](./getting-started.md) - Get up and running in 15 minutes
- [User Guide](./user-guide.md) - Complete feature walkthrough
- [FAQ](./faq.md) - Frequently asked questions

### Development
- [Developer Guide](./development.md) - Local development setup and workflow
- [Architecture Overview](./architecture.md) - System design and technical architecture
- [API Reference](./api-reference.md) - Complete API endpoint documentation

### Deployment
- [Deployment Guide](./deployment.md) - Production deployment instructions
- [Configuration Reference](./configuration.md) - Environment variables and settings

---

## What is the SEO Platform?

The SEO Platform is a comprehensive, self-hosted SEO analytics and optimization tool designed to help digital marketers and SEO professionals:

- **Audit websites** for technical SEO issues with automated crawling
- **Track keyword rankings** with SERP position monitoring
- **Analyze search performance** with Google Search Console integration
- **Monitor backlinks** using Common Crawl data
- **Compare PPC and SEO** with Google Ads integration
- **Aggregate traffic data** from multiple sources (GA4, GSC, CrUX, CSV)

### Key Features

| Feature | Description | Status |
|---------|-------------|--------|
| **Site Audits** | Automated crawling with 17+ SEO issue types | ✅ Complete |
| **Google Search Console** | OAuth integration, query/page explorers, opportunities | ✅ Complete |
| **Keyword Clustering** | N-gram based query clustering with Jaccard similarity | ✅ Complete |
| **Rank Tracking** | Provider-based SERP tracking with position history | ✅ Complete |
| **Backlinks** | Common Crawl ingestion, domain/anchor analysis | ✅ Complete |
| **PPC Analytics** | Google Ads integration, SEO+PPC overlap analysis | ✅ Complete |
| **Traffic Panel** | Multi-source aggregation with CSV import | ✅ Complete |
| **Background Jobs** | Celery task queue with Redis broker | ✅ Complete |
| **Rate Limiting** | slowapi endpoint throttling | ✅ Complete |
| **Caching** | Redis-based query caching | ✅ Complete |

---

## Technology Stack

### Backend
- **Framework**: FastAPI (Python 3.12+)
- **ORM**: SQLModel
- **Database**: PostgreSQL 17
- **Task Queue**: Celery + Redis
- **Authentication**: JWT with OAuth2
- **Migrations**: Alembic

### Frontend
- **Framework**: React 18 with TypeScript
- **Routing**: TanStack Router
- **UI Library**: shadcn/ui + Tailwind CSS
- **Build Tool**: Vite
- **API Client**: Auto-generated from OpenAPI

### Infrastructure
- **Containerization**: Docker + Docker Compose
- **Reverse Proxy**: Traefik (automatic HTTPS)
- **Monitoring**: Flower (Celery), Adminer (DB)
- **Testing**: Pytest (backend), Playwright (frontend E2E)

---

## Quick Links

### For End Users
- [Getting Started](./getting-started.md#first-login) - Create your first project
- [Running an Audit](./user-guide.md#site-audits) - Find SEO issues
- [Connecting GSC](./user-guide.md#keywords) - Import search data
- [Tracking Keywords](./user-guide.md#rank-tracking) - Monitor positions

### For Developers
- [Local Setup](./development.md#local-development-setup) - Start coding
- [Project Structure](./development.md#project-structure) - Understand the codebase
- [Adding Features](./development.md#adding-new-features) - Extend the platform
- [Running Tests](./development.md#testing) - Ensure quality

### For DevOps/Admins
- [Docker Deployment](./deployment.md#docker-deployment) - Self-hosted setup
- [Cloud Deployment](./deployment.md#cloud-deployment-options) - AWS, GCP, DigitalOcean
- [Environment Config](./deployment.md#environment-configuration) - Production settings
- [Monitoring](./deployment.md#monitoring--observability) - Health checks and logs

---

## Architecture at a Glance

```
┌─────────────────────────────────────────────────────────────┐
│                         Internet                             │
└───────────────────────┬─────────────────────────────────────┘
                        │
                   ┌────▼────┐
                   │ Traefik │ (SSL/TLS, Routing)
                   └────┬────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
   ┌────▼────┐   ┌──────▼──────┐  ┌────▼────┐
   │Frontend │   │   Backend   │  │ Adminer │
   │ (React) │   │  (FastAPI)  │  │ Flower  │
   └─────────┘   └──────┬──────┘  └─────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
   ┌────▼────┐   ┌──────▼──────┐  ┌────▼────┐
   │ Worker  │   │    Beat     │  │  Redis  │
   │(Celery) │   │ (Scheduler) │  │ (Queue) │
   └────┬────┘   └──────┬──────┘  └─────────┘
        │               │
        └───────┬───────┘
                │
         ┌──────▼──────┐
         │  PostgreSQL │
         │  (Database) │
         └─────────────┘
```

---

## Project Status

**Current Version**: 1.0.0 (MVP)
**Status**: Production Ready
**Last Updated**: December 2024

### Test Coverage
- Backend: 784 tests passing
- Zero mypy type errors
- Zero TypeScript/frontend build errors
- Full CRUD test coverage for all modules

### Performance Benchmarks
- API response time: <100ms (p95)
- Audit crawl speed: ~50 pages/minute (default settings)
- SERP refresh: ~100 keywords/day (GSC-based provider)
- Background job throughput: 4 concurrent workers

---

## Support and Contribution

### Getting Help
- **Documentation Issues**: Open a GitHub issue
- **Bug Reports**: Use the bug report template
- **Feature Requests**: Use the feature request template

### Contributing
1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Write tests for your changes
4. Ensure all tests pass (`pytest`, `npm test`)
5. Submit a pull request

---

## License

This project is licensed under the MIT License - see the [LICENSE](../LICENSE) file for details.

---

## Changelog

See [IMPLEMENTATION-OVERVIEW.md](../IMPLEMENTATION-OVERVIEW.md) for detailed sprint completion summaries and feature implementation history.

---

## Next Steps

1. **New Users**: Start with the [Quick Start Guide](./getting-started.md)
2. **Developers**: Read the [Developer Guide](./development.md)
3. **DevOps**: Check the [Deployment Guide](./deployment.md)
4. **API Integration**: Review the [API Reference](./api-reference.md)

---

**Questions?** Check the documentation or open an issue on GitHub.
