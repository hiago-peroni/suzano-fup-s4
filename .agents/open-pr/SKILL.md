---
name: open-pr
description: >
  Abre Pull Requests para refatoracoes seguindo o modelo padrao da empresa
  (pr-model.md). Le o diff do git, o decision log e a
  documentacao gerada, e constroi um PR description estruturado com: Overview,
  Definition of Done, Problem Overview, Data Flow, Test Script, System Impact e
  Code Changes. Usa gh CLI para criar o PR. Use apos o jira-refactor-flow mover
  o card para "Para Testar" e os testes + UAT passarem, ou quando o usuario
  disser "abrir PR", "criar pull request", "abre PR para quality".
metadata:
  author: Numen DS
  version: '1.0.0'
  tags: [refatoracao, pr, pull-request, github]
---

# Open PR — Pull Request padronizado

Abre PRs seguindo o modelo padrao da empresa (`./references/pr-model.md`). O PR description
e estruturado em seccoes fixas que explicam o que mudou, por que, como testar
e qual o impacto — nao apenas um lista de commits.

> **Principio:** um PR bom explica O QUE e POR QUE para um revisor humano que
> nao tem contexto do card Jira ou do research. O PR e autonomo.

## Quando usar

- Apos `jira-refactor-flow` mover o card para "Para Testar"
- Apos testes automatizados e UAT passarem
- O usuario pede "abrir PR", "criar pull request", "abre PR para quality"

## Escopo

Atua sobre:
- Git diff do branch atual contra o branch base (quality ou main)
- `docs/refactor/DECISIONS.md` (decision log)
- Notas no Obsidian (se existirem)
- Card do Jira (para contexto)

Nao atua sobre:
- Lifecycle no Jira (papel do `jira-refactor-flow`)
- Execucao da refatoracao (papel do `refactor-orchestrator`)

## Pre-requisitos

- Git: branch atual com commits, remote configurado
- `gh` CLI: autenticada (`gh auth status`)
- Branch base: `quality` (padrao da empresa) ou a que o usuario especificar
- Testes + UAT: devem ter passado antes de abrir o PR

## Estrutura do PR (modelo pr-model.md)

O PR description segue esta estrutura fixa:

```
## Overview + Definition of Done
## Problem Overview (Issue Description + Root Cause)
## Data Flow (Input Sources + Output Formats + Key Components)
## Test Script (testes numerados)
## System Impact (Business + Technical + Integration Points)
## Code Changes (Modified Files + Code Snippets before/after)
```

## Workflow

### Passo 1 — Coletar contexto

```bash
# Branch atual e branch base
git branch --show-current
git remote show origin | sed -n '/HEAD branch/s/.*: //p' || echo quality

# Diff completo contra o branch base
git diff --stat quality...HEAD
git diff --name-only quality...HEAD

# Diff de working tree (se houver uncommitted)
git diff --name-only
git ls-files --others --exclude-standard

# Decision log (se existir)
cat docs/refactor/DECISIONS.md
```

### Passo 2 — Construir o PR body

Montar o PR body seguindo o template abaixo. Cada seccao deve ter:

#### Overview
Uma frase que resume o que o PR faz. Direta, sem jargon.

#### Definition of Done
Checklist dos criterios de aceite (do "Done when" do card Jira + criterios
de verificacao como lint, typecheck, testes).

#### Problem Overview
- **Issue Description:** paragrafo explicando o problema que motivou a change
- **Root Cause:** paragrafo explicando a causa raiz (se aplicavel — se for
  scaffold/criacao, explicar o que faltava antes)

#### Data Flow
- **Input Sources:** de onde vem os dados/triggers
- **Output Formats:** o que e produzido/alterado
- **Key Components:** arquivos/chaves principais envolvidos

#### Test Script
Testes numerados que o revisor pode seguir. Cada teste:
```
### Test N: {titulo}
1. {passo}
2. {passo}
Expected: {resultado esperado}
```

#### System Impact
- **Business Impact:** como afeta o usuario/negocio
- **Technical Impact:** o que muda tecnicamente
- **Integration Points:** pontos de integracao afetados (se nenhum, dizer "none")

#### Code Changes
- **Modified Files:** lista de arquivos modificados com descricao breve
- **Code Snippets:** before/after dos trechos chave (nao todo o diff, so o
  que importa para o revisor entender)

#### Footer
```
---
Pull Request opened by Kilo (AI) with guidance from the PR author
```

### Passo 3 — Commitar (se houver uncommitted)

Se houver arquivos uncommitted:
```bash
git add {arquivos}
git commit -m "{acao}: {descrição em até 80 caracteres}"
```

Formato: `<ação>: <descrição em até 80 caracteres>`. Ações permitidas: `feat`,
`fix`, `docs`, `chore`, `refactor`, `test`, `ci`, `build`, `perf`. Sem scope
entre parenteses. Se cobrir um card RFIA-XX, incluir no final: `feat: create base
structure [RFIA-17]`.

