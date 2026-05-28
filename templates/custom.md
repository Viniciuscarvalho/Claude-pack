# CLAUDE.md — Global Coding Preferences

Senior iOS developer. Prioritise consistency, readability, and explicit code over cleverness.
Local `CLAUDE.md` overrides these defaults per project.

---

## Formatting
- Formatter must pass before review. Swift → SwiftFormat (`.swiftformat` at repo root); other languages use equivalent (Prettier, Black, gofmt)
- No inline formatter overrides — fix the rule
- Max line length: 120 chars · Trailing commas in multi-line collections · One statement per line

## Naming
- Descriptive, full words — `userIdentifier` not `usrId`
- Booleans: `is/has/should/can/did` prefix · Methods: verb-first (`fetchUser`, `validateInput`)
- No noise words (`Manager`, `Helper`, `Utils`) · Constants at narrowest possible scope

## Architecture
- SOLID as baseline · Composition over inheritance · Depend on abstractions, not concretions
- Unidirectional data flow · Value types preferred · Files > ~300 lines signal too many responsibilities
- **Patterns:** Coordinator (navigation), Repository (data), Factory/Builder (construction), Strategy (behaviour), Observer/Combine (events)

## Structure
Feature-first grouping, mirrored test tree:
```
Features/<n>/   →  View, ViewModel, Coordinator, Service
Core/           →  Networking, Persistence, Extensions
Tests/Features/ →  mirrors source tree
```

## Testing
- **TDD default** — Red → Green → Refactor
- Naming: `test_method_condition_expectedOutcome()`
- AAA structure (Arrange / Act / Assert) · One logical concept per test · No logic in tests
- Mocks in `TestDoubles/` · High domain coverage; pragmatic UI coverage
- Awkward-to-test code = design problem, not a testing problem

## Dependencies
- Prefer stdlib/platform APIs · New deps require justification · Pin versions explicitly
- Wrap third-party SDKs behind internal abstractions — never leak into domain code

## Git
- Atomic commits · Conventional format: `type(scope): summary`
- Small, focused PRs · No commented-out code on main

## Documentation & Comments
- Public APIs need doc comments · Comments explain **why**, not what
- `TODO: [TICKET-123]` required · `FIXME:` never committed to main — fix or file a ticket

## Principles
- Fail fast · Make invalid states unrepresentable · Don't optimise prematurely
- Delete dead code · Codebase consistency > personal preference

---