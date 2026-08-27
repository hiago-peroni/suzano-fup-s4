# FUP Backend Refactor — Tasks

**Spec**: `docs/specs/features/fup-backend-refactor/spec.md`
**Research**: `docs/researches/fup-backend-refactor/research.md`
**Status**: Draft

---

## Execution Plan

### Phase 1: Foundation (Sequential)

Scaffold do monorepo + migração dos 3 serviços para `packages/backend/`. Tudo depende deste.

```
T1 → T2 → T3
```

### Phase 2: Critical Fixes (Parallel after T3)

Segurança + bugs ativos. Independentes entre si por serviço.

```
     ┌→ T4 [P]  (APP SQL injection)
     │
T3 ──┼→ T5 [P]  (PROC SQL injection)
     │
     ├→ T6 [P]  (PUB SQL injection)
     │
     ├→ T7 [P]  (Error taxonomy — all 3 services)
     │
     └→ T8 [P]  (Domain purity — all 3 services)
```

### Phase 3: Architecture Fixes (Sequential after Phase 2)

Renames e reestruturação. Sequencial porque mudam paths que tarefas seguintes dependem.

```
T4-T8 → T9 → T10 → T11 → T12 → T13 → T14
```

### Phase 4: Config & Polish (Parallel after Phase 3)

```
     ┌→ T15 [P]  (Barrel elimination)
     │
T14 ─┼→ T16 [P]  (execute→handle)
     │
     ├→ T17 [P]  (tsconfig + eslint)
     │
     └→ T18 [P]  (public.ts business logic)
```

### Phase 5: Verification (Sequential)

```
T15-T18 → T19 → T20
```

---

## Task Breakdown

### T1: Criar estrutura base do monorepo suzano-fup-s4

**What**: Criar AGENTS.md raiz, package.json raiz, .nvmrc, .gitignore, renovate.json, docs/specs/project/ (já existe), e a estrutura de diretórios `packages/backend/`.
**Where**: Raiz do repo `/home/hiagoperoni/Projects/Suzano/S4/suzano-fup-s4/`
**Depends on**: None
**Reuses**: `suzano-cat2-frontend-s4/AGENTS.md` (template), scaffold prompt em `docs/obsidian/Obsidian/IAtizacao/Prompt - Scaffold Monorepo.md`
**Requirement**: FBR-01

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `AGENTS.md` raiz criado (≤150 linhas, segue template T-AGENTS-ROOT do scaffold prompt)
- [ ] `package.json` raiz criado com scripts `bd`, `build`, `deploy`, `undeploy`, `clean`
- [ ] `.nvmrc` com `22`
- [ ] `.gitignore` cobre `node_modules/`, `resources/`, `mta_archives/`, `dist/`, `gen/`, `@cds-models/`
- [ ] `renovate.json` com `baseBranches: [quality]`, patch-only
- [ ] `packages/backend/` diretório criado
- [ ] `packages/backend/AGENTS.md` criado (regras específicas do backend)

**Tests**: none
**Gate**: build

**Verify**:
```bash
ls -la AGENTS.md package.json .nvmrc .gitignore renovate.json packages/backend/AGENTS.md
wc -l AGENTS.md  # deve ser ≤150
```

**Commit**: `feat(scaffold): create monorepo base structure (AGENTS.md, package.json, configs)`

---

### T2: Migrar 3 serviços + db + app-routers para packages/backend/

**What**: Copiar `application-service/`, `processing-service/`, `public-service/`, `db/`, `app-router/` do repo original `suzano-fup-panel-backend-s4/` para `packages/backend/`. Copiar também o `app-router/` do `suzano-fup-public-frontend-s4/` como `public-app-router/`.
**Where**: `packages/backend/{application-service,processing-service,public-service,db,app-router,public-app-router}/`
**Depends on**: T1
**Reuses**: Código existente do `suzano-fup-panel-backend-s4/` e `suzano-fup-public-frontend-s4/app-router/`
**Requirement**: FBR-01

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `packages/backend/application-service/` contém todo o código do serviço original
- [ ] `packages/backend/processing-service/` contém todo o código do serviço original
- [ ] `packages/backend/public-service/` contém todo o código do serviço original
- [ ] `packages/backend/db/` contém models/, types/, views/, data/, src/ do original
- [ ] `packages/backend/app-router/` contém app-router interno (XSUAA)
- [ ] `packages/backend/public-app-router/` contém app-router público (sem auth)
- [ ] `yarn install` funciona em cada package copiado

**Tests**: none
**Gate**: build

**Verify**:
```bash
ls packages/backend/
# deve listar: application-service  app-router  db  processing-service  public-app-router  public-service
cd packages/backend/application-service && yarn install && yarn lint
```