### Passo 4 — Push

```bash
git push -u origin {branch-atual}
```

Se o branch nao existe no remote, `-u` cria o upstream.

### Passo 5 — Abrir o PR

```bash
gh pr create \
  --base {branch-base} \
  --head {branch-atual} \
  --title "{titulo-do-pr}" \
  --body-file {arquivo-temp-com-body.md}
```

O titulo segue o padrao: `<ação>: <descrição em até 80 caracteres>`.
Se o PR cobre um card RFIA-XX, incluir o ID: `feat: create base structure [RFIA-17]`.

### Passo 6 — Atualizar o Jira

Apos o PR ser aberto:
```bash
# Postar comentario no Jira com a URL do PR
curl -s -u "$JIRA_USER:$JIRA_TOKEN" \
  -X POST -H "Content-Type: application/json" \
  "$JIRA_URL/rest/api/2/issue/{RFIA-XX}/comment" \
  -d '{"body":"## 3. Pull Request aberto — [RFIA-XX](URL)\n\nPR: {PR_URL}\n\n**Branch:** {branch-atual} → {branch-base}\n\n**Status:** Aguardando review."}'
```

## Template de PR body

```markdown
# {tipo-emoji} Overview
{uma frase que resume o PR}

### Definition of Done

- [x] {criterio 1}
- [x] {criterio 2}
- [x] lint: pass
- [x] typecheck: pass

## Problem Overview

### Issue Description
{paragrafo explicando o problema}

### Root Cause
{paragrafo explicando a causa raiz — se scaffold/criacao, explicar o que faltava}

## Data Flow

### Input Sources
- **{fonte 1}**: {descricao}

### Output Formats
- **{output 1}**: {descricao}

### Key Components
- **{arquivo 1}**: {papel}

## Test Script

### Test 1: {titulo}
1. {passo}
2. {passo}
Expected: {resultado}

## System Impact

### Business Impact
- {impacto 1}

### Technical Impact
- {impacto 1}

### Integration Points
- {ponto 1 ou "none"}

## Code Changes

### Modified Files
- `{arquivo 1}` — {descricao da change}
- `{arquivo 2}` — {descricao da change}

### Code Snippets

#### {arquivo} — Before/After
```javascript
// BEFORE
{codigo antigo}
```
```javascript
// AFTER
{codigo novo}
```

---
Pull Request opened by Kilo (AI) with guidance from the PR author
```

## Regras de formato

| Regra | Detalhe |
|-------|---------|
| Titulo | `<ação>: <descrição em até 80 caracteres>` — ex: `feat: add dark mode toggle` |
| Base branch | `quality` (padrao) ou a especificada pelo usuario |
| Body | Segue o template acima integralmente — todas as seccoes |
| Test Script | Minimo 1 teste, idealmente 2-3 cobrindo os caminhos principais |
| Code Snippets | Before/after dos trechos chave — nao todo o diff |
| Footer | "Pull Request opened by Kilo (AI) with guidance from the PR author" |
| Idioma | Ingles para titulo e code; PT-BR ou ingles para body (escolher um e manter) |

## Gotchas

| # | Armadilha | Como evitar |
|---|-----------|-------------|
| 1 | PR body muito longo | O body deve ser escaneavel — seccoes curtas, nao paragrafas enormes |
| 2 | Diff muito grande para snippet | Mostrar apenas os trechos chave do before/after, nao o diff completo |
| 3 | gh CLI nao autenticada | Rodar `gh auth status` antes de tentar criar o PR |
| 4 | Branch nao pushed | Sempre `git push -u` antes de `gh pr create` |
| 5 | PR sem testes | Se nao houver testes automatizados, explicar no Test Script que e manual |
| 6 | Card Jira nao atualizado | Apos criar o PR, sempre postar a URL no Jira |

## Tratamento de Erros (LangGraph)

| Estado | Acao |
|--------|------|
| gh CLI nao instalada | Instalar ou usar curl com GitHub API |
| Branch base nao existe | Verificar `git branch -r` — se quality nao existe, perguntar |
| PR ja existe | `gh pr list --head {branch}` — se ja existe, atualizar em vez de criar |
| Push falha | Verificar permissoes do remote — pode ser necessario force-push |

## Referencias

- `/home/hiagoperoni/Projects/Numen-IA/numen-mcps/docs-exemplo/pr-model.md` — modelo original do PR
- `docs/obsidian/Obsidian/IAtizacao/Git Flow Padrao.md` — fluxo de branches
- `.kilo/skills/jira-refactor-flow/SKILL.md` — skill que move o card para "Para Testar"
- `.kilo/skills/refactor-plan/SKILL.md` — skill que cria o plano
- `.kilo/skills/refactor-document/SKILL.md` — skill que documenta a refatoracao
