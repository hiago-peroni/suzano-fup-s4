---
name: jira-refactor-flow
description: >
  Gerencia o lifecycle completo de um card de refatoracao no Jira (board RFIA).
  Fluxo: atribuir card → mover para "Em analise" → ler task + research → postar
  analise inicial → mover para "Fazendo" → delegar para refactor-orchestrator →
  postar resultado + decisoes → mover para "Para Testar". NAO abre PR — isso
  sera uma skill separada apos testes + UAT. Use quando o usuario disser
  "trabalhar no card RFIA-XX", "executar refatoracao do Jira", "lifecycle do
  card", "processar card do board". NAO use para executar a refatoracao em si
  (use refactor-orchestrator) nem para planejar (use refactor-plan).
metadata:
  author: Numen DS
  version: '1.0.0'
  tags: [refatoracao, jira, lifecycle, board, rfia]
---

# Jira Refactor Flow — Lifecycle no Board

Gerencia o card do Jira do inicio ao fim. O card SEMPRE esta atualizado com o
status e o que foi feito. Nao abre PR — isso e responsabilidade de uma skill
separada, apos testes + UAT.

> **Regra absoluta:** o card Jira e a fonte de verdade do status. Toda
> transicao de estado no fluxo corresponde a uma transicao no Jira. Se o Agent
> faz algo, o card reflete.

## Quando usar

- O usuario pede "trabalhar no RFIA-XX", "processar o card", "lifecycle do card"
- O usuario quer automatizar o fluxo: atribuir → analisar → executar → testar

## Escopo

Atua sobre:
- Board RFIA (ID: 2011) no Jira
- Cards RFIA-3 a RFIA-48 (14 epicos + 32 tasks)
- Transicoes do board

Nao atua sobre:
- Execucao da refatoracao (papel do `refactor-orchestrator`)
- Planejamento (papel do `refactor-plan`)
- PR (skill separada futura)

## Configuracao do board RFIA

| Coluna | Transition ID | Quando no fluxo |
|--------|---------------|-----------------|
| Backlog | 11 | Task criada, aguardando atribuicao |
| Em analise | 21 | Atribuida + plano sendo criado |
| Fazendo | 31 | Execucao (orchestrator + skills atomicas) |
| Para Testar | 41 | Review + card atualizado com resultado |
| Encerramento | 51 | Decision log finalizado |
| Feito | 61 | Tudo verde (apos testes + UAT + PR) |

**MCP:** `atlassian` (Jira + Confluence). Tools: `atlassian_jira_*`.

## Fluxo completo

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ 1. ATRIBUIR │────▶│ 2. EM       │────▶│ 3. FAZENDO  │────▶│ 4. PARA     │
│             │     │    ANALISE  │     │             │     │    TESTAR   │
│ update_issue│     │ transition │     │ transition  │     │ transition  │
│ + assignee  │     │ 21 + plan  │     │ 31 + exec   │     │ 41 + result │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
                                                                    │
                                                                    ▼
                                                            ┌─────────────┐
                                                            │ 5. FEITO    │
                                                            │ (skill      │
                                                            │  separada)  │
                                                            └─────────────┘
