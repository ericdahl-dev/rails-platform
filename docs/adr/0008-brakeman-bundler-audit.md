# Brakeman + Bundler Audit as mandatory security gates

Static security analysis and dependency vulnerability scanning run before every merge.

## Decision

```ruby
gem "brakeman", require: false
gem "bundler-audit", require: false
```

```bash
bundle exec brakeman -q              # static analysis — resolve ALL warnings
bundle exec bundler-audit check --update  # known CVEs in gem dependencies
```

Both are mandatory CI gates. No merge proceeds with unresolved Brakeman warnings or known vulnerable gems.

## Rationale

These two tools together cover the most common Rails security failure modes (XSS, SQLi, mass assignment, CSRF) and supply-chain vulnerabilities. The cost is low; the risk of skipping is not.

## Consequences

- Brakeman false positives must be explicitly annotated with `# brakeman:disable` and a comment explaining why — not silently ignored.
- `bundler-audit check --update` pulls the latest advisory database before checking; CI must have network access for this step (or cache the DB).
