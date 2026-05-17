# Rails Platform Decisions

Extracted shared architectural decisions from three production apps:

- **toaster** — Rails 7.2 API → Hotwire monolith, email booking assistant
- **voice-assistant** — Rails 8.1, AI-powered outbound calling agent
- **evs-mapper** — Rails 8.1, blueprint PDF → room inventory pipeline for Environmental Services

New apps use **Rails 8** as the baseline. Where apps diverged, the Rails 8 approach is preferred unless noted otherwise.

---

## Table of Contents

1. [Stack Baseline](#stack-baseline)
2. [Frontend Architecture](#frontend-architecture)
3. [Background Jobs](#background-jobs)
4. [Database](#database)
5. [Asset Pipeline](#asset-pipeline)
6. [Authentication & Authorization](#authentication--authorization)
7. [Storage](#storage)
8. [Deployment](#deployment)
9. [Docker](#docker)
10. [Dev Environment](#dev-environment)
11. [Security & Code Quality](#security--code-quality)
12. [Testing](#testing)
13. [Observability](#observability)
14. [Configuration & Secrets](#configuration--secrets)
15. [Environment Variables](#environment-variables)
16. [Domain Modelling](#domain-modelling)

---

## Stack Baseline

| Concern | Decision |
|---|---|
| Framework | Rails 8.x (monolith, not API-only) |
| Ruby | Latest stable (track `.ruby-version`; use RVM in dev) |
| Database | PostgreSQL — hosted (Neon in prod); `DATABASE_URL` for pooled connection, direct URL for migrations |
| Web server | Puma + Thruster |
| Cache | `solid_cache` (Postgres-backed, no Redis) |
| Cable | `solid_cable` (Postgres-backed, no Redis) |

**No Redis.** All queue, cache, and cable are Postgres-backed. This keeps the operational surface small and eliminates a separate infrastructure dependency.

---

## Frontend Architecture

**Decision: Rails monolith with Hotwire (Turbo + Stimulus). No separate frontend framework.**

A single Rails app owns the control plane, web UI, webhooks, and background jobs. Hotwire handles real-time UI without a separate process or origin.

### Rationale

- A Rails API + Next.js/React split introduces CORS complexity, same-site cookie friction, a two-origin authentication model, and two deployment units — all before the product shape is stable enough to justify an API surface.
- Hotwire (Turbo Streams over Action Cable) is sufficient for real-time UI requirements in the apps built so far.
- Single process, single deployment, auth cookies just work.

### Asset Pipeline

Use **Propshaft** (not Sprockets). Simpler, no fingerprinting complexity, works well with importmap and tailwind.

```ruby
gem "propshaft"
gem "importmap-rails"
gem "tailwindcss-rails"
gem "turbo-rails"
gem "stimulus-rails"
```

### When to extract a separate frontend

Only if a native mobile app or third-party integration makes a public API surface worthwhile. That will be an explicit new decision, not a default.

---

## Background Jobs

**Decision: GoodJob**

```ruby
gem "good_job"
```

### Rationale

- Postgres-backed — no Redis, fits the monolith.
- Built-in dashboard mounts as a Rails engine with no asset pipeline dependencies (unlike Solid Queue's Mission Control, which caused Sprockets friction in toaster).
- Proven in both apps.
- Mature observability for job state visibility (queued, running, stuck, failed) without custom tooling.

### Configuration

```ruby
# config/environments/production.rb
config.active_job.queue_adapter = :good_job
config.good_job.execution_mode = :external  # separate worker process in prod
```

```ruby
# config/environments/development.rb
config.active_job.queue_adapter = :good_job
config.good_job.execution_mode = :async  # runs inside Puma in dev, no separate process
```

### Worker process (production)

```bash
bundle exec good_job start
```

Run as a separate container/process. See [Docker](#docker) for compose pattern.

### Concurrency env vars

```
RAILS_MAX_THREADS=5   # Puma threads = DB pool size
JOB_CONCURRENCY=5     # GoodJob threads
WEB_CONCURRENCY=2     # Puma workers (processes)
```

---

## Database

**PostgreSQL only.** No SQLite.

- Use `DATABASE_URL` for the pooled application connection.
- Use a direct (non-pooled) URL for `db:migrate` in production (Neon/PgBouncer pattern).
- DB pool size should equal `RAILS_MAX_THREADS`.

### macOS fork safety

Forked workers (GoodJob, Puma) connecting to Postgres on macOS can hit a libpq GSS/Kerberos path that segfaults in the child. Set in an initializer:

```ruby
# config/initializers/0_pg_gssenc_fork_safety.rb
ENV["PGGSSENCMODE"] ||= "disable"
```

---

## Authentication & Authorization

**Authentication: Devise**

```ruby
gem "devise"
gem "bcrypt", "~> 3.1.7"
```

Standard Rails authentication. No custom auth framework.

**Authorization: Pundit**

```ruby
gem "pundit"
```

Pundit policy classes live in `app/policies/`. Every controller that accesses protected resources calls `authorize @record` or `policy_scope(Model)`. An `ApplicationPolicy` provides defaults; resource-specific policies override only what differs.

```ruby
# app/controllers/application_controller.rb
include Pundit::Authorization
after_action :verify_authorized, except: :index
after_action :verify_policy_scoped, only: :index
```

Pundit was chosen over CanCanCan for its explicit, object-oriented policy model — each policy is a plain Ruby class that is easy to test in isolation.

---

## Storage

**Decision: env-var configurable provider (local / MinIO / AWS S3)**

Apps that handle user-uploaded files must support both hosted (AWS S3) and on-prem (MinIO) deployments without code changes. Provider is selected via `STORAGE_PROVIDER` at deploy time.

```ruby
# config/environments/production.rb
config.active_storage.service = ENV.fetch("STORAGE_PROVIDER", "local").to_sym
```

```yaml
# config/storage.yml
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

minio:
  service: S3
  access_key_id:     <%= ENV.fetch("MINIO_ACCESS_KEY", "minioadmin") %>
  secret_access_key: <%= ENV.fetch("MINIO_SECRET_KEY", "minioadmin") %>
  region:            us-east-1
  bucket:            <%= ENV.fetch("MINIO_BUCKET", "myapp") %>
  endpoint:          <%= ENV.fetch("MINIO_ENDPOINT", "http://minio:9000") %>
  force_path_style:  true

amazon:
  service: S3
  access_key_id:     <%= ENV["AWS_ACCESS_KEY_ID"] %>
  secret_access_key: <%= ENV["AWS_SECRET_ACCESS_KEY"] %>
  region:            <%= ENV.fetch("AWS_REGION", "us-east-1") %>
  bucket:            <%= ENV.fetch("AWS_BUCKET", "myapp") %>
```

```ruby
# Gemfile — only require when S3-compatible storage is used
gem "aws-sdk-s3", require: false
gem "image_processing", "~> 1.2"
```

### MinIO in Docker Compose

For on-prem deployments add a `minio` service and a one-shot `minio_init` container that creates the bucket:

```yaml
minio:
  image: minio/minio:latest
  command: server /data --console-address ":9001"
  expose: ["9000", "9001"]
  environment:
    MINIO_ROOT_USER:     ${MINIO_ACCESS_KEY:-minioadmin}
    MINIO_ROOT_PASSWORD: ${MINIO_SECRET_KEY:-minioadmin}
  volumes:
    - minio_data:/data
  healthcheck:
    test: ["CMD", "mc", "ready", "local"]
    interval: 10s
    timeout: 5s
    retries: 5
    start_period: 10s
  restart: unless-stopped

minio_init:
  image: minio/mc:latest
  depends_on:
    minio:
      condition: service_healthy
  entrypoint: >
    /bin/sh -c "
      mc alias set local http://minio:9000 ${MINIO_ACCESS_KEY:-minioadmin} ${MINIO_SECRET_KEY:-minioadmin};
      mc mb --ignore-existing local/${MINIO_BUCKET:-myapp};
      exit 0
    "
  restart: "no"
```

---

## Deployment

**Decision: Coolify (self-hosted PaaS) via Docker Compose**

Apps deploy as Docker Compose stacks managed by Coolify. No Kamal, no Heroku, no Kubernetes.

### How it works

- Coolify points at the repo and a `docker-compose.yml`.
- Minimal apps have two services: `web` (Puma) and `worker` (GoodJob). Apps with extra infrastructure (Postgres container, MinIO, sidecar parsers) add more services to the same compose file.
- Environment variables and secrets are configured in the Coolify UI — not committed to the repo.
- Coolify handles SSL termination via Let's Encrypt automatically.

### SSL / HTTPS

Because Coolify terminates SSL upstream, the Rails app runs behind a trusted proxy:

```ruby
# config/environments/production.rb
config.assume_ssl = ActiveModel::Type::Boolean.new.cast(ENV.fetch("RAILS_ASSUME_SSL", true))
config.force_ssl  = ActiveModel::Type::Boolean.new.cast(ENV.fetch("RAILS_FORCE_SSL", true))
config.ssl_options = { redirect: { exclude: ->(request) { request.path == "/up" } } }
```

### Health check

Coolify uses the `/up` endpoint. Configure in `docker-compose.yml`:

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://127.0.0.1:3000/up"]
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 60s
```

---

## Docker & Coolify

### Multi-stage Dockerfile pattern

Follow the Rails 8 default `Dockerfile`:

1. **base** — slim Ruby image, runtime packages only (`curl libjemalloc2 libvips postgresql-client`), enable jemalloc via `LD_PRELOAD`.
2. **build** — adds build tools (`build-essential git libpq-dev libvips libyaml-dev pkg-config`), installs gems, precompiles bootsnap and assets.
3. **final** — copies artifacts from build stage, runs as non-root user (`rails`, uid 1000).

```dockerfile
ENV LD_PRELOAD="/usr/local/lib/libjemalloc.so"  # jemalloc for reduced memory/latency
```

**jemalloc** is installed in both apps and linked via `LD_PRELOAD`. Don't skip this.

### docker-compose.yml pattern

This file serves double duty: Coolify uses it for production deploys; `docker compose up --build` works locally.

```yaml
services:
  web:
    build: .
    command: ["./bin/rails", "server", "-b", "0.0.0.0"]
    expose:
      - "3000"
    environment:
      - DATABASE_URL
      - RAILS_MASTER_KEY
      - RAILS_ENV=production
      - RAILS_ASSUME_SSL=${RAILS_ASSUME_SSL:-true}
      - RAILS_FORCE_SSL=${RAILS_FORCE_SSL:-true}
      - RAILS_LOG_LEVEL=${RAILS_LOG_LEVEL:-info}
      - RAILS_MAX_THREADS=${RAILS_MAX_THREADS:-5}
      - WEB_CONCURRENCY=${WEB_CONCURRENCY:-2}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://127.0.0.1:3000/up"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 60s
    restart: unless-stopped

  worker:
    build: .
    command: ["bundle", "exec", "good_job", "start"]
    environment:
      - DATABASE_URL
      - RAILS_MASTER_KEY
      - RAILS_ENV=production
      - RAILS_LOG_LEVEL=${RAILS_LOG_LEVEL:-info}
      - RAILS_MAX_THREADS=${RAILS_MAX_THREADS:-5}
      - JOB_CONCURRENCY=${JOB_CONCURRENCY:-5}
    restart: unless-stopped
```

**Note:** No `thruster` in the command — Coolify's proxy handles HTTP caching and compression upstream. The Rails server binds directly on port 3000.

---

## Dev Environment

### Procfile.dev

```
web: bin/rails server
css: bin/rails tailwindcss:watch
```

### Ruby version management

Use RVM. Always run `rvm use .` from the project root (picks up `.ruby-version`) before `bundle`, `rspec`, or other gem commands to ensure native extensions match the active interpreter.

### dotenv

```ruby
group :development, :test do
  gem "dotenv-rails"
end
```

`.env` holds local secrets. Never commit it. `.env.example` documents required vars.

---

## Security & Code Quality

### Brakeman

```ruby
gem "brakeman", require: false
```

```bash
bundle exec brakeman -q
```

Run before every merge. Resolve all warnings — no exceptions.

### Bundler Audit

```ruby
gem "bundler-audit", require: false
```

```bash
bundle exec bundler-audit check --update
```

### Rubocop

```ruby
gem "rubocop-rails-omakase", require: false
```

Inherit from `rubocop-rails-omakase`. Minimal local overrides. Run before committing backend changes.

### Host authorization

```ruby
# config/environments/production.rb
config.hosts << ENV["APP_HOST"] if ENV["APP_HOST"].present?
ENV.fetch("RAILS_ALLOWED_HOSTS", "").split(",").map(&:strip).reject(&:blank?).each do |host|
  config.hosts << host
end
config.host_authorization = { exclude: ->(request) { request.path == "/up" } }
```

---

## Testing

### Framework

**RSpec** for all tests.

```ruby
gem "rspec-rails", "~> 8.0"
gem "factory_bot_rails"
gem "capybara"
gem "cuprite"
```

### Browser / system testing

**Capybara + Cuprite** is the default browser testing stack for system specs.

```ruby
# spec/support/capybara.rb
Capybara.register_driver :cuprite do |app|
  Capybara::Cuprite::Driver.new(app, window_size: [1400, 1400])
end

Capybara.javascript_driver = :cuprite
```

### Conventions

- Specs under `spec/` mirroring the `app/` tree.
- FactoryBot for all test data — no fixtures.
- Request specs for API/controller behaviour.
- Keep tests close to the code they cover (co-locate where sensible).
- **TDD is the default.** Write the test first.

### Coverage

**SimpleCov** — local only. Do not add Codecov or remote upload unless explicitly decided.

```ruby
gem "simplecov", require: false
gem "simplecov_json_formatter", require: false
```

### Commands

```bash
bundle exec rspec                   # full suite
bundle exec rspec spec/models/      # models only
bundle exec rspec spec/requests/    # request specs
```

---

## Observability

### Health check

Rails `/up` endpoint is the standard health check. Exclude it from SSL redirect and host authorization checks (see above).

### Logging

```ruby
# production.rb
config.logger   = ActiveSupport::TaggedLogging.logger(STDOUT)
config.log_tags = [ :request_id ]
config.log_level = ENV.fetch("RAILS_LOG_LEVEL", "info")
config.silence_healthcheck_path = "/up"
```

Structured JSON logging to stdout; log aggregator (e.g. Logtail, Datadog) collects from container output.

### GoodJob dashboard

Mount in routes:

```ruby
# config/routes.rb
mount GoodJob::Engine => "/jobs"
```

Protect with HTTP basic auth outside development:

```ruby
# config/initializers/good_job_auth.rb
Rails.application.config.good_job.dashboard_default_filter = { queues: nil }

GoodJob::Engine.middleware.use(Rack::Auth::Basic) do |user, password|
  unless Rails.env.local?
    ActiveSupport::SecurityUtils.secure_compare(
      user, ENV.fetch("MISSION_CONTROL_USERNAME", "ops")
    ) & ActiveSupport::SecurityUtils.secure_compare(
      password, ENV.fetch("MISSION_CONTROL_PASSWORD", "")
    )
  end
end
```

---

## Configuration & Secrets

### Rails credentials

Use `config/credentials.yml.enc` (per-environment or shared). `RAILS_MASTER_KEY` is the only secret injected at deploy time; everything else lives encrypted in credentials.

```bash
bin/rails credentials:edit
```

### Master key

- Never commit `config/master.key`.
- Pass `RAILS_MASTER_KEY` as an environment variable / Kamal secret in production.

---

## Environment Variables

Standard set across apps. All optional values should have safe defaults.

| Variable | Required | Default | Purpose |
|---|---|---|---|
| `RAILS_MASTER_KEY` | prod | — | Credentials decryption |
| `DATABASE_URL` | prod | — | Postgres connection (pooled) |
| `APP_HOST` | prod | — | Rails host authorization + mailer URLs |
| `RAILS_ALLOWED_HOSTS` | optional | `""` | Comma-separated extra allowed hosts |
| `RAILS_ASSUME_SSL` | optional | `true` | Tell Rails it's behind an SSL terminator |
| `RAILS_FORCE_SSL` | optional | `true` | Redirect HTTP → HTTPS |
| `RAILS_LOG_LEVEL` | optional | `info` | Log verbosity |
| `RAILS_MAX_THREADS` | optional | `5` | Puma thread count = DB pool size |
| `WEB_CONCURRENCY` | optional | `2` | Puma worker (process) count |
| `JOB_CONCURRENCY` | optional | `5` | GoodJob thread count |
| `PGGSSENCMODE` | optional | `disable` | macOS fork safety (set in initializer) |
| `STORAGE_PROVIDER` | optional | `local` | Active Storage backend: `local`, `minio`, `amazon` |
| `MINIO_ACCESS_KEY` | if minio | — | MinIO root user |
| `MINIO_SECRET_KEY` | if minio | — | MinIO root password |
| `MINIO_BUCKET` | if minio | app name | S3 bucket name |
| `MINIO_ENDPOINT` | if minio | — | MinIO service URL (e.g. `http://minio:9000`) |
| `AWS_ACCESS_KEY_ID` | if amazon | — | AWS S3 access key |
| `AWS_SECRET_ACCESS_KEY` | if amazon | — | AWS S3 secret |
| `AWS_REGION` | if amazon | `us-east-1` | AWS S3 region |
| `AWS_BUCKET` | if amazon | app name | AWS S3 bucket |

---

## Domain Modelling

**Decision: Scaffold `Organization` from day one**

Every app that may eventually serve multiple customers/tenants should include an `Organization` model from the first migration, even if the MVP ships with a single hard-coded org.

```
Organization → Facility → Floor → Room   (example from evs-mapper)
```

All models that belong to a resource owned by an org are scoped through that org transitively. Controllers scope queries through the org from the start.

**Why now:** Retrofitting multi-tenancy later requires adding `organization_id` to every table and touching every controller and export query — typically a 2–3 day refactor. Adding the model on day one costs ~2 hours and makes the schema ready without building any multi-tenant product surface (no signup flow, no org-switching UI, no billing).

**MVP pattern:**
- One `Organization` row in `seeds.rb`.
- No org-switching UI or signup flow.
- All controllers scope through `current_organization` (hardcoded to the seeded org in MVP).

---

## ADR Index

Individual decision records for this platform live in `docs/adr/`:

| # | Title |
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
| [0012](docs/adr/0012-browser-testing-capybara-cuprite.md) | Browser testing with Capybara + Cuprite |
