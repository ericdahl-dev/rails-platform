# 0011 — Organization model from day one for future multi-tenancy

**Status:** Accepted  
**Apps:** evs-mapper

## Decision

Every app that may eventually serve multiple customers includes an `Organization` model from the first migration, even when the MVP ships with a single hard-coded organization.

The ownership hierarchy flows through org:

```
Organization → top-level resource (e.g. Facility) → child resources
```

All models that belong to a top-level resource are transitively scoped to an Organization. Controllers scope every query through the org from day one.

The MVP seeds exactly one `Organization` row. There is no signup flow, no org-switching UI, and no per-org billing surface — just the model, the foreign keys, and the query scope.

## Rationale

Retrofitting multi-tenancy into an existing schema requires:
- Adding `organization_id` to every table (migrations across the full model surface)
- Updating every query to scope through the org (every controller, every export, every background job)
- Writing and testing data-isolation logic retrospectively

This is typically a 2–3 day refactor that touches the entire codebase. Adding the model on day one costs ~2 hours and leaves the schema correct without building any multi-tenant product surface.

## Consequences

- Every model beneath the top-level resource gets `organization_id` transitively via foreign keys.
- `seeds.rb` creates one `Organization` row; dev/test factories assume this.
- `current_organization` helper (hardcoded to the seeded org in MVP) is the only place the org lookup lives — swap to session/subdomain resolution later without touching controllers.
- Multi-org UI (signup, org switching, per-org billing) is explicitly deferred.
