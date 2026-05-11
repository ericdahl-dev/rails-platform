# rails-platform — Agent Instructions

This repo is a shared architectural baseline for Rails 8 apps. It contains no runnable code — only decision docs, ADRs, and opinionated defaults.

## Useful Commands

```bash
# No build/test/lint — docs only
# All changes go through PRs (branch protection on main)
gh issue list
gh issue view <number> --comments
```

## Agent Skills

### Issue tracker

Issues live in GitHub Issues on `ericdahl-dev/rails-platform`. See `docs/agents/issue-tracker.md`.

### Triage labels

Default canonical labels (`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`). See `docs/agents/triage-labels.md`.

### Domain docs

Single-context repo with no `CONTEXT.md` by design. Authoritative docs are `PLATFORM.md`, `BASELINE.md`, and `docs/adr/`. See `docs/agents/domain.md`.

## Adding or Updating Decisions

1. Update the relevant section in `PLATFORM.md` (narrative + code snippets).
2. Update stack table / gotchas checklist in `BASELINE.md` if applicable.
3. Write a new ADR in `docs/adr/` (next sequential number, same format as existing).
4. Update the ADR index table in both `PLATFORM.md` and `README.md`.
5. Open a PR — direct pushes to `main` are blocked.

## Auth Note

Repo lives under `ericdahl-dev` org; push/PR as `Skeyelab`. If `gh` commands fail with permission errors, prefix with `GITHUB_TOKEN=$(gh auth token --user Skeyelab)`.
