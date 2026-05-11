# Postgres for queue, cache, and cable; no Redis

All background job, cache, and Action Cable adapters use Postgres. Redis is not in the stack.

## Decision

| Concern | Gem | Adapter |
|---|---|---|
| Background jobs | `good_job` | Postgres |
| Cache | `solid_cache` | Postgres |
| Action Cable | `solid_cable` | Postgres |

## Rationale

- Single infrastructure dependency (Postgres) rather than two.
- Neon (managed Postgres) is already required for the database; no additional service to operate.
- GoodJob, Solid Cache, and Solid Cable are all production-proven at the traffic levels these apps target.

## Consequences

- Operational simplicity: one database service to monitor, back up, and pay for.
- Redis can be added later if Solid Cache/Cable become performance bottlenecks, but that is an explicit future decision.
