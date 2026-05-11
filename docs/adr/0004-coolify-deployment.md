# Coolify for deployment

Apps deploy as Docker Compose stacks managed by Coolify (self-hosted PaaS).

## Decision

- **Platform:** Coolify, self-hosted.
- **Deployment unit:** `docker-compose.yml` in the repo root (or app subdirectory).
- **Services:** `web` (Puma on port 3000) + `worker` (`bundle exec good_job start`), same image.
- **Secrets:** managed in the Coolify UI, injected as environment variables at runtime.
- **SSL:** Coolify terminates TLS via Let's Encrypt. Rails runs behind the proxy with `config.assume_ssl = true`.

## Rationale

- Coolify provides a full deployment UI (deploy logs, env var management, domain/SSL config) without the complexity of Kubernetes or the vendor lock-in of Heroku.
- Docker Compose is the deployment primitive — same file works for `docker compose up --build` locally.
- No Kamal: Kamal is oriented around SSH-based deploys to raw VMs. Coolify covers the same use case with a better operator UI and built-in reverse proxy.

## Key patterns

- Rails server binds on `0.0.0.0:3000` and is `expose`d (not `ports`-mapped) — Coolify's proxy routes traffic in.
- Health check on `/up` with a generous `start_period` (60s) to allow DB migrations to complete on first boot.
- Worker service shares the same image as `web`; only the `command` differs.
- `RAILS_ENV=production` set in the Compose file (not just via Coolify UI) so it's never missing.

## Consequences

- Deploys are triggered by pushing to the configured branch in Coolify (or manually via the UI).
- Rails console: exec into the running `web` container via Coolify's terminal or `docker exec`.
- No `thruster` needed — Coolify's reverse proxy handles HTTP compression and caching upstream.