**Commit**: `feat(migrate): move 3 services + db + 2 app-routers to packages/backend/`

---

### T3: Consolidar mta.yaml + xs-security.json + CI na raiz

**What**: Criar MTA unificado (`suzano-fup-s4` v2.0.0) na raiz com todos os módulos. Consolidar 2 xs-security.json (interno + público). Criar `.github/workflows/` com CI paths-filter + git-flow NUMENDS.
**Where**: Raiz do repo — `mta.yaml`, `xs-security.json`, `xs-security-public.json`, `.github/workflows/`
**Depends on**: T2
**Reuses**: `suzano-fup-panel-backend-s4/mta.yaml` (base), `suzano-fup-panel-backend-s4/xs-security.json`, `suzano-fup-public-frontend-s4/app-router/xs-app.json` (public sem auth)
**Requirement**: FBR-01

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `mta.yaml` na raiz com ID `suzano-fup-s4`, version `2.0.0`
- [ ] MTA contém 5 módulos nodejs (3 serviços) + hdb (db) + 2 app-routers
- [ ] MTA resources: HANA HDI, XSUAA (2), destination, connectivity, job-scheduler, 4 cross-service grants
- [ ] `xs-security.json` (interno) com scopes JOBSCHEDULER + uaa.user
- [ ] `xs-security-public.json` não existe (public sem auth — confirmar se precisa)
- [ ] `.github/workflows/fup-backend-ci.yml` com paths-filter (lint/typecheck por package)
- [ ] `.github/workflows/git-flow.yaml` com `NUMENDS/git-flow-action`
- [ ] `.github/pull_request_template.md` criado
- [ ] `yarn bd` (build + deploy) executa sem erro de sintaxe MTA

**Tests**: none
**Gate**: build

**Verify**:
```bash
# Validar MTA syntax
mbt build --mtar suzano-fup-s4 --dry-run 2>&1 | head -20
# Verificar módulos no MTA
grep "name:" mta.yaml | wc -l  # deve ter 7+ módulos
```

**Commit**: `feat(deploy): consolidate MTA (v2.0.0) + xs-security + CI with paths-filter`

---

### T4: Corrigir SQL injection — application-service [P]

**What**: Eliminar 3 sites de SQL injection no application-service usando `cds.ql` ou bound parameters.
**Where**:
- `packages/backend/application-service/src/infrastructure/persistence/purchase-order-item.ts:25-35,99-114`
- `packages/backend/application-service/src/infrastructure/persistence/fup-history.ts:23-25`
- `packages/backend/application-service/src/infrastructure/persistence/tax-invoice-item.ts:21-23`
**Depends on**: T3
**Reuses**: Padrão de `cds.ql` (SELECT.from().where()) do CAP
**Requirement**: FBR-02

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `purchase-order-item.ts`: zero `${keys.*}` em queries SQL — usar `cds.ql` ou `cds.run(sql, [params])`
- [ ] `fup-history.ts`: zero `${params.*}` em WHERE
- [ ] `tax-invoice-item.ts`: zero `${keys.*}` em WHERE
- [ ] `grep -rn '\${' --include="*.ts" packages/backend/application-service/src/infrastructure/ | grep -i "where\|in\|like"` retorna vazio
- [ ] `tsc --noEmit` passa em application-service
- [ ] Funcionalidade preservada (queries retornam mesmos resultados)

**Tests**: none (P1 não inclui testes unitários — D8)
**Gate**: build

**Verify**:
```bash
cd packages/backend/application-service
grep -rn '\${' src/infrastructure/ --include="*.ts" | grep -iE "where|in\(|like"
# deve retornar vazio
yarn ts-typecheck
```

**Commit**: `fix(app-service): eliminate SQL injection in 3 repository files (bound parameters)`

---

### T5: Corrigir SQL injection — processing-service [P]

**What**: Eliminar 3 sites de SQL injection no processing-service.
**Where**:
- `packages/backend/processing-service/src/infrastructure/persistence/purchase-order-item.ts:17,39,58,70`
- `packages/backend/processing-service/src/application/use-cases/actions/send-top-five-pending-fups-to-supplier.ts:40`
**Depends on**: T3
**Reuses**: Padrão de `cds.ql` do CAP
**Requirement**: FBR-02

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `purchase-order-item.ts`: zero `${suppliers}` ou `${taxNumbers}` em SQL
- [ ] `send-top-five-pending-fups-to-supplier.ts`: zero `goliveSuppliers.map(s => \`'${s.id}'\`)` — passar array para repository
- [ ] `grep -rn '\${' --include="*.ts" packages/backend/processing-service/src/ | grep -iE "where|in\(|like"` retorna vazio
- [ ] `tsc --noEmit` passa

