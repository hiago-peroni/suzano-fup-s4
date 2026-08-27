---
name: refactor-document
description: >
  Documenta cada refatoracao em tres camadas: decision log (DECISIONS.md no
  projeto), vault Obsidian (notas com wikilinks entre arquivo/padrao/decisao/card)
  e MemPalace (drawers + tunnels). Garante que NENHUM arquivo do escopo deixe
  de ser analisado, documentado e registrado. Mais assertivo que o STATE.md do
  TLC — registra o que mudou, por que, qual padrao seguido e qual card Jira
  originou. Use apos cada execucao do refactor-orchestrator, ou quando o
  usuario disser "documentar refatoracao", "registrar decisao", "atualizar
  decision log", "atualizar vault". NAO use para documentacao funcional completa
  (use functional-docs) nem para ADRs formais (use create-adr).
metadata:
  author: Numen DS
  version: '1.0.0'
  tags: [refatoracao, documentacao, decision-log, obsidian, mempalace]
---

# Refactor Document — Documentacao Assertiva

Registra cada refatoracao em tres camadas para garantir rastreabilidade completa
e que nenhum arquivo deixe de ser analisado.

> **Nenhum arquivo sem analise:** o orchestrator enumera TODOS os arquivos do
> escopo do card. Cada um e marcado como analisado ou dispensado (com motivo).

## Quando usar

- Apos `refactor-orchestrator` concluir a execucao (sucesso OU erro)
- O usuario pede "documentar a refatoracao", "registrar o que foi feito"
- Antes de mover o card para "Para Testar" no Jira

## Escopo

Atua sobre:
- `docs/refactor/DECISIONS.md` no projeto (decision log)
- `docs/obsidian/Obsidian/{projeto}/` (vault Obsidian)
- MemPalace via MCP (drawers + tunnels)

Nao atua sobre:
- Documentacao funcional completa (papel do `functional-docs`)
- ADRs formais com template MADR (papel do `create-adr`)
- STATE.md do TLC (ja existe, esta skill e complementar)

## Tres camadas de documentacao

### Camada 1 — Decision Log (`docs/refactor/DECISIONS.md`)

Arquivo no projeto com tabela cumulativa de decisoes. Mais rastro que STATE.md.

**Estrutura:**
```markdown
# Decision Log — {Projeto}

## Decisoes

| ID | Data | Card | Arquivo | Acao | Porque | Padrao seguido | Status |
|----|------|------|---------|------|--------|----------------|--------|
| D1 | 2026-08-25 | RFIA-25 | persistence/UserRepository.ts | Rename → repositories/ | Dependency rule CA | Clean Architecture §1 | done |
| D2 | 2026-08-25 | RFIA-29 | controllers/UserController.ts | execute() → handle() | Padrao CA | Clean Architecture §5 | done |
| D3 | 2026-08-25 | RFIA-34 | models/UserModel.ts | Anemico → rico | Models ricos | Clean Architecture §3 | done |
```

**Regras:**
- Cada linha = uma acao em um arquivo
- NUNCA apagar linhas anteriores (acumulativo)
- Se uma decisao for revertida, adicionar nova linha com status `reverted`
- Atualizar via `Edit` (acrescentar linhas no fim da tabela)

### Camada 2 — Vault Obsidian (`docs/obsidian/Obsidian/{projeto}/`)

Nota por refatoracao com `[[wikilinks]]` criando relacoes entre arquivos, padroes,
decisoes e cards Jira.

**Estrutura da nota:**
```markdown
Fonte: {arquivo(s) refatorado(s)}
Projeto: {nome do projeto}
Areas: #refatoracao #{tags}
Comum: {o que foi feito}
Tipo: Registro de refatoracao
Relacionado: [[Clean Architecture]], [[Decisoes Arquiteturais]], {wikilinks}

# {RFIA-XX} — {titulo da task}

## O que
{descricao da acao executada}

## Arquivos analisados
| Arquivo | Status | Acao |
|---------|--------|------|
| {path} | ✅ analisado | {mover/reescrever/criar} |
| {path} | ⬜ dispensado | {motivo: ja conforme / fora de escopo} |

## Por que
{racional da decisao, com [[wikilink]] para o padrao seguido}

## Padrao seguido
- [[Clean Architecture]] §{numero} — {regra}

## Card Jira
- [[{RFIA-XX}]] — {summary}

## Decisao
{o que decidiu e por que, com [[wikilinks]] para ADRs se aplicavel}
```

**Wikilinks obrigatorios:**
- `[[Clean Architecture]]` — padrao seguido
- `[[Decisoes Arquiteturais]]` — se ja existe decisao relacionada
- `[[{RFIA-XX}]]` — card Jira (se houver nota do card)

**Regras:**
- Um arquivo por refatoracao OU um arquivo por epico (agrupando tasks)
- Nome do arquivo: `{RFIA-XX}-{slug}.md`
- NENHUM arquivo do escopo deixa de ser listado na seccao "Arquivos analisados"

### Camada 3 — MemPalace (via MCP)

Registrar o conhecimento no MemPalace para consulta semantica futura.

| Acao | MCP call | Quando |
|------|----------|--------|
| Registrar drawer | `mempalace_add_drawer` | Apos cada refatoracao |
| Criar tunnel | `mempalace_create_tunnel` | Se integracao cross-project detectada |
| Diario do agent | `mempalace_diary_write` | Apos sessao completa |

