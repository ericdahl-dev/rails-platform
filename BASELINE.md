# Rails App Baseline

Opinionated defaults for new Rails apps in this org. Every decision here has been
validated in production across at least three apps (toaster, voice-assistant, evs-mapper).

Use this doc on day one of a new app. Fork the decisions that apply, skip the ones
that don't, and record deviations as ADRs in your own `docs/adr/`.

---

## Stack

| Concern | Choice | Notes |
|---|---|---|
| Framework | Rails 8+ (Hotwire monolith) | No separate frontend; extract API later if needed |
| Language | Ruby 3.4+ | Match `.ruby-version`; use RVM locally |
| Database | PostgreSQL via Neon (production) | Local socket in dev/test; `DATABASE_URL` only in prod/CI |
| Asset pipeline | Propshaft | Replaces Sprockets; zero config |
| CSS | Tailwind CSS (`tailwindcss-rails`) | Built at image build time via `assets:precompile` |
| JS | Importmap + Stimulus | No Node/webpack in production |
| Realtime | Turbo Streams over Action Cable | Sufficient for live UI without a separate websocket server |
| Jobs | GoodJob | Postgres-backed, no Redis, built-in dashboard |
| Web server | Puma + Thruster | Thruster handles SSL termination and HTTP/2 in production |
| Auth | `has_secure_password` (bcrypt) or Devise | bcrypt for simple session auth; Devise if you need full user management |
|| Authorization | Pundit | Policy objects in `app/policies/`; `authorize` in every controller action |
|| Storage | Active Storage + configurable provider | `local` dev, `minio` on-prem, `amazon` hosted — driven by `STORAGE_PROVIDER` env var |
|| Testing | RSpec + FactoryBot + SimpleCov + Capybara + Cuprite | `bundle exec rspec` from repo root |
| Linting | rubocop-rails-omakase | `bundle exec rubocop` |
| Security scan | Brakeman | `bundle exec brakeman -q` in CI |

---

## Repository Layout

Rails app files live at the **repo root** — not inside a `backend/` subdirectory.
`Gemfile`, `Dockerfile`, `docker-compose.yml`, `app/`, `config/`, `db/`, `spec/` etc.
are all at the top level. This matches standard Rails conventions and avoids
friction with Coolify's base directory config and CI `working-directory` overrides.

```
/
├── app/
├── bin/
├── config/
├── db/
├── spec/
├── Dockerfile
├── docker-compose.yml
├── Gemfile
├── AGENTS.md          # AI agent instructions (single file, not CLAUDE.md + AGENTS.md)
├── CONTEXT.md         # Domain glossary
└── docs/adr/          # App-specific ADRs
```

---

## Dockerfile

Use a two-stage build. Key decisions:

- **No `# syntax=` line** — Coolify prepends its own; two directives break the build.
- **jemalloc** — reduces memory usage and GC latency; link via `LD_PRELOAD`.
- **Assets compiled at build time** — run `SECRET_KEY_BASE_DUMMY=1 ./bin/rails assets:precompile` in the build stage. This compiles Tailwind CSS via Propshaft. Do NOT rely on a separate `tailwindcss:build` step — `assets:precompile` covers it.
- **Non-root user** — `rails` user uid/gid 1000.
- **Entrypoint** — `bin/docker-entrypoint` runs `db:prepare` before boot.
- **CMD** — `./bin/thrust ./bin/rails server` (Thruster wraps Puma).

```dockerfile
ARG RUBY_VERSION=3.4.x
FROM docker.io/library/ruby:$RUBY_VERSION-slim AS base

WORKDIR /rails

RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y curl libjemalloc2 postgresql-client && \
    ln -s /usr/lib/$(uname -m)-linux-gnu/libjemalloc.so.2 /usr/local/lib/libjemalloc.so && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

ENV RAILS_ENV="production" \
    BUNDLE_DEPLOYMENT="1" \
    BUNDLE_PATH="/usr/local/bundle" \
    BUNDLE_WITHOUT="development" \
    LD_PRELOAD="/usr/local/lib/libjemalloc.so"

FROM base AS build

RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y build-essential git libpq-dev pkg-config && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

COPY Gemfile Gemfile.lock ./
RUN bundle install && \
    rm -rf ~/.bundle/ "${BUNDLE_PATH}"/ruby/*/cache "${BUNDLE_PATH}"/ruby/*/bundler/gems/*/.git && \
    bundle exec bootsnap precompile --gemfile

COPY . .
RUN bundle exec bootsnap precompile app/ lib/
RUN SECRET_KEY_BASE_DUMMY=1 ./bin/rails assets:precompile

FROM base

RUN groupadd --system --gid 1000 rails && \
    useradd rails --uid 1000 --gid 1000 --create-home --shell /bin/bash
USER 1000:1000

COPY --chown=rails:rails --from=build "${BUNDLE_PATH}" "${BUNDLE_PATH}"
COPY --chown=rails:rails --from=build /rails /rails

ENTRYPOINT ["/rails/bin/docker-entrypoint"]
EXPOSE 80
CMD ["./bin/thrust", "./bin/rails", "server"]
```

