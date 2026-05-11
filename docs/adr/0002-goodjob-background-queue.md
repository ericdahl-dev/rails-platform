# GoodJob as background job queue

GoodJob is the background job queue. Postgres-backed, no Redis dependency.

## Context

Solid Queue is Rails 8's default Postgres-backed job queue, but its dashboard (Mission Control Jobs) requires the Sprockets asset pipeline, which caused friction in apps using Propshaft. GoodJob provides equivalent Postgres-backed, Redis-free operation with a dashboard that mounts as a Rails engine with no asset pipeline dependencies.

Both toaster and voice-assistant use GoodJob.

## Decision

Use GoodJob for all new Rails apps.

## Configuration

- **Production**: `execution_mode: :external` — separate `bundle exec good_job start` worker process.
- **Development**: `execution_mode: :async` — runs inside Puma, no separate process needed.

## Consequences

- No Redis dependency.
- Built-in dashboard for job observability at no extra integration cost.
- Proven stable in two production apps.
- Recurring jobs via `config/recurring.yml`.
