# Vinicius' iOS Dev Pack

Personal Claude Code setup for iOS/Swift development — skills, agents, slash commands and plugins curated for Swift, SwiftUI, Xcode, Jira and general engineering workflows.

Available on [techpacks.mcs-cli.dev](https://techpacks.mcs-cli.dev).

---

## Instalação rápida

### Pré-requisitos

```bash
brew install mcs-cli/tap/mcs
```

### Adicionar e sincronizar

```bash
mcs pack add Viniciuscarvalho/Claude-pack
mcs sync --global
mcs doctor
```

Pronto. Todos os skills, agents, commands e plugins estarão disponíveis no Claude Code.

---

## O que está incluído

### Plugins
| Plugin | Descrição |
|--------|-----------|
| Atlassian | Jira e Confluence |
| ARS Contexta | Gestão de contexto e notas |
| Claude HUD | Status line com info da sessão |
| Frontend Design | Assistência de design frontend |
| Swift LSP | Language server para Swift |

### MCP Servers
| Servidor | Descrição |
|----------|-----------|
| Cupertino | Documentação e APIs da Apple |

### Skills — Swift & iOS
`swift-best-practices` · `swift-concurrency` · `swift-testing` · `swift-testing-expert` · `swift-code-reviewer` · `swiftui-expert-skill` · `swiftui-ui-patterns` · `swiftui-view-refactor` · `swiftui-performance-audit` · `xcodebuildmcp-cli` · `xcode-build-fixer` · `xcode-build-orchestrator` · `xcode-build-benchmark` · `xcode-compilation-analyzer` · `xcode-project-analyzer` · `spm-build-analysis` · `ios-debugger-agent` · `kotlin-specialist` · `app-store-changelog`

### Skills — Workflow de Engenharia
`tdd` · `code-analyzer` · `feature-marker` · `creating-pr` · `jira-to-pr-workflow` · `enrich-jira-task` · `context-optimization` · `prompt-architect` · `grill-me` · `grill-with-docs` · `find-docs` · `handoff` · `diagnose` · `skill-creator` · `create-skill` · `skill-orchestrator` · `to-prd` · `product-manager` · `remotion-best-practices` · `zellij-specialist`

### Agents
`swift-expert` · `swift-reviewer` · `swiftui-specialist` · `system-architect` · `staff-engineer` · `backend-developer` · `senior-code-reviewer` · `code-reviewer` · `code-refactorer` · `ui-designer` · `premium-ux-designer` · `typescript-pro` · `api-documenter` · `git-commit-helper` · `product-strategy-advisor` · `feature-marker` · `compose-architect` · `datalayer-architect` · `shadcn-*` · `translation-updater`

### Slash Commands
`/commit` · `/commit-fast` · `/create-prd` · `/generate-spec` · `/generate-tasks` · `/add-to-changelog`

---

## Sincronização entre Macs

Este repositório é o source of truth do ambiente. Os diretórios do Claude Code são symlinks que apontam diretamente para cá:

```
~/.claude/skills         →  ~/claude-pack/skills/
~/.claude/agents         →  ~/claude-pack/agents/
~/.claude/settings.json  →  ~/claude-pack/config/settings.json
```

### Setup manual (sem MCS)

```bash
git clone git@github.com:Viniciuscarvalho/Claude-pack.git ~/claude-pack

rm -rf ~/.claude/skills ~/.claude/agents ~/.claude/settings.json

ln -s ~/claude-pack/skills               ~/.claude/skills
ln -s ~/claude-pack/agents               ~/.claude/agents
ln -s ~/claude-pack/config/settings.json ~/.claude/settings.json
```

### Propagar uma nova skill

Qualquer skill instalada cai direto no repo via symlink. Para sincronizar:

```bash
cd ~/claude-pack
git add -A && git commit -m "chore: add nova-skill" && git push
```

No outro Mac:

```bash
cd ~/claude-pack && git pull
# ou via MCS:
mcs pack update Viniciuscarvalho/Claude-pack && mcs sync --global
```

---

## Verificar instalação

```bash
mcs doctor
mcs doctor --fix   # corrige automaticamente
```
