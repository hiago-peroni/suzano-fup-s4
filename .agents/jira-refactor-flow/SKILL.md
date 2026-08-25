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
  version: '1.1.0'
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

## Configuracao

Antes de qualquer operacao no Jira, carregar as regras compartilhadas em
[`references/jira-api.md`](references/jira-api.md). Esse arquivo tem os
comandos curl para todas as operacoes (ler, atribuir, mover, comentar).

> **CRITICO:** o MCP atlassian so tem tools de LEITURA. Todas as operacoes de
> ESCRITA (atribuir, mover, comentar) usam curl com a REST API do Jira. As
> credenciais estao em `~/.config/kilo/kilo.json` (nao `.jsonc`).

## Transicoes do board RFIA

| Coluna | Transition ID | Quando no fluxo |
|--------|---------------|-----------------|
| Backlog | 11 | Task criada, aguardando atribuicao |
| Em analise | 21 | Atribuida + plano sendo criado |
| Fazendo | 31 | Execucao (orchestrator + skills atomicas) |
| Para Testar | 41 | Review + card atualizado com resultado |
| Encerramento | 51 | Decision log finalizado |
| Feito | 61 | Tudo verde (apos testes + UAT + PR — skill separada) |

## Fluxo completo

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ 1. ATRIBUIR │────▶│ 2. EM       │────▶│ 3. FAZENDO  │────▶│ 4. PARA     │
│             │     │    ANALISE  │     │             │     │    TESTAR   │
│ curl PUT    │     │ curl POST   │     │ curl POST   │     │ curl POST   │
│ assignee    │     │ transition  │     │ transition  │     │ transition  │
│             │     │ 21 + plan   │     │ 31 + exec   │     │ 41 + result │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
```

### Passo 0 — Setup (uma vez no inicio)

Carregar as credencias do Jira e o arquivo de regras:

```bash
# Extrair credenciais de ~/.config/kilo/kilo.json
JIRA_USER=$(grep '"JIRA_USERNAME"' ~/.config/kilo/kilo.json | head -1 | sed 's/.*: *"//;s/".*//')
JIRA_TOKEN=$(grep '"JIRA_API_TOKEN"' ~/.config/kilo/kilo.json | head -1 | sed 's/.*: *"//;s/".*//')
JIRA_URL="https://b2rise.atlassian.net"
```

Ler [`references/jira-api.md`](references/jira-api.md) para os comandos curl
completos de cada operacao.

### Passo 1 — Atribuir o card

1. Obter meu accountId:
```bash
curl -s -u "$JIRA_USER:$JIRA_TOKEN" "$JIRA_URL/rest/api/2/myself" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['accountId'])"
```

2. Ler o card para verificar assignee atual:
```bash
curl -s -u "$JIRA_USER:$JIRA_TOKEN" \
  "$JIRA_URL/rest/api/2/issue/{RFIA-XX}?fields=assignee" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['fields']['assignee'])"
```

3. Se ja atribuido a mim, pular. Se atribuido a outro, perguntar ao usuario.
4. Atribuir via curl PUT:
```bash
curl -s -o /dev/null -w "%{http_code}" -u "$JIRA_USER:$JIRA_TOKEN" \
  -X PUT -H "Content-Type: application/json" \
  "$JIRA_URL/rest/api/2/issue/{RFIA-XX}" \
  -d '{"fields":{"assignee":{"accountId":"{ACCOUNT_ID}"}}}'
# Esperar HTTP 204
```

### Passo 2 — Mover para "Em analise" + Planejar

1. Mover o card:
```bash
curl -s -o /dev/null -w "%{http_code}" -u "$JIRA_USER:$JIRA_TOKEN" \
  -X POST -H "Content-Type: application/json" \
  "$JIRA_URL/rest/api/2/issue/{RFIA-XX}/transitions" \
  -d '{"transition":{"id":"21"}}'
