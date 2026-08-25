---
name: refactor-plan
description: >
  Cria plano de ação temporario (.refactor-plan.md) antes de qualquer refatoracao.
  Le o card do Jira (RFIA-XX), o research.md vinculado e o AGENTS.md do projeto,
  bate o "Done when" do card contra o plano, e so entao libera para execucao.
  Use quando o usuario disser "planejar refatoracao", "criar plano", "refactor plan",
  "analisar card RFIA", "antes de executar", ou quando o jira-refactor-flow delegar
  a fase de planejamento. NAO use para executar a refatoracao (use refactor-orchestrator)
  nem para documentar (use refactor-document).
metadata:
  author: Numen DS
  version: '1.0.0'
  tags: [refatoracao, planejamento, jira, plan-validate-execute]
---

# Refactor Plan — Planejar antes de Executar

Cria um arquivo temporario `.refactor-plan.md` na raiz do projeto com o plano de
acao completo. O plano e validado contra o "Done when" do card do Jira e o
`research.md` antes de liberar para o `refactor-orchestrator`.

> **Padrao Planejar-Validar-Executar:** o plano e escrito, validado contra
> referencias, e so entao passado adiante. Nunca executar sem validar.

## Quando usar

- O `jira-refactor-flow` delega a fase de planejamento apos atribuir o card
- O usuario pede "planejar a refatoracao do RFIA-XX"
- Antes de qualquer execucao do `refactor-orchestrator`

## Escopo

Atua sobre:
- Projetos SAP S4 da Suzano (ou qualquer projeto com vault Obsidian IAtizacao)
- Cards do board RFIA (ou board equivalente de refatoracao)
- Arquivos `docs/researches/<slug>/research.md` como referencia

Nao atua sobre:
- Execucao da refatoracao em si (papel do `refactor-orchestrator`)
- Documentacao pos-refatoracao (papel do `refactor-document`)
- Review da implementacao (papel do `review-implementation`)

## Workflow

### Passo 1 — Ler o card do Jira

```
atlassian_jira_get_issue(issue_key="{RFIA-XX}", fields="*all")
```

Extrair do card:

| Campo | Onde no card | Uso |
|-------|-------------|-----|
| Summary | `summary` | Titulo da task |
| Description | `description` | Seccoes: O que, Onde, Depends on, Done when, Detalhes |
| Epic link | `issuelinks` ou custom field | Epico pai (EP-XX) |
| Priority | `priority.name` | Prioridade |
| Status | `status.name` | Status atual (deve estar "Em analise") |

### Passo 2 — Ler o research vinculado

O campo "Detalhes" do card referencia `docs/researches/<slug>/research.md`.

Se o research existir:
```
Read("docs/researches/<slug>/research.md")
```

Se NAO existir:
- Sinalizar GAP no plano
- Continuar com o que esta no card (o "Done when" e a fonte minima)

### Passo 3 — Ler o AGENTS.md do projeto

```
Read("{project-root}/AGENTS.md")
```

Se houver AGENTS.md em subpasta mais proxima do codigo alvo, ler tambem
(proximity-based: o mais proximo prevalece).

### Passo 4 — Classificar a acao

Baseado no "O que" e "Onde" do card, classificar o tipo:

| Tipo | Condicao | Skill atomica |
|------|----------|---------------|
| `move-file` | Mover arquivo de uma pasta/camada para outra | `refactor-move-file` |
| `rewrite-logic` | Reescrever logica interna de funcao/classe | `refactor-rewrite-logic` |
| `dependency-inversion` | Criar interface + impl + factory | `refactor-dependency-inversion` |
| `config` | Padronizar tsconfig, eslint, CI | Direto (sem skill atomica) |
| `mixed` | Mais de um tipo acima | Orquestrar em ordem |

### Passo 5 — Escrever o plano

Criar `{project-root}/.refactor-plan.md` com a estrutura:

```markdown
# Refactor Plan — {RFIA-XX}

> Temporario. Deletar apos execucao. Validar antes de executar.

## Task
- **Card:** RFIA-XX — {summary}
- **Epico:** RFIA-XX (EP-NN: {epic summary})
- **Research:** docs/researches/{slug}/research.md §{secao}
- **Depends on:** {RFIA-YY ou "none"}

## Acao
- **Tipo:** {move-file | rewrite-logic | dependency-inversion | config | mixed}
- **Skill atomica:** {refactor-move-file | refactor-rewrite-logic | ...}

## Arquivos
| Arquivo | Camada atual | Camada destino | Acao |
|---------|-------------|----------------|------|
| {path} | {domain/application/...} | {domain/...} | {mover/reescrever/criar} |

## Validacao
- [ ] {Done when criterio 1 do card}
- [ ] {Done when criterio 2 do card}
- [ ] lint passa
- [ ] typecheck passa
- [ ] Dependencias resolvidas ({Depends on})

## Erro recovery
Se review-implementation encontrar critico:
  -> classificar erro (bug logica | violacao CA | regressao | teste falha)
  -> acrescentar seccao CORRECAO neste plano
  -> re-executar via refactor-orchestrator
  -> se falhar 2x: escalar para o usuario com diagnostico
```

### Passo 6 — Validar o plano

Antes de liberar para execucao, bater cada item do "Done when" do card contra
o plano:

| Done when do card | Coberto no plano? | Como |
|-------------------|-------------------|------|
| {criterio 1} | ✅/❌ | {secao do plano} |
| {criterio 2} | ✅/❌ | {secao do plano} |

Se algum criterio NAO estiver coberto:
- Adicionar ao plano
- Se nao for possivel cobrir, marcar como GAP e escalar para o usuario

### Passo 7 — Retornar para o orquestrador

O plano esta pronto. Retornar:
- Status: `ready` | `gap` (com lista de gaps)
- Caminho: `{project-root}/.refactor-plan.md`
- Proximo passo: `refactor-orchestrator` para execucao

## Gotchas

| # | Armadilha | Como evitar |
|---|-----------|-------------|
| 1 | Card sem "Done when" estruturado | Se o campo description nao tiver seccao "Done when", perguntar ao usuario os criterios de aceite |
| 2 | Research nao existe para o slug | Sinalizar no plano como GAP mas continuar — o "Done when" do card e a fonte minima |
| 3 | Depends on nao resolvido | Se Depends on referencia outro card RFIA-YY que nao esta "Feito", bloquear e escalar |
| 4 | Multiplos arquivos sem escopo claro | Se >20 arquivos, quebrar em sub-tasks e pedir confirmacao do usuario |
| 5 | AGENTS.md do projeto diverge do vault | O AGENTS.md do projeto prevalece (proximity-based) |

## Tratamento de Erros (LangGraph)

| Estado | Acao |
|--------|------|
| Card nao encontrado no Jira | Escalar: "Card RFIA-XX nao encontrado" |
| Research existe mas slug diverge | Usar o slug do card; se ambiguo, perguntar |
| Plano cobre <50% do Done when | Escalar com lista de gaps |
| Depends on bloqueado | Escalar: "Depende de RFIA-YY que esta em {status}" |

## Referencias

- `docs/obsidian/Obsidian/IAtizacao/Skills Research.md` — padrao de skill (estrutura, plan-validate-execute)
- `docs/obsidian/Obsidian/IAtizacao/Spec-Driven Development.md` — workflow TLC
- `docs/specs/features/<slug>/tasks.md` — tasks detalhadas (se existir)
