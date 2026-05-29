# Claude Pack

Ambiente portátil do Claude Code — skills, agents, settings e CLAUDE.md empacotados e sincronizados entre qualquer Mac via [MCS (Managed Claude Stack)](https://github.com/mcs-cli/mcs).

---

## Como funciona

Os diretórios do Claude Code (`~/.claude/skills`, `~/.claude/agents`, etc.) são symlinks apontando para este repositório. Qualquer skill ou agent instalado localmente cai direto no repo — basta fazer `git push` para propagar para outros computadores.

```
~/.claude/skills         →  ~/claude-pack/skills/
~/.claude/agents         →  ~/claude-pack/agents/
~/.claude/settings.json  →  ~/claude-pack/config/settings.json
~/.claude/CLAUDE.md      →  ~/claude-pack/CLAUDE.md
```

---

## Setup em um novo Mac

### 1. Instalar dependências

```bash
brew install mcs-cli/tap/mcs
```

### 2. Clonar o repositório

```bash
git clone git@github.com:Viniciuscarvalho/Claude-pack.git ~/claude-pack
```

### 3. Criar os symlinks

```bash
rm -rf ~/.claude/skills ~/.claude/agents ~/.claude/CLAUDE.md ~/.claude/settings.json

ln -s ~/claude-pack/skills               ~/.claude/skills
ln -s ~/claude-pack/agents               ~/.claude/agents
ln -s ~/claude-pack/CLAUDE.md            ~/.claude/CLAUDE.md
ln -s ~/claude-pack/config/settings.json ~/.claude/settings.json
```

### 4. Sincronizar via MCS

```bash
mcs pack add Viniciuscarvalho/Claude-pack
mcs sync --global
mcs doctor
```

Pronto. Todos os skills, agents e configurações estão disponíveis.

---

## Sincronizar depois de instalar algo novo

Quando instalar uma nova skill ou agent, o arquivo já cai dentro do repo automaticamente (via symlink). Para propagar para outros Macs:

```bash
cd ~/claude-pack
git add -A
git commit -m "chore: add nova-skill"
git push
```

No outro Mac, basta puxar:

```bash
cd ~/claude-pack && git pull
```

---

## Atualizar o MCS pack

Se alterar o `techpack.yaml` (adicionar MCP servers, hooks, etc.), rode no outro Mac:

```bash
mcs pack update Viniciuscarvalho/Claude-pack
mcs sync --global
```

---

## Estrutura do repositório

```
claude-pack/
├── agents/          # Subagents customizados (.md com frontmatter)
├── skills/          # Skills do Claude Code (cada um com SKILL.md)
├── commands/        # Slash commands customizados
├── config/
│   └── settings.json  # Configurações globais do Claude Code
├── templates/       # Templates de CLAUDE.local.md para projetos
├── CLAUDE.md        # Instruções globais de comportamento
└── techpack.yaml    # Manifesto MCS — define o que é instalado
```

---

## Verificar se tudo está funcionando

```bash
mcs doctor
```

Para corrigir automaticamente qualquer problema:

```bash
mcs doctor --fix
```
