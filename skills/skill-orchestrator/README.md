# Skill Orchestrator Agent

An autonomous agent that eliminates the friction of manually invoking skills. It scans your project, indexes available knowledge, and automatically routes the right expertise to every task — so you focus on building, not on tooling.

## The Problem

You have 6+ specialized skills for Swift development: concurrency, testing, SwiftUI, code review, best practices, and more. Each has dozens of reference files. Manually knowing which skill to invoke for each task is cognitive overhead you shouldn't have to carry.

## The Solution

The Skill Orchestrator sits between you and your skills. On every task:

1. **Scans** your project (CLAUDE.md, Package.swift, dependencies)
2. **Classifies** what you're asking for (domain, action, scope)
3. **Routes** to the right skills automatically (often 2-3 at once)
4. **Loads** only the reference files that matter for this specific task
5. **Applies** all knowledge silently — you just get better code
6. **Learns** project patterns to get smarter over time

## How It Works

```
You: "Build a user profile screen with async data loading"

Orchestrator (silently):
  ├── Detects: SwiftUI (ui) + async/await (concurrency) + data flow
  ├── Loads: swiftui-expert-skill → state-management.md, modern-apis.md
  ├── Loads: swift-concurrency → async-await-basics.md
  ├── Reads: CLAUDE.md → MVVM pattern, @Observable, AppColors
  └── Applies: All knowledge combined

You get: Production-quality SwiftUI view with correct state management,
         proper async patterns, following your project's conventions.
```

## Installation

### For Claude Code

```bash
# Copy to your user skills directory
cp -r skill-orchestrator ~/.claude/skills/skill-orchestrator

# Or add as a project skill
cp -r skill-orchestrator .claude/skills/skill-orchestrator
```

### For Claude.ai Projects

Upload the files to your Project's knowledge:
- `SKILL.md` (required)
- `references/skill-registry.md`
- `references/routing-engine.md`
- `references/project-scanner.md`
- `references/knowledge-graph.md`

## What Skills It Orchestrates

The orchestrator works with any skill that follows the standard format (SKILL.md + references/). It ships pre-configured for these Swift skills:

| Skill | Domains | References |
|-------|---------|------------|
| swift-best-practices | API design, naming, Swift 6 | 4 files |
| swift-concurrency | async/await, actors, Sendable | 12 files |
| swift-testing | Test doubles, fixtures, TDD | 8 files |
| swift-testing-expert | @Test, traits, parameterized | 9 files |
| swiftui-expert-skill | State, views, performance | 11 files |
| swift-code-reviewer | Review, security, architecture | 8 files |

**Total**: 52 reference files, automatically routed.

## Language Agnostic

While it ships with deep Swift knowledge, the orchestrator's architecture is language-agnostic:

- **Tech stack detection** supports Swift, TypeScript, Python, Rust, Go, and more
- **Skill routing** adapts automatically when you install new skills
- **CLAUDE.md reading** works for any project type
- **Knowledge graph** captures patterns in any language

To extend to a new language, just install skills for that language. The orchestrator discovers and routes to them automatically.

## Knowledge Graph

The orchestrator builds project intelligence over time in `.claude/knowledge/`:

```
.claude/knowledge/
├── context.yaml     → Auto-generated project scan results
├── patterns.md      → Discovered code patterns and idioms
├── decisions.md     → Architecture decision records (ADRs)
├── skill-gaps.md    → Missing skills with suggestions
└── learnings.md     → Task insights for future reference
```

This knowledge persists across sessions and can be committed to git for team sharing.

## Skill Gap Detection

When the orchestrator encounters a domain with no matching skill, it:

1. Notes the gap in `skill-gaps.md`
2. Applies general best practices as a fallback
3. Suggests what skill would help
4. Offers to create a basic skill from observed patterns

## Architecture

```
skill-orchestrator/
├── SKILL.md                          # Main orchestrator logic
└── references/
    ├── skill-registry.md             # Complete skill catalog
    ├── routing-engine.md             # Task → skill decision trees
    ├── project-scanner.md            # Project context detection
    └── knowledge-graph.md            # Continuous learning system
```

## Example Workflows

### Writing Code
```
You: "Add a settings screen with theme selection"
→ Loads: swiftui-expert (state-management, modern-apis)
→ Reads: CLAUDE.md (design system, navigation pattern)
→ Produces: SwiftUI view with @Observable, AppColors, proper navigation
```

### Code Review
```
You: "Review PR #42"
→ Loads: swift-code-reviewer (review-workflow, all checklists)
→ Scans diff, loads relevant specialist skills per file type
→ Produces: Structured review with severity levels
```

### Fixing Bugs
```
You: "Fix: 'Sending value of non-Sendable type risks data races'"
→ Loads: swift-concurrency (sendable, threading, actors)
→ Identifies isolation boundary crossing
→ Produces: Minimal fix with explanation
```

### Migration
```
You: "Migrate UserService tests from XCTest to Swift Testing"
→ Loads: swift-testing-expert (migration-from-xctest, fundamentals)
→ Reads existing tests for patterns
→ Produces: Migrated tests with @Test, #expect, proper fixtures
```

## Requirements

- Any Claude interface (Claude Code, Claude.ai, API)
- Skills installed in discoverable paths
- Project with CLAUDE.md (optional but recommended)

## Philosophy

- **Zero configuration**: Works out of the box
- **Invisible orchestration**: You never think about skills
- **Project-first**: Your CLAUDE.md rules override everything
- **Additive knowledge**: Skills combine, they don't compete
- **Continuous learning**: Gets smarter with each task

## License

MIT
