# rails-platform

Shared architectural baseline for Rails 8 apps in this org. Every decision here has been validated across at least three production apps: **toaster**, **voice-assistant**, and **evs-mapper**.

Use this on day one of a new app. Fork the decisions that apply, skip the ones that don't, and record deviations as ADRs in your own `docs/adr/`.

---

## Docs

| Document | Purpose |
|---|---|
| [BASELINE.md](BASELINE.md) | Quick-reference checklist — stack table, Dockerfile pattern, Compose pattern, gotchas |
| [PLATFORM.md](PLATFORM.md) | Full narrative — rationale, code snippets, and configuration detail for every decision |
| [docs/adr/](docs/adr/) | Individual Architecture Decision Records |

## ADR Index

| # | Decision |
|---|---|
| [0001](docs/adr/0001-rails-hotwire-monolith.md) | Rails + Hotwire monolith; no separate frontend |
| [0002](docs/adr/0002-goodjob-background-queue.md) | GoodJob as background job queue |
| [0003](docs/adr/0003-postgres-only-no-redis.md) | Postgres for queue, cache, and cable; no Redis |
| [0004](docs/adr/0004-coolify-deployment.md) | Coolify for deployment |
| [0005](docs/adr/0005-propshaft-asset-pipeline.md) | Propshaft over Sprockets |
| [0006](docs/adr/0006-jemalloc.md) | jemalloc via LD_PRELOAD in Docker |
| [0007](docs/adr/0007-tdd-rspec-factorybot.md) | TDD with RSpec + FactoryBot |
| [0008](docs/adr/0008-brakeman-bundler-audit.md) | Brakeman + Bundler Audit as mandatory gates |
| [0009](docs/adr/0009-pundit-authorization.md) | Pundit for authorization |
| [0010](docs/adr/0010-configurable-storage-provider.md) | Configurable storage provider (local / MinIO / S3) |
| [0011](docs/adr/0011-organization-scaffold.md) | Organization model from day one for future multi-tenancy |

## Stack at a Glance

| Concern | Choice |
|---|---|
| Framework | Rails 8+ (Hotwire monolith) |
| Language | Ruby 3.4+ |
| Database | PostgreSQL (Neon in prod; no Redis) |
| Jobs | GoodJob |
| Cache / Cable | solid_cache + solid_cable |
| Assets | Propshaft + Importmap + Tailwind |
| Auth | Devise |
| Authorization | Pundit |
| Storage | Active Storage — local / MinIO / S3 via `STORAGE_PROVIDER` |
| Deploy | Coolify via Docker Compose |
| Testing | RSpec + FactoryBot |
| Security | Brakeman + Bundler Audit (CI gates) |
