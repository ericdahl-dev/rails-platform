# 0009 — Pundit for authorization

**Status:** Accepted  
**Apps:** evs-mapper

## Decision

Use **Pundit** for access control. Authorization logic lives in plain Ruby policy classes under `app/policies/`, one per model.

```ruby
gem "pundit"
```

```ruby
# app/controllers/application_controller.rb
include Pundit::Authorization
after_action :verify_authorized, except: :index
after_action :verify_policy_scoped, only: :index
```

Each policy inherits from `ApplicationPolicy` and overrides only the actions that differ from the default (deny all). `policy_scope` is used in index actions to return only records the current user is permitted to see.

## Rationale

- Policy classes are plain Ruby objects — easy to unit-test without a request stack.
- Explicit `verify_authorized` / `verify_policy_scoped` after_actions catch controllers that forget to authorize.
- Co-located with the model they govern; readable by non-Rails developers.
- Preferred over CanCanCan, which uses a single `Ability` file that grows unwieldy as the model surface expands.

## Consequences

- Every controller action must call `authorize @record` or `policy_scope(Model)`, or explicitly skip verification.
- `ApplicationPolicy` defaults should deny access so new resources fail closed until a policy is written.
- Policy specs live in `spec/policies/` and should cover each role × action combination.