**Tests**: none
**Gate**: build

**Verify**:
```bash
cd packages/backend/processing-service
grep -rn '\${' src/ --include="*.ts" | grep -iE "where|in\(|like"
# deve retornar vazio
yarn lint
```

**Commit**: `fix(proc-service): eliminate SQL injection in repository + use case (bound parameters)`

---

### T6: Corrigir SQL injection — public-service [P]

**What**: Eliminar 2 sites de SQL injection no public-service.
**Where**:
- `packages/backend/public-service/src/infrastructure/persistence/purchase-order-item.ts:46`
- `packages/backend/public-service/src/main/routes/public.ts:30`
**Depends on**: T3
**Reuses**: Padrão de `cds.ql` do CAP
**Requirement**: FBR-02

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `purchase-order-item.ts:46`: zero `LIKE '%${taxDocument}%'` — usar `cds.ql` com `.like()`
- [ ] `public.ts:30`: zero `= '${supplierTaxNumber}'` — mover SQL para repository (preparação para T18 que move business logic inteira)
- [ ] `grep -rn '\${' --include="*.ts" packages/backend/public-service/src/ | grep -iE "where|in\(|like|="` retorna vazio
- [ ] `tsc --noEmit` passa

**Tests**: none
**Gate**: build

**Verify**:
```bash
cd packages/backend/public-service
grep -rn '\${' src/ --include="*.ts" | grep -iE "where|in\(|like|="
# deve retornar vazio
yarn lint
```

**Commit**: `fix(pub-service): eliminate SQL injection in repository + route (bound parameters)`

---

### T7: Padronizar error taxonomy — todos os 3 serviços [P]

**What**: Fixar error classes em todos os 3 serviços: adicionar ConflictError(409), renomear Forbiden→Forbidden, fixar bug recursão infinita getter message(), remover dead code (BadGateway, UnprocessableEntity), mover SAPGatewayError/DellavolpeError para domain/adapters/external-api/.
**Where**:
- `packages/backend/{application-service,processing-service,public-service}/src/domain/errors/`
- `packages/backend/{application-service,processing-service,public-service}/src/domain/adapters/external-api/` (destino dos namespaces movidos)
**Depends on**: T3
**Reuses**: `docs/standards/cap-clean-architecture/domain-layer/errors/README.md`
**Requirement**: FBR-03

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `ConflictError` (409) criado em todos os 3 serviços
- [ ] `forbiden-request.ts` → `forbidden-request.ts` + `ForbidenRequestError` → `ForbiddenError` (9 files: 3 services × 3 files)
- [ ] Zero getters/setters `message()` customizados — usar apenas `super(message)` no construtor do `AbstractError`
- [ ] `BadGatewayError` (502) deletado de todos os serviços (0 instâncias — dead code)
- [ ] `UnprocessableEntity` (422) deletado do application-service (0 instâncias — dead code)
- [ ] `UnavailableServiceError` (503) mantido (4 instâncias ativas)
- [ ] `SAPGatewayError` namespace movido de `domain/errors/sap-gateway.ts` para `domain/adapters/external-api/sap-gateway/` em todos os serviços
- [ ] `DellavolpeError` namespace movido de `domain/errors/dellavolpe.ts` para `domain/adapters/external-api/dellavolpe/` no processing-service
- [ ] `ServerError` constructor padronizado (PROC tinha `constructor(stack: string)` inconsistente)
- [ ] `domain/errors/index.ts` barrel atualizado (remover BadGateway, UnprocessableEntity, SAPGateway, Dellavolpe; adicionar Conflict)
- [ ] Todos os imports atualizados nos use cases/controllers que referenciavam SAPGatewayError/DellavolpeError
- [ ] `tsc --noEmit` passa em todos os 3 serviços
- [ ] Teste manual: instanciar cada error class, acessar `.message`, verificar que não há stack overflow

**Tests**: none (D8 — testes são P2)
**Gate**: build

**Verify**:
```bash
# Verificar 7 classes
for svc in application-service processing-service public-service; do
  echo "=== $svc ==="
  ls packages/backend/$svc/src/domain/errors/*.ts | grep -v index | grep -v abstract
  # deve listar: bad-request, forbidden-request, not-found, conflict, server, unauthorized, unavailable-service
done

# Verificar zero recursão
grep -rn "get message\|set message" packages/backend/*/src/domain/errors/
# deve retornar vazio

# Verificar dead code removido
find packages/backend/ -name "bad-gateway*" -o -name "unprocessable-entity*"
# deve retornar vazio

# Verificar namespaces movidos
find packages/backend/ -path "*/external-api/sap-gateway*" -o -path "*/external-api/dellavolpe*"
# deve encontrar os arquivos movidos

# tsc
for svc in application-service processing-service public-service; do
  cd packages/backend/$svc && yarn ts-typecheck && cd -
done
```