---

## docker-compose.yml

Two services: `web` (Thruster + Puma) and `worker` (GoodJob). All config via env vars — no hardcoded values.

```yaml
services:
  web:
    build: .
    command: ["./bin/thrust", "./bin/rails", "server"]
    expose:
      - "80"
    environment:
      RAILS_MASTER_KEY: ${RAILS_MASTER_KEY}
      DATABASE_URL: ${DATABASE_URL}
      APP_HOST: ${APP_HOST}
      RAILS_ALLOWED_HOSTS: ${RAILS_ALLOWED_HOSTS:-}
      RAILS_ASSUME_SSL: ${RAILS_ASSUME_SSL:-true}
      RAILS_FORCE_SSL: ${RAILS_FORCE_SSL:-true}
      RAILS_LOG_LEVEL: ${RAILS_LOG_LEVEL:-info}
      RAILS_MAX_THREADS: ${RAILS_MAX_THREADS:-5}
      WEB_CONCURRENCY: ${WEB_CONCURRENCY:-2}
      JOB_CONCURRENCY: ${JOB_CONCURRENCY:-5}
    restart: unless-stopped

  worker:
    build: .
    command: ["bundle", "exec", "good_job", "start"]
    environment:
      RAILS_MASTER_KEY: ${RAILS_MASTER_KEY}
      DATABASE_URL: ${DATABASE_URL}
      APP_HOST: ${APP_HOST}
      RAILS_LOG_LEVEL: ${RAILS_LOG_LEVEL:-info}
      RAILS_MAX_THREADS: ${RAILS_MAX_THREADS:-5}
      JOB_CONCURRENCY: ${JOB_CONCURRENCY:-5}
    restart: unless-stopped
```

**Coolify note:** Set "Docker Compose Location" to `/docker-compose.yml` and base directory to `/` (repo root). Set the domain in `docker_compose_domains` with the container port, e.g. `https://myapp.example.com:80`. The port tells Caddy which container port to proxy to; it does not appear in the public URL.

---

## Production Config (`config/environments/production.rb`)

Key decisions:

```ruby
# SSL
config.assume_ssl = ActiveModel::Type::Boolean.new.cast(ENV.fetch("RAILS_ASSUME_SSL", true))
config.force_ssl  = ActiveModel::Type::Boolean.new.cast(ENV.fetch("RAILS_FORCE_SSL", true))
config.ssl_options = { redirect: { exclude: ->(request) { request.path == "/up" } } }

# Logging
config.logger    = ActiveSupport::TaggedLogging.logger(STDOUT)
config.log_tags  = [:request_id]
config.log_level = ENV.fetch("RAILS_LOG_LEVEL", "info")
config.silence_healthcheck_path = "/up"

# GoodJob runs in external mode in production (separate worker process)
config.active_job.queue_adapter  = :good_job
config.good_job.execution_mode   = :external

# Allowed hosts — driven by env so no code change needed per deployment
app_host = ENV["APP_HOST"].presence
config.hosts << app_host if app_host
ENV.fetch("RAILS_ALLOWED_HOSTS", "").split(",").map(&:strip).reject(&:blank?).each do |h|
  config.hosts << h
end
# Always exclude /up from host authorization so the healthcheck works
config.host_authorization = { exclude: ->(request) { request.path == "/up" } }
```

**Never hardcode hostnames in `application.rb` or `production.rb`.** Use `APP_HOST` + `RAILS_ALLOWED_HOSTS` env vars so the same image works across environments without rebuild.

---

## Database (Neon)

- **Production:** Neon serverless Postgres. Use the pooled connection string for `DATABASE_URL` (app/workers). Use the direct (non-pooled) connection for migrations (`bin/rails db:migrate`).
- **Dev/test:** Local Postgres via Unix socket. Do NOT set `DATABASE_URL` locally — omitting it lets `database.yml` fall back to local socket, which is faster and avoids network latency in the test cycle.
- **CI:** Postgres service container in GitHub Actions. `DATABASE_URL` is set in the workflow; local socket is not available.

```yaml
# config/database.yml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= [ENV.fetch("RAILS_MAX_THREADS", 5).to_i, 5].max %>
  url: <%= ENV["DATABASE_URL"] %>

development:
  <<: *default
  database: myapp_development   # used when DATABASE_URL is absent

test:
  <<: *default
  database: myapp_test<%= ENV.fetch("TEST_ENV_NUMBER", "") %>

production:
  <<: *default
```

---

