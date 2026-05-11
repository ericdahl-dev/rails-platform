# Propshaft over Sprockets

Use Propshaft as the asset pipeline, not Sprockets.

## Decision

```ruby
gem "propshaft"
```

Paired with:

```ruby
gem "importmap-rails"       # JS via import maps, no bundler
gem "tailwindcss-rails"     # Tailwind via the standalone CLI
```

## Rationale

- Propshaft is simpler — no fingerprinting complexity at the framework level, no asset compilation DSL.
- Import maps eliminate the need for a Node.js build step for JavaScript.
- Tailwind via `tailwindcss-rails` uses the standalone CLI binary, no npm required.
- Sprockets caused friction when mounting engine dashboards (Mission Control Jobs) in Propshaft-based apps.

## Consequences

- No Webpack, no esbuild, no Vite as defaults.
- `bin/rails tailwindcss:watch` in development (via Procfile.dev).
- If a complex JS build is needed later (e.g. TypeScript-heavy components), that is an explicit new decision.
