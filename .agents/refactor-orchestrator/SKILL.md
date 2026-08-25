---
name: refactor-orchestrator
description: >
  Orquestra a execucao de refatoracoes lendo o .refactor-plan.md e delegando
  para a skill atomica correta (refactor-move-file, refactor-rewrite-logic,
  refactor-dependency-inversion). Apos execucao, roda lint + typecheck, chama
  refactor-document para registrar decisoes, e chama review-implementation
  para validar. Se review encontrar criticos, entra em recovery: re-planeja,
  re-executa e re-testa. Se passar 2x em falha, escala para o usuario.
  Use quando o jira-refactor-flow delegar a execucao, ou quando o usuario
  disser "executar refatoracao", "orquestrar refactor", "rodar o orchestrator".
  NAO use para planejar (use refactor-plan) nem para gerenciar o card Jira
  (use jira-refactor-flow).
metadata:
  author: Numen DS
  version: '1.0.0'
  tags: [refatoracao, orchestrator, clean-architecture, recovery]
---

# Refactor Orchestrator — Orquestracao de Execucao

Ponto central da execucao. Le o plano, chama a skill atomica, valida, documenta
e faz review. Gerencia o ciclo de sucesso e erro (recovery).

> **Principio:** o orchestrator decide o QUE executar baseado no plano. As skills
> atomicas sabem COMO executar. O orchestrator nao escreve codigo — ele coordena.

## Quando usar

- O `jira-refactor-flow` delega a fase de execucao apos o planejamento
- O usuario pede "executar a refatoracao" ou "rodar o orchestrator"

## Escopo

Atua sobre:
- `.refactor-plan.md` (criado pelo `refactor-plan`)
- Skills atomicas: `refactor-move-file`, `refactor-rewrite-logic`, `refactor-dependency-inversion`
- `review-implementation` (skill existente, reusada)
- `refactor-document` (documentacao)

Nao atua sobre:
- Planejamento (papel do `refactor-plan`)
- Lifecycle no Jira (papel do `jira-refactor-flow`)

## Workflow

```
┌──────────────┐
│ Ler o plano  │  .refactor-plan.md
└──────┬───────┘
       │
       ▼
┌──────────────────────┐
│ Para cada acao:      │
│  chamar skill atomica│  move-file | rewrite-logic | dependency-inversion
└──────┬───────────────┘
       │
       ▼
┌──────────────┐     falhou?     ┌──────────────┐
│ lint + type  │────────────────│ classificar  │
│ check        │                │ erro         │
└──────┬───────┘                └──────┬───────┘
       │                               │
       │ passou                        │
       ▼                               ▼
┌──────────────┐              ┌──────────────┐
│ refactor-    │              │ CORRECAO:    │
│ document     │              │ re-plan +    │
│ (registrar)  │              │ re-executar  │
└──────┬───────┘              └──────┬───────┘
       │                             │
       ▼                             │
┌──────────────┐                     │
│ review-      │── criticos? ────────┘
│ implement.   │
└──────┬───────┘
       │
       │ zero criticos
       ▼
┌──────────────┐
│ Retornar OK  │  → jira-refactor-flow move para "Para Testar"
└──────────────┘
```

### Passo 1 — Ler o plano

```
Read("{project-root}/.refactor-plan.md")
```

Extrair do plano:
- Tipo de acao (move-file | rewrite-logic | dependency-inversion | config | mixed)
- Lista de arquivos
- Camadas de origem e destino
- Done when (criterios de aceite)

### Passo 2 — Delegar para a skill atomica

| Tipo no plano | Skill chamada | Como |
|---------------|---------------|------|
| `move-file` | `refactor-move-file` | Skill tool |
| `rewrite-logic` | `refactor-rewrite-logic` | Skill tool |
| `dependency-inversion` | `refactor-dependency-inversion` | Skill tool |
| `config` | Executar direto | Bash (tsconfig/eslint) |
| `mixed` | Chamadar em ordem: move → rewrite → DI | Sequencial |

Para `mixed`, executar nesta ORDEM:
1. `refactor-move-file` primeiro (mover arquivos para a camada certa)
2. `refactor-rewrite-logic` segundo (corrigir logica na nova camada)
3. `refactor-dependency-inversion` terceiro (criar interfaces/factories)

> **Sub-agentes:** para tasks com [P] (paralelas) no plano, delegar cada
> sub-task a um sub-agente via `Task` tool. O orchestrator principal consolida.

### Passo 3 — Validar (lint + typecheck)

Apos cada skill atomica concluir:

```
cd "{service-dir}" && yarn lint && yarn tsc --noEmit
```