```

### Passo 1 — Atribuir o card

```
atlassian_jira_get_user_profile(user_identifier="hiago.peroni@numenit.com")
atlassian_jira_update_issue(issue_key="{RFIA-XX}", fields={"assignee": "hiago.peroni@numenit.com"})
```

Se o usuario ja estiver atribuido, pular.

### Passo 2 — Mover para "Em analise" + Planejar

```
atlassian_jira_transition_issue(issue_key="{RFIA-XX}", transition_id="21")
```

Delegar para `refactor-plan`:
```
skill("refactor-plan")  # le card + research + AGENTS.md → cria .refactor-plan.md
```

Apos o plano:
```
atlassian_jira_add_comment(issue_key="{RFIA-XX}", comment="""
## Analise inicial — {RFIA-XX}

### Task
{summary do card}

### Plano
- **Tipo:** {move-file | rewrite-logic | dependency-inversion | mixed}
- **Arquivos:** {quantidade}
- **Depends on:** {RFIA-YY ou none}

### Done when
- [ ] {criterio 1}
- [ ] {criterio 2}
- [ ] lint + typecheck passando

### Research
{referencia ao research.md}
""")
```

### Passo 3 — Mover para "Fazendo" + Executar

```
atlassian_jira_transition_issue(issue_key="{RFIA-XX}", transition_id="31")
```

Delegar para `refactor-orchestrator`:
```
skill("refactor-orchestrator")  # le .refactor-plan.md → chama skills atomicas → valida → documenta → review
```

O orchestrator retorna:
- `success` com lista de arquivos + resultado do review
- `escalated` com diagnostico do erro

### Passo 4 — Postar resultado + Mover para "Para Testar"

**Se sucesso:**
```
atlassian_jira_add_comment(issue_key="{RFIA-XX}", comment="""
## Execucao concluida — {RFIA-XX}

### O que foi feito
- {acao 1}: {arquivo} — {detalhe}
- {acao 2}: {arquivo} — {detalhe}

### Decisoes tomadas
- {decisao 1}: {porque} — [[Clean Architecture §X]]
- {decisao 2}: {porque}

### Verificacao
- [x] lint: pass
- [x] typecheck: pass
- [x] review-implementation: zero criticos, {N} warnings

### Documentacao
- Decision log: {caminho}
- Obsidian: {caminho da nota}
- MemPalace: {drawer ID}

### Proximos passos
1. Testes automatizados
2. UAT (testes de usuario)
3. PR (skill separada)
""")
atlassian_jira_transition_issue(issue_key="{RFIA-XX}", transition_id="41")
```

**Se escalado (erro):**
```
atlassian_jira_add_comment(issue_key="{RFIA-XX}", comment="""
## Execucao com problemas — {RFIA-XX}

### Status
Escalado apos 3 tentativas de recovery.

### Erro
{descricao do critical nao resolvido}

### Diagnostico
{analise do que deu errado}

### Arquivos afetados
{lista de arquivos parcialmente modificados}

### Proximo passo
Revisar manualmente o arquivo {path} e re-executar.
""")
```

Nao mover para "Para Testar" se escalado — manter em "Fazendo" e escalar para o usuario.

### Passo 5 — Mover para "Feito" (skill separada futura)

A transicao para "Feito" (transition_id=61) NAO e feita por esta skill. sera feita
por uma skill separada que:
1. Verifica que testes automatizados passaram
2. Verifica que UAT (testes de usuario) passaram
3. Abre PR
4. Aguarda merge
5. Move para "Feito"

## Tabela de transicoes

| Passo | De → Para | Transition ID | Pre-condicao | MCP call |
|-------|-----------|---------------|-------------|----------|
| 1 | (any) → Em analise | 21 | Card atribuido | `transition_issue` |
| 2 | Em analise → Fazendo | 31 | Plano criado + analise postada | `transition_issue` |
| 3 | Fazendo → Para Testar | 41 | Orchestrator success + review zero criticos | `transition_issue` |
| 4 | Para Testar → Feito | 61 | Testes + UAT + PR mergeado (skill separada) | `transition_issue` |

## Gotchas

| # | Armadilha | Como evitar |
|---|-----------|-------------|
| 1 | Card ja atribuido a outro | Verificar `assignee` antes de atribuir — se diferente, perguntar ao usuario |
| 2 | Transition ID invalido | Sempre usar `get_transitions` antes de `transition_issue` se nao tiver mapeado |
| 3 | Card depende de outro nao feito | Verificar `Depends on` no plano — se RFIA-YY nao esta "Feito", bloquear |
| 4 | Comentario muito longo | Jira aceita Markdown — manter conciso com seccoes curtas |
| 5 | Orchestrator escalado mas card movido | NUNCA mover para "Para Testar" se status = escalated |
| 6 | Mover para "Feito" prematuro | Esta skill NAO move para "Feito" — apenas ate "Para Testar" |

## Tratamento de Erros (LangGraph)

| Estado | Acao |
|--------|------|
| Card nao encontrado | `get_issue` retorna erro — avisar usuario |
| Transition falha | Verificar que o transition_id e valido para o status atual |
| Orchestrator retorna `escalated` | Postar diagnostico no card, NAO mover, escalar para usuario |
| Depends on bloqueado | Postar no card: "Bloqueado por RFIA-YY ({status})" e parar |
| MCP atlassian indisponivel | Nao conseguir ler/mover o card — avisar usuario e parar |

## Referencias

- `docs/obsidian/Obsidian/IAtizacao/Skills Research.md` — padrao de skill
- `docs/obsidian/Obsidian/IAtizacao/Spec-Driven Development.md` — workflow TLC
- `.kilo/skills/refactor-plan/SKILL.md` — skill de planejamento
- `.kilo/skills/refactor-orchestrator/SKILL.md` — skill de execucao
- `.kilo/skills/refactor-document/SKILL.md` — skill de documentacao
- Board RFIA: https://b2rise.atlassian.net/jira/software/projects/RFIA/boards/2011