**Commit**: `fix(errors): standardize error taxonomy (7 classes, fix recursion bug, remove dead code, move type namespaces)`

---

### T8: Eliminar framework do domain — todos os 3 serviços [P]

**What**: Remover 28 imports de `@models/db/models` de arquivos em `domain/`. Declarar tipos wrapper no namespace do próprio contrato. Mover `Params`/`Result` types de `domain/models/{actions,functions,entities}/` para `domain/use-cases/`.
**Where**:
- `packages/backend/application-service/src/domain/repositories/*.ts` (12 files)
- `packages/backend/application-service/src/domain/models/{actions,functions,entities}/*.ts` (15 files)
- `packages/backend/processing-service/src/domain/{repositories,models/actions}/*.ts` (7 files)
- `packages/backend/public-service/src/domain/{repositories,models/actions}/*.ts` (6 files)
**Depends on**: T3, T7 (T7 move SAPGatewayError/DellavolpeError que mudam imports em domain)
**Reuses**: `docs/standards/cap-clean-architecture/domain-layer/repositories/README.md` (namespace pattern)
**Requirement**: FBR-04

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `grep -rn "@models" packages/backend/*/src/domain/` retorna vazio
- [ ] `grep -rn "@sap/cds" packages/backend/*/src/domain/` retorna vazio
- [ ] Cada `domain/repositories/*.ts` declara `DbRow` type no próprio namespace (ex.: `export namespace PurchaseOrderItemRepository { type DbRow = { ... } }`)
- [ ] `Params`/`Result` types movidos de `domain/models/` para `domain/use-cases/<type>/<name>.ts`
- [ ] Todos os imports em `application/` e `infrastructure/` que referenciavam `@models` em domain atualizados
- [ ] `tsc --noEmit` passa em todos os 3 serviços

**Tests**: none (D8)
**Gate**: build

**Verify**:
```bash
grep -rn "@models" packages/backend/*/src/domain/ --include="*.ts"
# deve retornar vazio

grep -rn "@sap/cds" packages/backend/*/src/domain/ --include="*.ts"
# deve retornar vazio

for svc in application-service processing-service public-service; do
  cd packages/backend/$svc && yarn ts-typecheck && cd -
done
```

**Commit**: `refactor(domain): remove @models imports from domain (28 files, declare DbRow types in namespace)`

---

### T9: Rename persistence/ → repositories/ (todos os 3 serviços)

**What**: Renomear `infrastructure/persistence/` → `infrastructure/repositories/` em todos os 3 serviços. Atualizar todos os imports que referenciam `@/infrastructure/persistence/`.
**Where**: `packages/backend/{application-service,processing-service,public-service}/src/infrastructure/`
**Depends on**: T8 (T8 modifica arquivos em domain que podem importar de persistence)
**Reuses**: `docs/standards/cap-clean-architecture/infrastructure-layer/repositories/README.md`
**Requirement**: FBR-06

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `find packages/backend/ -type d -name "persistence"` retorna vazio
- [ ] `find packages/backend/ -type d -name "repositories"` encontra em todos os 3 serviços
- [ ] `grep -rn "persistence" packages/backend/*/src/ --include="*.ts"` retorna vazio
- [ ] `tsc --noEmit` passa em todos os 3 serviços

**Tests**: none
**Gate**: build

**Verify**:
```bash
find packages/backend/ -type d -name "persistence"
# deve retornar vazio

grep -rn "persistence" packages/backend/*/src/ --include="*.ts"
# deve retornar vazio

for svc in application-service processing-service public-service; do
  cd packages/backend/$svc && yarn ts-typecheck && cd -
done
```

**Commit**: `refactor(infra): rename persistence/ → repositories/ (3 services)`

---

### T10: Rename entities/ → entity-events/ (application-service)

**What**: Renomear `entities/` → `entity-events/` em 5 camadas do application-service (domain/use-cases, application/use-cases, presentation/controllers, main/factories). Também mover `domain/models/{actions,functions,entities}/` contratos para `domain/use-cases/` (iniciado em T8, finalizado aqui).
**Where**: `packages/backend/application-service/src/` — todas as camadas com `entities/`
**Depends on**: T9
**Reuses**: `docs/standards/cap-clean-architecture/` estrutura canônica
**Requirement**: FBR-06

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `find packages/backend/application-service/src/ -type d -name "entities"` retorna vazio
- [ ] `find packages/backend/application-service/src/ -type d -name "entity-events"` encontra em domain/use-cases, application/use-cases, presentation/controllers, main/factories
- [ ] `grep -rn "/entities/" packages/backend/application-service/src/ --include="*.ts"` retorna vazio
- [ ] `tsc --noEmit` passa