**Drawer:**
- `wing`: nome do projeto (ex: "suzano-s4")
- `room`: "refatoracao"
- `content`: resumo da refatoracao + padrao seguido
- `source_file`: caminho do arquivo refatorado

**Tunnel:**
- Se a refatoracao afeta integracao entre projetos (ex: FUP → EVA, EDI)
- `source_wing`/`target_wing`: nomes dos projetos
- `source_room`/`target_room`: "refatoracao"

## Workflow

### Passo 1 — Coletar informacoes da refatoracao

Ler do contexto:
- `.refactor-plan.md` (o que foi planejado)
- Resultado do `refactor-orchestrator` (o que foi executado)
- `git diff` do que mudou

```
Read("{project-root}/.refactor-plan.md")
bash("cd {project-root} && git diff --name-only")
bash("cd {project-root} && git diff --stat")
```

### Passo 2 — Enumerar TODOS os arquivos do escopo

```
Glob("{caminho do escopo do card}/**/*.ts")
```

Para cada arquivo:
- Marcar como `✅ analisado` se foi modificado
- Marcar como `⬜ dispensado` se nao foi modificado (com motivo)

### Passo 3 — Atualizar Decision Log

```
Read("{project-root}/docs/refactor/DECISIONS.md")  # se existir
# se nao existir, criar
Write/Edit("{project-root}/docs/refactor/DECISIONS.md", adicionar linhas)
```

Para cada arquivo modificado, uma linha na tabela com:
- ID (proximo sequencial)
- Data (hoje)
- Card (RFIA-XX)
- Arquivo (path relativo)
- Acao (mover/reescrever/criar/inverter)
- Porque (racional)
- Padrao seguido (Clean Architecture §X)
- Status (done / reverted / partial)

### Passo 4 — Criar nota no Obsidian

```
Write("{vault-path}/Obsidian/{projeto}/{RFIA-XX}-{slug}.md", notaContent)
```

A nota deve conter:
- YAML frontmatter com `Fonte`, `Projeto`, `Areas`, `Relacionado`
- Secao "Arquivos analisados" com TODOS os arquivos do escopo
- `[[wikilinks]]` para `[[Clean Architecture]]`, `[[Decisoes Arquiteturais]]`
- Card Jira referenciado

### Passo 5 — Registrar no MemPalace

```
mempalace_check_duplicate(content=resumo)
mempalace_add_drawer(wing={projeto}, room="refatoracao", content=resumo, source_file={path})
```

Se houver integracao cross-project:
```
mempalace_create_tunnel(
  source_wing={projeto}, source_room="refatoracao",
  target_wing={projeto integrado}, target_room="refatoracao",
  label="integracao {tipo}"
)
```

Diario do agent:
```
mempalace_diary_write(agent_name="kilo", entry="SESSION:{date}|refactor.{RFIA-XX}|...", topic="refatoracao")
```

### Passo 6 — Verificar completude

| Checklist | Como verificar |
|-----------|----------------|
| Todos os arquivos do escopo listados? | Comparar Glob com a tabela da nota |
| Decision log atualizado? | Read do DECISIONS.md |
| Obsidian tem wikilinks? | Grep por `[[` no arquivo |
| MemPalace registrou? | mempalace_get_drawer confirmar |

### Passo 7 — Retornar

Retornar para o `refactor-orchestrator` ou `jira-refactor-flow`:
- Status: `documented` | `partial` (com gaps)
- Decision log: {caminho} + {quantidade de linhas adicionadas}
- Obsidian: {caminho da nota}
- MemPalace: {drawer ID} + {tunnels criados}
- Arquivos analisados: {total} ({analisados} + {dispensados})

## Gotchas

| # | Armadilha | Como evitar |
|---|-----------|-------------|
| 1 | Arquivo do escopo esquecido na analise | Rodar Glob NO PASSO 2 e comparar com a lista de modificados |
| 2 | Decision log sem padrao seguido | Toda linha DEVE ter "Padrao seguido" referenciando Clean Architecture |
| 3 | Wikilink quebrado no Obsidian | Verificar que o alvo do `[[link]]` existe no vault |
| 4 | MemPalace timeout | Proceder sem MemPalace — Decision Log + Obsidian sao a fonte de verdade |
| 5 | DECISIONS.md nao existe | Criar com header + tabela vazia na primeira execucao |
| 6 | Refatoracao revertida mas log nao atualizado | Adicionar linha com status `reverted` — nunca apagar |

## Tratamento de Erros (LangGraph)

| Estado | Acao |
|--------|------|
| MemPalace indisponivel | Proceder sem — Decision Log + Obsidian sao suficientes |
| Vault Obsidian nao existe | Criar a nota no path do projeto (`docs/obsidian/Obsidian/{projeto}/`) |
| Arquivo grande demais para uma nota | Quebrar em sub-notas por epico, com wikilinks entre elas |
| git diff vazio | Verificar se houve commit — se nao, nao documentar |

## Referencias

- `docs/obsidian/Obsidian/IAtizacao/Clean Architecture.md` — padroes para wikilinks
- `docs/obsidian/Obsidian/IAtizacao/Spec-Driven Development.md` — STATE.md (complementar)
- `docs/obsidian/Obsidian/suzano-s4/Decisoes Arquiteturais.md` — decisoes existentes do projeto
- MemPalace MCP — `mempalace_add_drawer`, `mempalace_create_tunnel`, `mempalace_diary_write`
