# 0012 — Browser testing with Capybara + Cuprite

**Status:** Accepted  
**Apps:** platform baseline

## Decision

Use **Capybara** for browser-level acceptance/system testing and **Cuprite** as the default JavaScript driver.

```ruby
gem "capybara"
gem "cuprite"
```

```ruby
# spec/support/capybara.rb
Capybara.register_driver :cuprite do |app|
  Capybara::Cuprite::Driver.new(app, window_size: [1400, 1400])
end

Capybara.javascript_driver = :cuprite
```

## Rationale

- Browser-level coverage is needed for future Hotwire flows and JS-enhanced interactions.
- Capybara provides stable, user-focused system test APIs aligned with Rails conventions.
- Cuprite is a modern headless driver (Ferrum/Chromium-based) that avoids Selenium/WebDriver setup complexity for most app cases.

## Consequences

- New apps should include both gems in the test group and configure `:cuprite` as the JS driver.
- CI images must include a Chromium-compatible browser runtime for Cuprite-backed specs.
- Feature/system specs that need JavaScript should use `js: true` and run through the Cuprite driver.