**Tests**: none
**Gate**: build

**Verify**:
```bash
find packages/backend/application-service/src/ -type d -name "entities"
# deve retornar vazio

find packages/backend/application-service/src/ -type d -name "entity-events"
# deve encontrar em 4+ camadas

cd packages/backend/application-service && yarn ts-typecheck
```

**Commit**: `refactor(app-service): rename entities/ → entity-events/ (5 layers)`

---

### T11: Eliminar pastas proibidas (extractors, formatters, hidrators, i18n, main/adapters)

**What**: Mover `extractors/` → `utils/get-environment.ts`, `formatters/` → `utils/` ou model method, `hidrators/` → `hydrators/` (fix typo), `i18n/` → `utils/translator/i18n/`, `main/adapters/` → `main/routes/` ou `main/config/`.
**Where**: Todos os 3 serviços em `domain/`, `infrastructure/`, `main/`
**Depends on**: T10
**Reuses**: `docs/standards/cap-clean-architecture/README.md` anti-padrão table
**Requirement**: FBR-07

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `find packages/backend/ -type d -name "extractors"` retorna vazio
- [ ] `find packages/backend/ -type d -name "formatters"` retorna vazio
- [ ] `find packages/backend/ -type d -name "hidrators"` retorna vazio (renomeado para `hydrators/` com 'y')
- [ ] `find packages/backend/ -path "*/infrastructure/i18n"` retorna vazio (movido para `utils/translator/i18n/`)
- [ ] `find packages/backend/ -path "*/main/adapters"` retorna vazio (movido para `main/routes/` ou `main/config/`)
- [ ] `grep -rn "hidrat" packages/backend/*/src/ --include="*.ts"` retorna vazio (method `hidrate()` → `hydrate()`)
- [ ] Todos os imports atualizados
- [ ] `tsc --noEmit` passa em todos os 3 serviços

**Tests**: none
**Gate**: build

**Verify**:
```bash
find packages/backend/ -type d \( -name "extractors" -o -name "formatters" -o -name "hidrators" \)
# deve retornar vazio

find packages/backend/ -path "*/infrastructure/i18n" -o -path "*/main/adapters"
# deve retornar vazio

grep -rn "hidrat" packages/backend/*/src/ --include="*.ts"
# deve retornar vazio

for svc in application-service processing-service public-service; do
  cd packages/backend/$svc && yarn ts-typecheck && cd -
done
```

**Commit**: `refactor(structure): eliminate prohibited folders (extractors, formatters, hidrators→hydrators, i18n, main/adapters)`

---

### T12: Eliminar barrel exports (53 index.ts)

**What**: Remover todos os `index.ts` com `export *` exceto `domain/errors/index.ts`. Atualizar todos os imports para apontar para arquivo específico.
**Where**: Todos os 3 serviços em todas as camadas
**Depends on**: T11 (T11 move pastas que mudam paths dos barrels)
**Reuses**: `docs/standards/cap-clean-architecture/README.md` Regra 9
**Requirement**: FBR-08

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `grep -rn "export \*" packages/backend/*/src/ --include="index.ts"` retorna apenas `domain/errors/index.ts`
- [ ] Todos os imports que usavam `from '@/domain/repositories'` mudaram para `from '@/domain/repositories/purchase-order-item.js'` (arquivo específico)
- [ ] `tsc --noEmit` passa em todos os 3 serviços

**Tests**: none
**Gate**: build

**Verify**:
```bash
grep -rn "export \*" packages/backend/*/src/ --include="index.ts" | grep -v "domain/errors/index"
# deve retornar vazio

for svc in application-service processing-service public-service; do
  cd packages/backend/$svc && yarn ts-typecheck && cd -
done
```

**Commit**: `refactor(structure): eliminate barrel exports (53 index.ts files, keep only domain/errors/index.ts)`

---

### T13: Controllers execute() → handle() (34 arquivos)

**What**: Renomear método público `execute` para `handle` em todos os controllers + base classes. Atualizar todos os callers (factories, routes).
**Where**: `packages/backend/*/src/presentation/controllers/` + `packages/backend/*/src/main/factories/`
**Depends on**: T12
**Reuses**: `docs/standards/cap-clean-architecture/presentation-layer/controllers/base.md`
**Requirement**: FBR-09

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `grep -rn "execute(" packages/backend/*/src/presentation/controllers/ --include="*.ts"` retorna vazio
- [ ] `grep -rn "\.execute(" packages/backend/*/src/main/ --include="*.ts"` retorna vazio (callers atualizados para `.handle()`)
- [ ] `tsc --noEmit` passa em todos os 3 serviços

