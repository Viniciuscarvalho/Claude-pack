# Vinicius' iOS Dev Pack

Personal Claude Code setup for iOS/Swift development — skills, agents, slash commands and plugins curated for Swift, SwiftUI, Xcode, Jira and general engineering workflows.

Available on [techpacks.mcs-cli.dev](https://techpacks.mcs-cli.dev).

---

## Quick Start

### Prerequisites

```bash
brew install mcs-cli/tap/mcs
```

### Add and sync

```bash
mcs pack add Viniciuscarvalho/Claude-pack
mcs sync --global
mcs doctor
```

All skills, agents, commands and plugins will be available in Claude Code.

---

## What's Included

### Plugins
| Plugin | Description |
|--------|-------------|
| Atlassian | Jira and Confluence integration |
| ARS Contexta | Context management and note-taking |
| Claude HUD | Status line with session info |
| Frontend Design | Frontend design assistance |
| Swift LSP | Swift language server integration |

### MCP Servers
| Server | Description |
|--------|-------------|
| Cupertino | Apple documentation and APIs |

### Skills — Swift & iOS
`swift-best-practices` · `swift-concurrency` · `swift-testing` · `swift-testing-expert` · `swift-code-reviewer` · `swiftui-expert-skill` · `swiftui-ui-patterns` · `swiftui-view-refactor` · `swiftui-performance-audit` · `xcodebuildmcp-cli` · `xcode-build-fixer` · `xcode-build-orchestrator` · `xcode-build-benchmark` · `xcode-compilation-analyzer` · `xcode-project-analyzer` · `spm-build-analysis` · `ios-debugger-agent` · `kotlin-specialist` · `app-store-changelog`

### Skills — Engineering Workflow
`tdd` · `code-analyzer` · `feature-marker` · `creating-pr` · `jira-to-pr-workflow` · `enrich-jira-task` · `context-optimization` · `prompt-architect` · `grill-me` · `grill-with-docs` · `find-docs` · `handoff` · `diagnose` · `skill-creator` · `create-skill` · `skill-orchestrator` · `to-prd` · `product-manager` · `remotion-best-practices` · `zellij-specialist`

### Agents
`swift-expert` · `swift-reviewer` · `swiftui-specialist` · `system-architect` · `staff-engineer` · `backend-developer` · `senior-code-reviewer` · `code-reviewer` · `code-refactorer` · `ui-designer` · `premium-ux-designer` · `typescript-pro` · `api-documenter` · `git-commit-helper` · `product-strategy-advisor` · `feature-marker` · `compose-architect` · `datalayer-architect` · `shadcn-*` · `translation-updater`

### Slash Commands
`/commit` · `/commit-fast` · `/create-prd` · `/generate-spec` · `/generate-tasks` · `/add-to-changelog`

---

## Syncing Between Machines

This repository is the source of truth for the entire Claude Code environment. The Claude Code directories are symlinks pointing directly here:

```
~/.claude/skills         →  ~/claude-pack/skills/
~/.claude/agents         →  ~/claude-pack/agents/
~/.claude/settings.json  →  ~/claude-pack/config/settings.json
```

### Adding a new skill or agent

Because the Claude Code directories are symlinks, any skill or agent installed locally lands directly inside this repo. To propagate it to other machines, just commit and push:

```bash
cd ~/claude-pack
git add -A && git commit -m "chore: add new-skill" && git push
```

> **Important:** If you want the new skill to be installable via `mcs sync` (for new machines or other people), you also need to add a component entry to `techpack.yaml`. Without it, the files exist in the repo but MCS won't know to install them.

```yaml
- id: skill-new-skill
  displayName: New Skill
  description: What it does
  skill:
    source: skills/new-skill
    destination: new-skill
```

### Sync scenarios

| Scenario | What to do |
|----------|------------|
| Your own Macs (already set up with symlinks) | `git push` / `git pull` — that's it |
| New Mac from scratch | `mcs pack add` + `mcs sync --global` — requires `techpack.yaml` to be up to date |
| Someone else using your pack | Same as above |

**In short:** `git` syncs the files between your own machines. `techpack.yaml` is what makes the pack installable for anyone else via MCS.

---

## New Machine Setup

### Via MCS (recommended)

```bash
brew install mcs-cli/tap/mcs
mcs pack add Viniciuscarvalho/Claude-pack
mcs sync --global
mcs doctor
```

### Manual setup with symlinks

If you want to contribute back to the pack from this machine (push new skills/agents), clone the repo and create symlinks so Claude Code points directly to it:

```bash
git clone git@github.com:Viniciuscarvalho/Claude-pack.git ~/claude-pack

rm -rf ~/.claude/skills ~/.claude/agents ~/.claude/settings.json

ln -s ~/claude-pack/skills               ~/.claude/skills
ln -s ~/claude-pack/agents               ~/.claude/agents
ln -s ~/claude-pack/config/settings.json ~/.claude/settings.json
```

---

## Verify Installation

```bash
mcs doctor
mcs doctor --fix   # auto-fix issues
```