# Esperar HTTP 204
```

2. **Carregar a skill `refactor-plan`:** use a ferramenta `skill` com
   `name="refactor-plan"` para carregar suas instrucoes, entao siga o workflow
   dela. O `refactor-plan` vai:
   - Ler o card completo (descricao, done when, depends on)
   - Ler o research.md vinculado (se existir)
   - Ler o AGENTS.md do projeto (se existir)
   - Criar `.refactor-plan.md` no projeto
   - Retornar o plano validado

3. Postar analise inicial no card via curl POST comment. Usar o template de
   comentario humanizado (ver secao "Templates de Comentario" abaixo). O
   comentario DEVE:
   - Incluir o link clicavel do card: `[RFIA-XX](https://b2rise.atlassian.net/browse/RFIA-XX)`
   - Explicar em prosa fluida O QUE os arquivos/changes fazem e POR QUE
   - Nao usar listas quebradas com muitos bullets fragmentados — preferir
     paragrafos continuos que explicam o contexto
   - Listar o "Done when" em checklist (unica lista permitida)

4. **Apresentar o plano para o usuario aprovar.** Use a ferramenta `question`
   com o resumo do plano e opcoes:
   - "Aprovar e executar" — prosseguir para o Passo 3
   - "Revisar plano" — voltar ao `refactor-plan` para ajustar
   - "Cancelar" — parar o fluxo

   > **CRITICO:** NUNCA mover para "Fazendo" ou iniciar a execucao sem a
   > aprovacao explicita do usuario. O padrao e Planejar-Validar-Executar — a
   > validacao inclui a aprovacao humana.

### Passo 3 — Mover para "Fazendo" + Executar (somente apos aprovacao)

1. Mover o card:
```bash
curl -s -o /dev/null -w "%{http_code}" -u "$JIRA_USER:$JIRA_TOKEN" \
  -X POST -H "Content-Type: application/json" \
  "$JIRA_URL/rest/api/2/issue/{RFIA-XX}/transitions" \
  -d '{"transition":{"id":"31"}}'
```

2. **Carregar a skill `refactor-orchestrator`:** use a ferramenta `skill` com
   `name="refactor-orchestrator"` para carregar suas instrucoes, entao siga o
   workflow dela. O `refactor-orchestrator` vai:
   - Ler o `.refactor-plan.md`
   - Chamar a skill atomica correta (move-file, rewrite-logic, dependency-inversion)
   - Rodar lint + typecheck
   - Carregar `refactor-document` para registrar decisoes
   - Carregar `review-implementation` para validar
   - Retornar `success` ou `escalated`

3. Avaliar retorno do orchestrator:
   - `success` → seguir para Passo 4
   - `escalated` → seguir para Passo 4 (erro)

### Passo 4 — Postar resultado + Mover para "Para Testar"

**Se sucesso** — postar resultado humanizado via curl POST comment e mover:

O comentario de resultado DEVE:
- Incluir o link clicavel do card
- Explicar em prosa fluida o QUE foi feito e POR QUE cada arquivo/change
  importa — nao so listar "arquivo X criado"
- Marcar o "Done when" com checklist (unica lista)
- Listar documentacao gerada (decision log, Obsidian, MemPalace)
- Indicar os proximos passos (UAT, PR)

```bash
# Mover para Para Testar
curl -s -o /dev/null -w "%{http_code}" -u "$JIRA_USER:$JIRA_TOKEN" \
  -X POST -H "Content-Type: application/json" \
  "$JIRA_URL/rest/api/2/issue/{RFIA-XX}/transitions" \
  -d '{"transition":{"id":"41"}}'
```

**Se escalado (erro)** — postar diagnostico humanizado e NAO mover:

O comentario de erro DEVE:
- Explicar em prosa o que deu errado e por que
- Nao usar jargao tecnico sem contexto — explicar para um humano entender
- Listar os arquivos afetados
- Sugerir o proximo passo concreto

NUNCA mover para "Para Testar" se escalado — manter em "Fazendo" e escalar.

## Templates de Comentario

### Template — Analise inicial

```markdown
## 1. Análise inicial — [RFIA-XX](https://b2rise.atlassian.net/browse/RFIA-XX)

**Task:** {summary do card}

{paragrafo em prosa fluida explicando O QUE a task faz e POR QUE — qual o
contexto, qual o problema que resolve, como se conecta com tasks anteriores
ou epicos. Minimo 3-5 linhas de explicacao humana.}

**Plano:**
*Tipo:* {tipo} | *Depends on:* {RFIA-YY ou "nenhuma"}

{paragrafo curto explicando a estrategia de execucao — quais arquivos serao
tocados, qual skill atomica sera usada, qual o resultado esperado.}

**Done when:**
- [ ] {criterio 1 do card}
- [ ] {criterio 2 do card}
- [ ] {criterios de verificacao: lint, typecheck, testes}

**Research:** {referencia ao research.md e tasks.md}
```

### Template — Resultado (sucesso)

```markdown
## 2. Execução concluída — [RFIA-XX](https://b2rise.atlassian.net/browse/RFIA-XX)

{paragrafo em prosa fluida explicando o QUE foi feito e POR QUE cada
arquivo/change importa. Nao so "arquivo X criado" — explicar o papel de
cada um no contexto do projeto. Minimo 5-8 linhas.}

**Verificação (Done when):**
- [x] {criterio 1}: {resultado}
- [x] {criterio 2}: {resultado}
- [x] lint: {pass/fail}
- [x] typecheck: {pass/fail}

**Documentação gerada:**
- Decision log: {caminho} ({N} entradas)
- Obsidian: {caminho da nota}
- MemPalace: {drawer ID ou "registrado"}

**Próximos passos:**
1. UAT (testes de usuário)
2. PR (skill separada, após testes + UAT)
```

### Template — Resultado (erro/escalado)

```markdown
## 2. Execução com problemas — [RFIA-XX](https://b2rise.atlassian.net/browse/RFIA-XX)

A execução encontrou um problema que não foi possível resolver
automaticamente após {N} tentativas de recuperação.

**O que aconteceu:**
{paragrafo em prosa explicando o erro de forma que um humano entenda —
qual o sintoma, qual a causa raiz, qual o impacto.}

**Arquivos afetados:**
{lista dos arquivos que foram parcialmente modificados, com o estado atual
de cada um.}

**Próximo passo:**
{acao concreta sugerida — ex: "revisar manualmente o arquivo X na linha Y
e corrigir o tipo Z, depois re-executar a skill".}
```

### Regras de formato dos comentarios

| Regra | Detalhe |
|-------|---------|
| Link do card | SEMPRE incluir `[RFIA-XX](URL)` no header |
| Numeracao | `## 1. Análise inicial`, `## 2. Execução ...` — sequencial |
| Prosa | Explicar O QUE e POR QUE em paragrafos — nao so listar |
| Listas | Usar apenas para "Done when" (checklist) e "Próximos passos" |
| Idioma | Portugues do Brasil, termos tecnicos em ingles quando consagrados |
| Tom | Como explicando para um colega de equipe — humano, nao robotico |
| Tamanho | Analise: 8-15 linhas. Resultado: 12-20 linhas. Erro: 10-15 linhas. |

### Passo 5 — "Feito" (NAO e desta skill)

A transicao para "Feito" (ID 61) e responsabilidade de uma skill separada futura
que verifica testes + UAT + PR antes de mover.

## Gotchas

| # | Armadilha | Como evitar |
|---|-----------|-------------|
| 1 | Card ja atribuido a outro | Verificar `assignee` antes — se diferente, perguntar |
| 2 | curl retorna HTTP nao-204 | Verificar credenciais e transition_id valido |
| 3 | Card depende de outro nao feito | Verificar `Depends on` no plano — se RFIA-YY nao esta "Feito", bloquear |
| 4 | Skill chaining | Use a ferramenta `skill` para CARREGAR instrucoes, nao pseudocode `skill()` |
| 5 | Orchestrator escalado mas card movido | NUNCA mover para "Para Testar" se status = escalated |
| 6 | Project path errado | O `refactor-plan` detecta o path do projeto do card ou pergunta ao usuario |

## Tratamento de Erros (LangGraph)

| Estado | Acao |
|--------|------|
| Card nao encontrado | curl retorna erro 404 — avisar usuario |
| Transition falha (HTTP nao-204) | Verificar que o transition_id e valido para o status atual |
| Orchestrator retorna `escalated` | Postar diagnostico no card, NAO mover, escalar |
| Depends on bloqueado | Postar no card: "Bloqueado por RFIA-YY" e parar |
| Credenciais nao encontradas | Verificar ~/.config/kilo/kilo.json (nao .jsonc) |

## Referencias

- [`references/jira-api.md`](references/jira-api.md) — comandos curl do Jira REST API
- `docs/obsidian/Obsidian/IAtizacao/Skills Research.md` — padrao de skill
- `.kilo/skills/refactor-plan/SKILL.md` — skill de planejamento
- `.kilo/skills/refactor-orchestrator/SKILL.md` — skill de execucao
- `.kilo/skills/refactor-document/SKILL.md` — skill de documentacao
- Board RFIA: https://b2rise.atlassian.net/jira/software/projects/RFIA/boards/2011