**Tests**: none
**Gate**: build

**Verify**:
```bash
grep -rn "execute" packages/backend/*/src/presentation/controllers/ --include="*.ts"
# deve retornar vazio

grep -rn "\.execute(" packages/backend/*/src/main/ --include="*.ts"
# deve retornar vazio

for svc in application-service processing-service public-service; do
  cd packages/backend/$svc && yarn ts-typecheck && cd -
done
```

**Commit**: `refactor(controllers): rename execute() → handle() (34 files, 3 services)`

---

### T14: Padronizar tsconfig + eslint (3 serviços)

**What**: Alinhar tsconfig (strict, strictNullChecks, noImplicitAny) e eslint (printWidth 180, no-console allow error, no-explicit-any error) ao standard. Criar .nvmrc + renovate.json na raiz (se não criado em T1).
**Where**: `packages/backend/{application-service,processing-service,public-service}/tsconfig.json` + `eslint.config.mjs`
**Depends on**: T13 (T13 é o último rename antes de config changes que geram diff massivo)
**Reuses**: `docs/standards/code-style/typescript.md`
**Requirement**: FBR-10

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `strictNullChecks: true` em todos os 3 tsconfig
- [ ] `noImplicitAny: true` em todos os 3 tsconfig
- [ ] `strictPropertyInitialization: true` em todos os 3 tsconfig
- [ ] `printWidth: 180` em todos os 3 eslint.config.mjs
- [ ] `no-console: ['error', { allow: ['error'] }]` em todos
- [ ] `no-explicit-any: 'error'` em todos
- [ ] `eslint --fix` executado para auto-formatar
- [ ] `tsc --noEmit` passa (pode exigir fixes de null/any — corrigir conforme surgirem)
- [ ] `eslint` passa em todos os 3 serviços

**Tests**: none
**Gate**: build

**Verify**:
```bash
for svc in application-service processing-service public-service; do
  echo "=== $svc ==="
  grep "strictNullChecks\|noImplicitAny" packages/backend/$svc/tsconfig.json
  grep "printWidth\|no-console\|no-explicit-any" packages/backend/$svc/eslint.config.mjs
done

for svc in application-service processing-service public-service; do
  cd packages/backend/$svc && yarn ts-typecheck && yarn lint && cd -
done
```

**Commit**: `refactor(config): standardize tsconfig (strict) + eslint (printWidth 180, no-console allow error)`

---

### T15: Mover business logic de main/routes/public.ts [P]

**What**: Mover SQL inline + business logic de `public-service/main/routes/public.ts:15-44` para repository + controller + use case.
**Where**:
- `packages/backend/public-service/src/domain/repositories/notification-log.ts` (criar interface)
- `packages/backend/public-service/src/infrastructure/repositories/notification-log.ts` (criar impl com bound params)
- `packages/backend/public-service/src/application/use-cases/actions/create-notification-log.ts` (criar use case)
- `packages/backend/public-service/src/presentation/controllers/actions/create-notification-log.ts` (criar controller)
- `packages/backend/public-service/src/main/factories/controllers/actions/create-notification-log.ts` (criar factory)
- `packages/backend/public-service/src/main/routes/public.ts` (modificar — delegar para controller)
**Depends on**: T14
**Reuses**: Padrão de controllers/factories existentes nos outros serviços
**Requirement**: FBR-05

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `main/routes/public.ts` não tem SQL inline nem business logic (apenas `service.on('CREATE', 'NotificationLogs', adaptRoute(makeCreateNotificationLogController()))`)
- [ ] `NotificationLogRepository` interface criada em `domain/repositories/`
- [ ] Repository impl criada em `infrastructure/repositories/` com bound parameters
- [ ] Use case criado retornando `Either<AbstractError, T>`
- [ ] Controller criado com método `handle()`
- [ ] Factory criada em `main/factories/`
- [ ] `grep -rn "cds.run\|cds.create\|SELECT.*FROM" packages/backend/public-service/src/main/routes/` retorna vazio
- [ ] `tsc --noEmit` passa

**Tests**: none
**Gate**: build

**Verify**:
```bash
grep -rn "cds.run\|cds.create\|SELECT.*FROM" packages/backend/public-service/src/main/routes/
# deve retornar vazio

cd packages/backend/public-service && yarn ts-typecheck
```

