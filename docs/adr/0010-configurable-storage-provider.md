# 0010 — Configurable storage provider (local / MinIO / AWS S3)

**Status:** Accepted  
**Apps:** evs-mapper

## Decision

Active Storage's backend is selected at deploy time via the `STORAGE_PROVIDER` environment variable. No code changes are needed to switch between hosted (S3) and on-prem (MinIO) deployments.

```ruby
# config/environments/production.rb
config.active_storage.service = ENV.fetch("STORAGE_PROVIDER", "local").to_sym
```

`config/storage.yml` defines three stanzas: `local` (disk, for dev/CI), `minio` (self-hosted S3-compatible), and `amazon` (AWS S3). The MinIO stanza uses `force_path_style: true` because MinIO does not support virtual-hosted-style bucket URLs by default.

```ruby
# Gemfile
gem "aws-sdk-s3", require: false   # loaded only when needed
gem "image_processing", "~> 1.2"   # Active Storage variants
```

For on-prem Docker Compose deployments, a `minio` service and a one-shot `minio_init` container (which creates the bucket) are added to `docker-compose.yml`.

## Rationale

Some clients require that uploaded files (e.g. blueprint PDFs) never leave their own infrastructure. Supporting on-prem MinIO without a separate codebase eliminates a fork and keeps the deployment surface to one image per environment.

## Considered options

- **S3 only** — simpler but blocks on-prem adoption.
- **Separate on-prem codebase** — avoids env-var branching but creates two codebases to maintain.
- **Env-var configurable provider** ← chosen — one image, two deployment modes.

## Consequences

- `STORAGE_PROVIDER`, `MINIO_*`, and `AWS_*` env vars must be documented and wired in Coolify per deployment.
- MinIO adds two Docker Compose services (`minio`, `minio_init`) to on-prem stacks.
- `aws-sdk-s3` is in the Gemfile with `require: false` to avoid load-time overhead when using local disk.
