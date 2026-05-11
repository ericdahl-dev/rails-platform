# Domain Docs

How engineering skills should consume this repo's documentation.

## Layout

Single-context repo. No `CONTEXT.md` — this is a pure platform decisions repo, not a product codebase.

```
/
├── PLATFORM.md        ← full narrative: rationale, config snippets, decisions
├── BASELINE.md        ← quick-reference checklist for new apps
├── README.md          ← stack overview and ADR index
└── docs/adr/          ← individual Architecture Decision Records (0001–0011)
```

## Before working on an issue, read these

- **`BASELINE.md`** — the quick-reference stack table and gotchas checklist
- **`PLATFORM.md`** — the relevant section(s) for the area you're changing
- **`docs/adr/`** — any ADR that touches the decision area

## Adding a new decision

1. Add or update the relevant section in `PLATFORM.md` (narrative + rationale).
2. Update the stack table and gotchas checklist in `BASELINE.md` if applicable.
3. Write a new ADR in `docs/adr/` following the existing numbering and format.
4. Update the ADR index table in both `PLATFORM.md` and `README.md`.

## No CONTEXT.md by design

There is no domain glossary here — the ADRs are the authoritative record. If a skill looks for `CONTEXT.md`, proceed silently without it.