**Commit**: `refactor(pub-service): move business logic from public.ts route to controller/use case/repository`

---

### T16: Smoke test de build completo [P]

**What**: Validar que `yarn bd` (build + deploy MTA) funciona end-to-end. Rodar `cds build` + `mbt build` + verificar que não há erros de compilação.
**Where**: Raiz do repo
**Depends on**: T15
**Reuses**: NONE
**Requirement**: FBR-01 (verification)

**Tools**:
- MCP: NONE
- Skill: NONE

**Done when**:
- [ ] `yarn install` funciona na raiz e em cada package
- [ ] `cds build --production` passa para todos os 3 serviços
- [ ] `mbt build --mtar suzano-fup-s4` gera o MTAR sem erro
- [ ] `cf deploy mta_archives/suzano-fup-s4.mtar --retries 0 -f` não é executado (apenas validar build, não deployar em PRD)

**Tests**: none
**Gate**: build

**Verify**:
```bash
yarn install
yarn build  # mbt build
ls -la mta_archives/suzano-fup-s4.mtar
# arquivo deve existir
```

**Commit**: `test(build): smoke test — MTA build passes end-to-end`

---

### T17: Auditoria final de conformidade

**What**: Re-rodar a auditoria do research para verificar que zero violations critical/high permanecem. Atualizar `docs/specs/project/STATE.md` com resultado.
**Where**: `docs/researches/fup-backend-refactor/research.md` (re-verificar), `docs/specs/project/STATE.md` (atualizar)
**Depends on**: T16
**Reuses**: Checklist de auditoria do research
**Requirement**: FBR-01 a FBR-10 (verification)

**Tools**:
- MCP: NONE
- Skill: `review-implementation`

**Done when**:
- [ ] `grep -rn "@models" packages/backend/*/src/domain/ --include="*.ts"` retorna vazio
- [ ] `grep -rn '\${' packages/backend/*/src/ --include="*.ts" | grep -iE "where|in\(|like|="` retorna vazio
- [ ] `grep -rn "get message\|set message" packages/backend/*/src/domain/errors/` retorna vazio
- [ ] `find packages/backend/ -type d \( -name "extractors" -o -name "formatters" -o -name "hidrators" -o -name "persistence" \)` retorna vazio
- [ ] `grep -rn "export \*" packages/backend/*/src/ --include="index.ts" | grep -v "domain/errors/index"` retorna vazio
- [ ] `grep -rn "execute" packages/backend/*/src/presentation/controllers/ --include="*.ts"` retorna vazio
- [ ] `grep -rn "Forbiden" packages/backend/` retorna vazio
- [ ] `find packages/backend/ -name "bad-gateway*" -o -name "unprocessable-entity*"` retorna vazio
- [ ] STATE.md atualizado com resultado da auditoria

**Tests**: none
**Gate**: build

**Verify**:
```bash
# Rodar todos os greps acima — todos devem retornar vazio
echo "Conformidade: PASS"
```

**Commit**: `docs(state): update STATE.md with final audit results (zero critical/high violations)`

---

## Parallel Execution Map

```
Phase 1 (Sequential):
  T1 ──→ T2 ──→ T3

Phase 2 (Parallel after T3):
  T3 complete, then:
    ├── T4 [P]  (APP SQL injection)
    ├── T5 [P]  (PROC SQL injection)
    ├── T6 [P]  (PUB SQL injection)
    ├── T7 [P]  (Error taxonomy — all 3 services)
    └── T8 [P]  (Domain purity — all 3 services, depends on T7)

Phase 3 (Sequential after Phase 2):
  T4-T8 complete, then:
    T8 ──→ T9 ──→ T10 ──→ T11 ──→ T12 ──→ T13 ──→ T14

Phase 4 (Parallel after T14):
  T14 complete, then:
    ├── T15 [P]  (public.ts business logic)
    └── T16 [P]  (smoke test — can start after T15 actually)

Phase 5 (Sequential):
  T15-T16 ──→ T17
```

**Parallelism constraint:** T8 depends on T7 (error taxonomy moves namespaces that T8 needs to reference). All Phase 2 tasks depend only on T3 and don't share mutable state. Phase 4 tasks are independent.

---

## Task Granularity Check

