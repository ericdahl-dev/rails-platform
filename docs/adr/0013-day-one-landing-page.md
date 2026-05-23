# 0013 — Day-one themed landing page for early deployability

**Status:** Accepted  
**Apps:** platform baseline

## Decision

Every new app built from this platform must include a branded landing page at the root route (`/`) before or alongside the first deployment.

The landing page should match the app's intended theme and include:

- A clear headline and value proposition
- A primary call to action
- Basic mobile-friendly layout
- Real draft copy aligned with product direction (no generic scaffold/default placeholder page)

If requirements are unclear, the building agent asks for minimum input up front: app name, target audience, value proposition, CTA, and visual tone/branding direction.

## Rationale

The first deploy should always be useful and reviewable by humans, even before core workflows are complete. A themed landing page establishes product intent, validates brand direction, and avoids the "blank scaffold in production" anti-pattern.

This also creates an early, low-risk integration point for layout, styling, and copy decisions that would otherwise be deferred and then rushed later.

## Consequences

- Agents creating new apps must treat a themed landing page as an MVP requirement, not polish.
- Root route ownership is explicit from day one, reducing rework when auth/onboarding flows are added.
- Teams get an immediately presentable URL for stakeholder review after initial deployment.
