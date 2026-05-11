# TDD with RSpec + FactoryBot

Test-driven development is the default. RSpec is the test framework; FactoryBot provides test data.

## Decision

```ruby
gem "rspec-rails", "~> 8.0"
gem "factory_bot_rails"
gem "simplecov", require: false
gem "simplecov_json_formatter", require: false
```

## Conventions

- Write the test first (red → green → refactor).
- Specs mirror the `app/` tree under `spec/`.
- FactoryBot for all test data — no fixtures.
- Request specs for API and controller behaviour.
- Unit specs for service objects and models.
- SimpleCov for local coverage reporting only. No remote upload (Codecov, etc.) unless explicitly decided.

## Anti-patterns to avoid

- Shared examples that obscure intent.
- `let` chains that are hard to follow — prefer explicit setup in each example when clarity matters.
- Testing implementation details instead of behaviour.

## Commands

```bash
bundle exec rspec                   # full suite
bundle exec rspec spec/models/      # models only
bundle exec rspec spec/requests/    # request specs
bundle exec rspec spec/path/to/file_spec.rb  # single file
```
