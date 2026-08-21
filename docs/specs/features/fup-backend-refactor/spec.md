# FUP Backend Refactor Specification

## Problem Statement

O backend FUP S4 (`suzano-fup-panel-backend-s4`) tem 3 serviços CAP (application-service, processing-service, public-service) + db/ + app-router em um repositório separado. Uma auditoria arquivo a arquivo identificou **148 violações** do standard `cap-clean-architecture` (32 critical, 44 high), incluindo 8 sites de SQL injection, 28 arquivos em `domain/` importando framework, 21 arquivos com bug de recursão infinita no getter `message()`, e 53 barrel exports proibidos. O código está em produção — a refatoração deve preservar comportamento.

## Goals

- [ ] Backend 100% conforme `cap-clean-architecture` standard (zero violations critical/high)
- [ ] Zero sites de SQL injection (todos queries usam bound parameters)
- [ ] 7 classes de erro canônicas em todos os 3 serviços (400/401/403/404/409/500/503)
- [ ] Zero imports de `@models` ou `@sap/cds` em `domain/`
- [ ] Zero pastas proibidas (`extractors/`, `formatters/`, `hidrators/`, `i18n/`, `persistence/`)
- [ ] Zero barrel exports (exceto `domain/errors/index.ts`)
- [ ] Controllers usando `handle()` ao invés de `execute()`
- [ ] tsconfig strict + eslint printWidth 180 + NodeNext
- [ ] `main/routes/public.ts` sem SQL inline nem business logic
- [ ] Funcionalidade de produção preservada

## Out of Scope

| Feature | Reason |
|---------|--------|
| Models anêmicos → ricos (class + with() + validate()) | P2 — exige mover validação de use cases, trabalho manual extenso (M2) |
| Extrair código compartilhado `shared/` package | P3 — exige reconciliar 4 arquivos drifted entre serviços (M2) |
| Testes unitários completos | P2 — P1 inclui apenas smoke test de build (D8) |
| Frontend refactoring | Fase posterior (M3) |
| MTA unificado com frontend | Fase posterior (M4) |
| yarn workspaces | Decisão D2: sem workspaces |
| NodeNext + .js extensions em imports | P2 — migrar module/moduleResolution é breaking change (EP-08) |

---

## User Stories

### P1: EP-01 — Scaffold do monorepo ⭐ MVP

**User Story**: Como dev, quero o monorepo `suzano-fup-s4` com `packages/backend/` contendo os 3 serviços + db + 2 app-routers, para ter uma estrutura base padronizada.

**Why P1**: Sem o scaffold, nenhuma outra refatoração pode começar — é o foundation.

**Acceptance Criteria**:

1. WHEN `yarn install` na raiz THEN system SHALL instalar dependências sem erro
2. WHEN `yarn bd` na raiz THEN system SHALL executar `mbt build` + `cf deploy` do MTA unificado
3. WHEN dev abre o repo THEN system SHALL ter `AGENTS.md` raiz (≤150 linhas) + `packages/backend/AGENTS.md`
4. WHEN CI roda em PR para `quality` THEN system SHALL executar lint + typecheck com paths-filter
5. WHEN `git-flow-action` dispara THEN system SHALL usar `NUMENDS/git-flow-action` (não VAEES)

**Independent Test**: `yarn bd` executa sem erro e o MTA deploya em Cloud Foundry.

**Jira**: RFIA-3

---

### P1: EP-04 — Corrigir SQL injection ⭐ MVP

**User Story**: Como dev, quero eliminar os 8 sites de SQL injection usando parâmetros bound, para que o backend não seja vulnerável a injeção de SQL.

**Why P1**: Vulnerabilidade de segurança ativa em produção.

**Acceptance Criteria**:

1. WHEN repository executa query com input de usuário THEN system SHALL usar bound parameters (`?` ou `cds.ql`) — nunca string interpolation
2. WHEN grep por `${` em `.ts` com `WHERE`/`IN`/`LIKE` THEN system SHALL retornar zero resultados
3. WHEN query com `IN (${array})` THEN system SHALL passar array como parâmetro bound, não string interpolada
4. WHEN funcionalidade testada THEN system SHALL produzir mesmo resultado que antes (preservar comportamento)

**Independent Test**: Rodar queries com inputs especiais (aspas, ponto-e-vírgula) e verificar que não causam SQL injection.

**Jira**: RFIA-6

---

### P1: EP-07 — Padronizar error taxonomy ⭐ MVP

**User Story**: Como dev, quero 7 classes de erro canônicas (400/401/403/404/409/500/503) com o bug de recursão infinita fixado, para que o tratamento de erros seja confiável.

**Why P1**: Bug ativo de recursão infinita no getter `message()` (bomba relógio) + missing `ConflictError` (409).

**Acceptance Criteria**:

1. WHEN error class é instanciada THEN system SHALL ter exatamente 7 classes: BadRequest(400), Unauthorized(401), Forbidden(403), NotFound(404), Conflict(409), Server(500), UnavailableService(503)
2. WHEN `errorInstance.message` é acessado THEN system SHALL retornar a mensagem sem recursão infinita (usar `super(message)`)
3. WHEN grep por `Forbiden` THEN system SHALL retornar zero (renomeado para `Forbidden`)
4. WHEN `BadGatewayError` ou `UnprocessableEntity` são buscados THEN system SHALL não existir (dead code removido)
5. WHEN `SAPGatewayError` ou `DellavolpeError` são buscados THEN system SHALL estar em `domain/adapters/external-api/` (não em `domain/errors/`)
6. WHEN ADR é criado THEN system SHALL justificar `UnavailableServiceError` (503) como 7ª classe

**Independent Test**: Instanciar cada error class, acessar `.message`, verificar que não há stack overflow.

**Jira**: RFIA-9

---

### P1: EP-02 — Eliminar framework do domain ⭐ MVP

**User Story**: Como dev, quero remover todos os imports de `@models/db/models` de `domain/` (28 arquivos), para que o domain seja puro TypeScript sem dependência de framework.

**Why P1**: Violação critical da regra 7 do standard (Sem framework no domain). O domain deve ser puro TS com apenas `@sweet-monads/either`.

**Acceptance Criteria**:

1. WHEN grep por `@models` em `domain/` THEN system SHALL retornar zero resultados
2. WHEN grep por `@sap/cds` em `domain/` THEN system SHALL retornar zero resultados
3. WHEN repository interface declara tipos THEN system SHALL usar namespace do próprio contrato (`XxxRepository.DbRow`) ao invés de importar `@models`
4. WHEN use case `Params`/`Result` types são declarados THEN system SHALL viver em `domain/use-cases/<type>/<name>.ts` (não em `domain/models/`)
5. WHEN `tsc --noEmit` roda THEN system SHALL passar sem erro em todos os 3 serviços

**Independent Test**: `grep -r "@models" domain/` retorna vazio + `tsc --noEmit` passa.

**Jira**: RFIA-4

---

### P1: EP-12 — Mover business logic de public.ts ⭐ MVP

**User Story**: Como dev, quero mover o SQL inline + business logic de `main/routes/public.ts:15-44` para repository + controller + use case, para que a rota seja apenas binding (composition root).

**Why P1**: Violação critical da dependency rule — `main/` não tem business logic nem SQL.

**Acceptance Criteria**:

1. WHEN `main/routes/public.ts` é lido THEN system SHALL não conter SQL inline nem business logic
2. WHEN rota `CREATE NotificationLogs` é disparada THEN system SHALL delegar para controller via factory
3. WHEN repository executa query THEN system SHALL usar bound parameters (não string interpolation)
4. WHEN use case retorna THEN system SHALL retornar `Either<AbstractError, T>`
5. WHEN funcionalidade testada THEN system SHALL criar NotificationLogs igual ao comportamento anterior

**Independent Test**: Criar uma NotificationLog via API e verificar que o registro é criado corretamente.

**Jira**: RFIA-14

---

### P2: EP-09 — Rename persistence→repositories + entities→entity-events

**User Story**: Como dev, quero renomear `infrastructure/persistence/` → `infrastructure/repositories/` e `entities/` → `entity-events/`, para que a estrutura de pastas siga o standard canônico.

**Why P2**: Rename mecânico (find-replace + atualizar imports), mas não é critical para segurança ou bugs.

**Acceptance Criteria**:

1. WHEN grep por `persistence` em `src/` THEN system SHALL retornar zero (renomeado para `repositories/`)
2. WHEN grep por `/entities/` em `src/` THEN system SHALL retornar zero (renomeado para `entity-events/`)
3. WHEN `tsc --noEmit` roda THEN system SHALL passar sem erro

**Independent Test**: `tsc --noEmit` passa em todos os 3 serviços.

**Jira**: RFIA-11

---

### P2: EP-05 — Eliminar pastas proibidas

**User Story**: Como dev, quero remover `extractors/`, `formatters/`, `hidrators/` (typo), `i18n/` e mover conteúdo para `utils/`, para que não haja pastas anti-padrão.

**Why P2**: Trabalho mecânico (rename + mover + atualizar imports), não é critical.

**Acceptance Criteria**:

1. WHEN find por `extractors/` em `src/` THEN system SHALL retornar zero
2. WHEN find por `formatters/` em `src/` THEN system SHALL retornar zero
3. WHEN find por `hidrators/` em `src/` THEN system SHALL retornar zero (renomeado para `hydrators/` com 'y')
4. WHEN find por `i18n/` em `infrastructure/` THEN system SHALL retornar zero (movido para `utils/translator/i18n/`)
5. WHEN `tsc --noEmit` roda THEN system SHALL passar

**Independent Test**: `find src/ -name "extractors" -o -name "formatters" -o -name "hidrators"` retorna vazio.

**Jira**: RFIA-7

---

### P2: EP-06 — Eliminar barrel exports

**User Story**: Como dev, quero eliminar os 53 arquivos `index.ts` que re-exportam, para que os imports sejam diretos por arquivo.

**Why P2**: Trabalho mecânico, mas volumoso (53 files + atualizar todos os imports).