| Task | Scope | Status |
|------|-------|--------|
| T1: Create monorepo base | 6 config files | ✅ Granular |
| T2: Migrate 3 services + db + routers | Copy operation (6 dirs) | ✅ Granular |
| T3: Consolidate MTA + xs-security + CI | 4 config files | ✅ Granular |
| T4: Fix SQL injection APP | 3 files | ✅ Granular |
| T5: Fix SQL injection PROC | 2 files | ✅ Granular |
| T6: Fix SQL injection PUB | 2 files | ✅ Granular |
| T7: Error taxonomy all services | ~30 files (delete/create/rename/move) | ⚠️ Large but cohesive (1 concept across services) |
| T8: Domain purity all services | 28 files (remove @models, declare DbRow) | ⚠️ Large but cohesive (1 concept across services) |
| T9: Rename persistence→repositories | 3 dirs + imports | ✅ Granular |
| T10: Rename entities→entity-events | 1 service, 5 layers | ✅ Granular |
| T11: Eliminate prohibited folders | ~15 dirs across services | ⚠️ Large but cohesive (all are "move folder" operations) |
| T12: Eliminate barrel exports | 53 files | ⚠️ Large but mechanical (find-replace) |
| T13: execute→handle | 34 files | ✅ Granular (mechanical find-replace) |
| T14: tsconfig + eslint | 6 config files + fixes | ✅ Granular |
| T15: Move business logic public.ts | 5 new files + 1 modify | ✅ Granular |
| T16: Smoke test build | 1 command | ✅ Granular |
| T17: Final audit | Verification only | ✅ Granular |

**Verdict:** T7, T8, T11, T12 are large but cohesive — each implements ONE concept across all services. Splitting by service would create 3× the tasks with same total work but more overhead. Acceptable.

---

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
|------|------------------------|---------------|--------|
| T1 | None | Start node | ✅ Match |
| T2 | T1 | T1→T2 | ✅ Match |
| T3 | T2 | T2→T3 | ✅ Match |
| T4 | T3 | T3→T4 [P] | ✅ Match |
| T5 | T3 | T3→T5 [P] | ✅ Match |
| T6 | T3 | T3→T6 [P] | ✅ Match |
| T7 | T3 | T3→T7 [P] | ✅ Match |
| T8 | T3, T7 | T3→T8 [P] + T7→T8 | ✅ Match |
| T9 | T8 | T8→T9 | ✅ Match |
| T10 | T9 | T9→T10 | ✅ Match |
| T11 | T10 | T10→T11 | ✅ Match |
| T12 | T11 | T11→T12 | ✅ Match |
| T13 | T12 | T12→T13 | ✅ Match |
| T14 | T13 | T13→T14 | ✅ Match |
| T15 | T14 | T14→T15 [P] | ✅ Match |
| T16 | T15 | T15→T16 [P] | ✅ Match |
| T17 | T16 | T16→T17 | ✅ Match |

---

## Test Co-location Validation

> TESTING.md não existe neste projeto (greenfield specs). Testes são P2 por decisão D8. P1 inclui apenas smoke test de build (T16).

| Task | Code Layer Created/Modified | Matrix Requires | Task Says | Status |
|------|---------------------------|-----------------|-----------|--------|
| T1-T3 | Config files | none | none | ✅ OK (config only) |
| T4-T6 | infrastructure/repositories | none (P1) | none | ✅ OK (D8 — testes P2) |
| T7 | domain/errors | none (P1) | none | ✅ OK (D8) |
| T8 | domain/repositories, domain/use-cases | none (P1) | none | ✅ OK (D8) |
| T9-T13 | Infrastructure restructure | none (P1) | none | ✅ OK (D8) |
| T14 | tsconfig + eslint | none | none | ✅ OK (config only) |
| T15 | domain/infra/application/presentation/main | none (P1) | none | ✅ OK (D8) |
| T16 | Build verification | build | build | ✅ OK |
| T17 | Audit | none | none | ✅ OK |

**Note:** Testes unitários (vitest) para os use cases e repositories serão criados no M2 (EP-22 do research). A decisão D8 (testes como P2) justifica `Tests: none` em todas as tarefas de P1.

---

## Requirement Traceability

| Requirement ID | Tasks | Status |
|----------------|-------|--------|
| FBR-01 (Scaffold) | T1, T2, T3, T16 | Pending |
| FBR-02 (SQL injection) | T4, T5, T6 | Pending |
| FBR-03 (Error taxonomy) | T7 | Pending |
| FBR-04 (Domain purity) | T8 | Pending |
| FBR-05 (Business logic public.ts) | T15 | Pending |
| FBR-06 (Rename pastas) | T9, T10 | Pending |
| FBR-07 (Pastas proibidas) | T11 | Pending |
| FBR-08 (Barrel exports) | T12 | Pending |
| FBR-09 (execute→handle) | T13 | Pending |
| FBR-10 (tsconfig+eslint) | T14 | Pending |
| (Verification) | T17 | Pending |

**Coverage:** 10 requirements, 17 tasks, 10/10 mapped ✅