## GoodJob

- **Development:** `execution_mode :async` — runs inside Puma, no separate process needed.
- **Production:** `execution_mode :external` — separate `worker` container runs `bundle exec good_job start`.
- **Dashboard:** mounts at `/jobs`. Protect with HTTP basic auth outside development (env vars `GOOD_JOB_USERNAME` / `GOOD_JOB_PASSWORD` or similar).
- **Queue names:** configure in `config/recurring.yml` and `config/queue.yml`. Worker `queues` must be a YAML array — a single comma-separated string is treated as one literal queue name.

---

## Coolify Deployment

- **Proxy:** Caddy (not Traefik). Domain in `docker_compose_domains` must include the container port: `https://myapp.example.com:80`. Caddy strips the port from the public URL.
- **Preview deployments:** template is `{{pr_id}}.{{domain}}`. Requires a wildcard DNS entry (`*.myapp.example.com`) pointing at the server.
- **Env vars:** set `RAILS_MASTER_KEY`, `DATABASE_URL`, `APP_HOST`, `RAILS_ALLOWED_HOSTS` in Coolify. Never commit secrets to the repo.
- **`RAILS_ALLOWED_HOSTS`:** comma-separated list of additional hosts beyond `APP_HOST`. Use this for preview deployment domains rather than hardcoding.

---

## macOS Dev / Fork Safety

GoodJob (and any gem that forks workers) connecting to Postgres on macOS can segfault via libpq's GSS/Kerberos probe in the fork child. Add this initializer:

```ruby
# config/initializers/0_pg_gssenc_fork_safety.rb
if RUBY_PLATFORM.include?("darwin") && ENV["PGGSSENCMODE"].to_s.empty?
  ENV["PGGSSENCMODE"] = "disable"
end
```

---

## CI (GitHub Actions)

```yaml
# .github/workflows/ci.yml
jobs:
  backend:
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: myapp_test
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U postgres -d myapp_test"
          --health-interval 10s --health-timeout 5s --health-retries 5
    env:
      RAILS_ENV: test
      DATABASE_URL: postgres://postgres:postgres@127.0.0.1:5432/myapp_test
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: 3.4.x
          bundler-cache: true
      - run: bundle exec rails db:prepare
      - run: bundle exec rails tailwindcss:build   # or assets:precompile
      - run: bundle exec rspec
      - run: bundle exec brakeman -q
      - run: bundle exec rubocop
```

---

## AGENTS.md

Every repo has a single `AGENTS.md` at the root (not both `AGENTS.md` and `CLAUDE.md`). It is the authoritative instruction file for all AI agents. Minimum sections:

- **Useful Commands** — build, test, lint, deploy sync
- **Build & Test** — exact commands with RVM/nvm setup
- **Beads Issue Tracker** — `bd prime` reference, session close protocol
- **Learned Workspace Facts** — env-specific gotchas discovered during development

---

## Issue Tracking & Agent Workflow

- Issues live in **GitHub Issues**.
- Triage labels: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`.
- Local issue sync: `GITHUB_TOKEN="$(gh auth token --user <username>)" bd github sync`
- Agent briefs use the [AGENT-BRIEF format](https://github.com/ericdahl-dev/rails-platform): behavioral spec, no file paths, concrete acceptance criteria, explicit out-of-scope.

---

## Known Gotchas Checklist

Before shipping a new app to production, verify:

- [ ] `Dockerfile` uses `assets:precompile` (not a separate tailwind step) — covers Propshaft + Tailwind in one pass
- [ ] No `# syntax=` line in `Dockerfile` — Coolify prepends its own
- [ ] `APP_HOST` + `RAILS_ALLOWED_HOSTS` env vars drive `config.hosts` — no hardcoded hostnames
- [ ] `/up` is excluded from both SSL redirect and host authorization
- [ ] `DATABASE_URL` is NOT set in local `.env` — dev/test use local socket
- [ ] `PGGSSENCMODE=disable` initializer present for macOS fork safety
- [ ] GoodJob `execution_mode :async` in development, `:external` in production
- [ ] Worker `queues` in `queue.yml` is a YAML array, not a string
- [ ] Coolify `docker_compose_domains` includes container port (e.g. `:80` or `:3000`)
- [ ] Wildcard DNS set up if preview deployments are needed
- [ ] `Organization` model present from first migration if multi-tenancy is plausible
- [ ] All controllers scope queries through the org from the start
- [ ] `STORAGE_PROVIDER` env var wired in `config/environments/production.rb`; `storage.yml` has `local`, `minio`, and `amazon` stanzas
- [ ] Pundit `authorize` and `policy_scope` calls verified in every controller (`verify_authorized` / `verify_policy_scoped` after_actions)
- [ ] Browser/system specs use Capybara with Cuprite as the JavaScript driver