Se falhar:
- Ler o output do erro
- Classificar (import quebrado | tipo incompatible | naming | config)
- Se for erro estrutural que a skill atomica deveria ter tratado → retornar para a skill
- Se for erro novo introduzido → corrigir diretamente

### Passo 4 — Documentar (refactor-document)

```
skill("refactor-document")  # registra em DECISIONS.md + Obsidian + MemPalace
```

O `refactor-document` registra:
- Decision log com cada arquivo modificado
- Nota no vault Obsidian com wikilinks
- Drawer no MemPalace

### Passo 5 — Review (review-implementation)

```
skill("review-implementation")  # 6 subagentes: security, requirements, tests, architecture, regression, performance
```

O `review-implementation` valida contra:
- spec.md e tasks.md (se existirem)
- Clean Architecture rules
- Padroes de codigo do projeto

### Passo 6 — Avaliar resultado do review

| Resultado do review | Acao |
|---------------------|------|
| Zero criticos | Retornar `success` para `jira-refactor-flow` |
| Criticals encontrados | Entrar em RECOVERY (Passo 7) |
| Apenas warnings/suggestions | Retornar `success` com lista de warnings |

### Passo 7 — RECOVERY (se criticos)

Se `review-implementation` encontrar criticals:

1. **Classificar o erro:**

| Tipo | Causa provavel | Acao |
|------|----------------|------|
| bug de logica | Refatoracao quebrou comportamento | `refactor-rewrite-logic` para corrigir |
| violacao CA | Dependency rule violada | `refactor-move-file` ou `refactor-dependency-inversion` |
| regressao | Comportamento existente quebrou | Reescrever a parte que regrediu |
| teste falha | Teste esperava comportamento antigo | Atualizar teste OU corrigir regressao |

2. **Re-planear:** acrescentar secao `## CORRECAO` no `.refactor-plan.md`:
   ```markdown
   ## CORRECAO (tentativa N)
   - Erro: {descricao}
   - Tipo: {classificacao}
   - Arquivo: {path}
   - Acao corretiva: {o que fazer}
   ```

3. **Re-executar** a skill atomica correta para a correcao

4. **Re-validar** (lint + typecheck + review-implementation)

5. **Contador de tentativas:**
   - Tentativa 1: recovery normal
   - Tentativa 2: recovery normal
   - Tentativa 3: ESCALAR para o usuario com diagnostico completo

### Passo 8 — Retornar resultado final

Retornar para o `jira-refactor-flow`:

**Sucesso:**
```
Status: success
Arquivos modificados: [lista]
Skills chamadas: [move-file, rewrite-logic, ...]
Review: zero criticos, {N} warnings
Documentacao: DECISIONS.md + {nota Obsidian} + {drawer MemPalace}
Tentativas de recovery: {N}
```

**Falha escalada:**
```
Status: escalated
Arquivos modificados: [lista parcial]
Erro: {descricao do critical nao resolvido}
Tentativas: 3 (todas falharam)
Diagnostico: {analise do que deu errado}
Proximo passo: revisar manualmente o arquivo {path}
```

## Gotchas

| # | Armadilha | Como evitar |
|---|-----------|-------------|
| 1 | Skill atomica falha silenciosamente | Verificar status de retorno de cada skill |
| 2 | Lint passa mas typecheck falha | Rodar AMBOS — lint E typecheck |
| 3 | Review encontra bug que nao e da refatoracao | Filtrar apenas criticals relacionados ao diff da refatoracao |
| 4 | Recovery entra em loop infinito | Limite de 3 tentativas — depois escalar |
| 5 | Config task pulada | Se plano tem tipo `config`, executar tsconfig/eslint diretamente |
| 6 | Sub-tarefa paralela [P] executada serial | Se plano marca [P], usar Task tool para paralelizar |

## Tratamento de Erros (LangGraph)

| Estado | Acao |
|--------|------|
| Skill atomica retorna `partial` | Verificar o que falta — se e da mesma skill, re-chamar |
| Lint falha apos 3 correcoes | Escalar — problema estrutural mais profundo |
| Review-implementation indisponivel | Proceder sem review mas avisar: "Review pulado" |
| .refactor-plan.md nao existe | Escalar: "Plano nao encontrado — rodar refactor-plan primeiro" |

## Referencias

- `docs/obsidian/Obsidian/IAtizacao/Skills Research.md` — padrao de orchestracao com sub-agentes
- `docs/obsidian/Obsidian/IAtizacao/Clean Architecture.md` — regras que as skills atomicas aplicam
- `.kilo/skills/review-implementation/SKILL.md` — skill de review (reusada)
- `.kilo/skills/refactor-plan/SKILL.md` — skill que cria o plano
- `.kilo/skills/refactor-document/SKILL.md` — skill que documenta
