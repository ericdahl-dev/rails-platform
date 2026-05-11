# Rails + Hotwire monolith; no separate frontend

A single Rails app owns the control plane, web UI, webhooks, and background jobs. Hotwire (Turbo + Stimulus) handles the UI. No separate API or frontend framework.

## Context

Both toaster (originally Rails API + Next.js) and voice-assistant (Rails 8 + Hotwire from the start) converged on this architecture. Toaster's split introduced CORS complexity, same-site cookie friction, a two-origin authentication model, and two deployment units — all before the product shape was stable enough to justify the API surface.

## Decision

Start every new app as a Rails + Hotwire monolith.

## Consequences

- Single process, single deployment, auth cookies just work.
- No CORS configuration.
- Hotwire (Turbo Streams over Action Cable) covers real-time UI requirements.
- A separate React or native frontend can be extracted later if a native mobile app or third-party integration makes a public API worthwhile. That is a new explicit decision, not a default.