**Acceptance Criteria**:

1. WHEN grep por `export *` em `index.ts` THEN system SHALL retornar apenas `domain/errors/index.ts` (única exceção permitida)
2. WHEN imports são verificados THEN system SHALL apontar para arquivo específico (não para pasta)
3. WHEN `tsc --noEmit` roda THEN system SHALL passar

**Independent Test**: `grep -r "export \*" src/ --include="index.ts"` retorna apenas `domain/errors/index.ts`.

**Jira**: RFIA-8

---

### P2: EP-10 — Controllers execute→handle

**User Story**: Como dev, quero renomear o método público de `execute` para `handle` em todos os controllers, para que siga o standard.

**Why P2**: Trabalho mecânico (find-replace), não é critical.

**Acceptance Criteria**:

1. WHEN grep por `execute(` em `presentation/controllers/` THEN system SHALL retornar zero
2. WHEN controller é instanciado THEN system SHALL ter método `handle()` público
3. WHEN callers (factories, routes) são verificados THEN system SHALL chamar `handle()` (não `execute()`)
4. WHEN `tsc --noEmit` roda THEN system SHALL passar

**Independent Test**: `grep -r "execute" src/presentation/controllers/` retorna vazio.

**Jira**: RFIA-12

---

### P2: EP-08 — Padronizar tsconfig + eslint

**User Story**: Como dev, quero alinhar tsconfig (strict) e eslint (printWidth 180, no-console allow error) ao standard, para que a configuração seja consistente.

**Why P2**: Config changes que geram diff massivo mas não quebram funcionalidade.

**Acceptance Criteria**:

1. WHEN tsconfig é lido THEN system SHALL ter `strictNullChecks: true`, `noImplicitAny: true`, `strictPropertyInitialization: true`
2. WHEN eslint config é lido THEN system SHALL ter `printWidth: 180`, `no-console: ['error', { allow: ['error'] }]`, `no-explicit-any: 'error'`
3. WHEN `.nvmrc` é lido THEN system SHALL conter `22`
4. WHEN `renovate.json` é lido THEN system SHALL ter `baseBranches: [quality]` + patch-only
5. WHEN `tsc --noEmit` roda THEN system SHALL passar (pode exigir fixes de tipo)

**Independent Test**: `tsc --noEmit` passa com strict mode + `eslint` passa com printWidth 180.

**Jira**: RFIA-10

---

## Edge Cases

- WHEN query SQL tem array dinâmico de IDs THEN system SHALL usar `cds.ql` com `.in()` ao invés de string interpolation
- WHEN error class é extendida por subclasse customizada THEN system SHALL não override `message` getter/setter
- WHEN rename de pasta quebra import relativo THEN system SHALL atualizar todos os imports afetados
- WHEN barrel elimination quebra import que usava `from '@/domain/repositories'` THEN system SHALL mudar para `from '@/domain/repositories/purchase-order-item.js'`
- WHEN `strictNullChecks: true` revela null errors existentes THEN system SHALL fixar com narrowing (`if (x !== null)`) ou `?.` operator

---

## Requirement Traceability

| Requirement ID | Story | Phase | Status |
|----------------|-------|-------|--------|
| FBR-01 | P1: EP-01 Scaffold | Design | Pending |
| FBR-02 | P1: EP-04 SQL injection | Design | Pending |
| FBR-03 | P1: EP-07 Error taxonomy | Design | Pending |
| FBR-04 | P1: EP-02 Domain purity | Design | Pending |
| FBR-05 | P1: EP-12 Business logic public.ts | Design | Pending |
| FBR-06 | P2: EP-09 Rename pastas | - | Pending |
| FBR-07 | P2: EP-05 Pastas proibidas | - | Pending |
| FBR-08 | P2: EP-06 Barrel exports | - | Pending |
| FBR-09 | P2: EP-10 execute→handle | - | Pending |
| FBR-10 | P2: EP-08 tsconfig+eslint | - | Pending |

**ID format:** `FBR-[NUMBER]` (FUP Backend Refactor)

**Status values:** Pending → In Design → In Tasks → Implementing → Verified

**Coverage:** 10 total, 0 mapped to tasks, 10 unmapped ⚠️

---

## Success Criteria

- [ ] Zero violations critical/high de cap-clean-architecture no backend (audit re-run retorna 0)
- [ ] Zero sites de SQL injection (grep retorna vazio)
- [ ] 7 classes de erro canônicas em todos os 3 serviços
- [ ] Zero imports de `@models` em `domain/`
- [ ] `yarn bd` (build + deploy MTA) executa sem erro
- [ ] Funcionalidade de produção preservada (todas as OData endpoints funcionam igual)
- [ ] CI gate verde (lint + typecheck + smoke test)

---

## Jira Board

- Board: https://b2rise.atlassian.net/jira/software/projects/RFIA/boards/2011
- 14 épicos criados: RFIA-3 a RFIA-16
- Ordem de execução: EP-01 → EP-04 → EP-07 → EP-02 → EP-12 → EP-09 → EP-05 → EP-06 → EP-10 → EP-08
