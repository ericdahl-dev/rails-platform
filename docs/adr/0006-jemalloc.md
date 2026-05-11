# jemalloc via LD_PRELOAD in Docker

Enable jemalloc in all production Docker images for reduced memory usage and lower allocation latency.

## Decision

In the Dockerfile `base` stage:

```dockerfile
RUN apt-get install --no-install-recommends -y libjemalloc2 && \
    ln -s /usr/lib/$(uname -m)-linux-gnu/libjemalloc.so.2 /usr/local/lib/libjemalloc.so

ENV LD_PRELOAD="/usr/local/lib/libjemalloc.so"
```

## Rationale

Ruby's default allocator (glibc malloc) retains virtual memory pages aggressively, causing container memory to grow over time even when live objects are small. jemalloc returns memory to the OS more aggressively and reduces fragmentation. The overhead is negligible and the memory behavior is strictly better for long-running Puma/GoodJob processes.

Both voice-assistant and toaster use this pattern.

## Consequences

- Docker image includes `libjemalloc2` (~500KB).
- `LD_PRELOAD` applies to all processes in the container (web and worker).
- No application code changes required.
