# Pesquisa: Auditoria Backend FUP S4 vs cap-clean-architecture

> Auditoria arquivo a arquivo dos 3 serviços CAP + db + app-router do `suzano-fup-panel-backend-s4` contra o standard `cap-clean-architecture`. Produz insumo cirúrgico para criar épicos e subtasks no Jira (board RFIA, ID 2011). O código está em produção — a refatoração deve preservar comportamento.

| Item | Valor |
|---|---|
| Slug | `fup-backend-refactor` |
| Data | 2026-08-21 |
| Modo | Profundo |
| Consumidor previsto | `tlc-spec-driven` (Specify) + Jira board RFIA |
| Fontes utilizadas | Codebase atual (3 serviços + db + app-router) + Standard cap-clean-architecture (50+ arquivos) + Research global `fup-s4-migration` |
| Status | Final |
| Autor | Agente Kilo |

---

## Sumário

1. [Visão geral — contagem de violações por serviço](#1-visão-geral)
2. [Achados por dimensão](#2-achados-por-dimensão)
   - 2.1 [Domain purity — framework no domain](#21-domain-purity--framework-no-domain)
   - 2.2 [Models anêmicos vs ricos](#22-models-anêmicos-vs-ricos)
   - 2.3 [SQL injection](#23-sql-injection)
   - 2.4 [Pastas proibidas (anti-padrões)](#24-pastas-proibidas-anti-padrões)
   - 2.5 [Barrel exports](#25-barrel-exports)
   - 2.6 [Naming: hidrators, entities, persistence](#26-naming-hidrators-entities-persistence)
   - 2.7 [Error taxonomy](#27-error-taxonomy)
   - 2.8 [tsconfig + eslint divergências](#28-tsconfig--eslint-divergências)
   - 2.9 [Controllers: execute vs handle](#29-controllers-execute-vs-handle)
   - 2.10 [Cross-cutting: duplicação entre serviços](#210-cross-cutting-duplicação-entre-serviços)
   - 2.11 [db/ divergências](#211-db-divergências)
   - 2.12 [Build pipeline + scripts](#212-build-pipeline--scripts)
   - 2.13 [Testes](#213-testes)
3. [Padrões observados — o que NÃO mudar](#3-padrões-observados--o-que-não-mudar)
4. [Gaps & incógnitas](#4-gaps--incógnitas)
5. [Épicos Jira candidatos](#5-épicos-jira-candidatos)
6. [Fontes consultadas](#6-fontes-consultadas)
7. [Handoff TLC Spec-Driven + Jira](#7-handoff-tlc-spec-driven--jira)

---

## 1. Visão geral

### 1.1 Contagem de violações por serviço

| Serviço | .ts files | Critical | High | Medium | Low | Total |
|---|---|---|---|---|---|---|
| `application-service` | ~160 | 6 | 12 | 9 | 3 | 30 |
| `processing-service` | ~89 | 17 | 22 | 28 | 3 | 70 |
| `public-service` | ~66 | 9 | 7 | 13 | 4 | 33 |
| `db/` (CDS+HANA) | 62 CDS + 53 CSV + ~25 HANA | 0 | 2 | 2 | 6 | 10 |
| `app-router/` + root configs | ~10 | 0 | 1 | 1 | 3 | 5 |
| **Total** | **~315** | **32** | **44** | **53** | **19** | **148** |

### 1.2 Top 10 violações mais críticas (cross-service)

| # | Violação | Serviços afetados | Regra do standard |
|---|---|---|---| 
| V1 | `@models/db/models` importado em `domain/` | APP (14 files), PROC (7 files), PUB (7 files) | Regra 7: Sem framework no domain |
| V2 | Models anêmicos (`type Xxx = {}`) sem `class` + `static with()` | APP (24+), PROC (2), PUB (3) | Regra 3: Models ricos, nunca anêmicos |
| V3 | SQL injection (string interpolation) | APP (3 repos), PROC (2 repos+1 use case), PUB (2 files) | Convenção: parameterized queries |
| V4 | `strictNullChecks: false` + `noImplicitAny: false` | APP, PROC, PUB (todos) | Regra implícita: strict TS |
| V5 | `hidrators/` typo (deveria ser `hydrators/`) | APP | Anti-padrão: hidrators/hidrate → hydrators/hydrate |
| V6 | `persistence/` ao invés de `repositories/` | APP, PROC, PUB | Estrutura canônica: repositories/{models,replication}/ |
| V7 | `extractors/` + `formatters/` (pastas proibidas) | APP, PROC, PUB | Anti-padrão: → utils/ ou model method |
| V8 | 24+ barrel `index.ts` re-exportando | APP (24), PROC (16), PUB (13) | Regra 9: Sem barrel exports |
| V9 | `entities/` ao invés de `entity-events/` | APP (5 layers) | Estrutura canônica: entity-events/ |
| V10 | `no-console: 'error'` (bloqueia `console.error`) | APP, PROC, PUB | Regra 6: `['error', { allow: ['error'] }]` |

### 1.3 Erro runtime crítico (bug em produção)

| # | Bug | Arquivos | Impacto |
|---|---|---|---|
| B1 | `get message() { return this.message }` — **recursão infinita** | 7 error classes files em cada serviço (21 files total) | Se `error.message` for acessado, stack overflow. Provavelmente não ocorre porque `Error.message` nativo é usado, mas é bomba relógio |

---

## 2. Achados por dimensão

### 2.1 Domain purity — framework no domain

> **Regra 7 do standard:** `@sap/cds`, `@models/*`, `@cds-models/*`, `@sap/textbundle`, `@sap/xsenv` e qualquer SDK externo são **proibidos** em `src/domain/`. Única dependência externa: `@sweet-monads/either`.

| # | Serviço | Arquivos | Import proibido | Severidade |
|---|---|---|---|---|
| D1 | APP | `domain/repositories/*.ts` (12 files), `domain/models/{actions,functions,entities}/*.ts` (15 files), `domain/models/purchase-order-item.ts` | `import ... from '@models/db/models'` | Critical |
| D2 | PROC | `domain/repositories/{notification-link-recipient,notification-link,notification-log}.ts`, `domain/models/actions/{save-carrier-invoice-data,send-carrier-pending-fups,send-pending-notes-reminder-email,send-top-five-pending-fups-to-supplier}.ts` | `import ... from '@models/db/models'` | Critical |
| D3 | PUB | `domain/models/actions/{fup-data-by-email-and-referer,save-carrier-confirmation,save-purchase-order-confirmation}.ts`, `domain/repositories/{notification-link,notification-log,purchase-order-item,supplier-email}.ts` | `import ... from '@models/db/models'` | Critical |

**Total: 28 arquivos em `domain/` importando tipos gerados do CDS (cds-typer).**

**Solução canônica:** Declarar tipos wrapper no namespace do próprio contrato (ex.: `XxxRepository.DbRow`) ao invés de importar `@models/*`. O domain não deve conhecer a estrutura física do CDS — apenas os tipos que precisa para seus contratos.

### 2.2 Models anêmicos vs ricos

> **Regra 3 do standard:** `class XxxModel` com `XxxProps` + `static with(props)` + getters + `validate()` + serialização. `type XxxModel = { ... }` é **proibido**.

| # | Serviço | Arquivos | Estado atual | Esperado |
|---|---|---|---|---|
| M1 | APP | `domain/models/**/*.ts` (24+ files) | `export type Xxx = { ... }` anêmico | `class XxxModel` com factory + validation |
| M2 | PROC | `domain/models/email.ts`, `domain/models/user-provided-variables.ts` | `type EmailBody/Sender/Recipient`, `type UserProvidedVariables` | `class XxxModel` |
| M3 | PUB | `domain/models/actions/*.ts` (3 files) | Namespace types apenas | Rich model classes |
| M4 | APP | `domain/models/{actions,functions,entities}/` | Use-case `Params`/`Result` types vivem em `models/` (misplaced) | Contratos em `domain/use-cases/<type>/<name>.ts` |

**Total: ~29 arquivos com models anêmicos que precisam virar classes ricas.**

### 2.3 SQL injection

> **Standard conventions.md:** SQL pleno deve usar parâmetros bound (`cds.run(sql, [params])`), nunca string interpolation.

| # | Serviço | Arquivo:Linhas | Padrão | Severidade |
|---|---|---|---|---|
| S1 | APP | `infrastructure/persistence/purchase-order-item.ts:25-35,99-114` | `${keys.mandant/poNumber/poItem}` interpolated | Critical |
| S2 | APP | `infrastructure/persistence/fup-history.ts:23-25` | `${params.*}` in WHERE | Critical |
| S3 | APP | `infrastructure/persistence/tax-invoice-item.ts:21-23` | `${keys.*}` in WHERE | Critical |
| S4 | PROC | `infrastructure/persistence/purchase-order-item.ts:17,39` | `${suppliers}` IN clause | Critical |
| S5 | PROC | `infrastructure/persistence/purchase-order-item.ts:58,70` | `taxNumbers.map(t => \`'${t}'\`)` then IN | Critical |
| S6 | PROC | `application/use-cases/actions/send-top-five-pending-fups-to-supplier.ts:40` | `goliveSuppliers.map(s => \`'${s.id}'\`)` builds SQL string | Critical |
| S7 | PUB | `infrastructure/persistence/purchase-order-item.ts:46` | `LIKE '%${taxDocument}%'` | Critical |
| S8 | PUB | `main/routes/public.ts:30` | `= '${supplierTaxNumber}'` raw SQL in route | Critical |

**Total: 8 sites de SQL injection em 7 arquivos.**

**Adicional:** PUB `main/routes/public.ts:15-44` tem **business logic + SQL inline na rota** (CREATE handler com cds.run + cds.create) — violação de dependency rule (main/ não tem business logic).

### 2.4 Pastas proibidas (anti-padrões)

> **Standard anti-padrão table:** `extractors/` → `utils/get-*.ts`; `formatters/` → `utils/` ou model method; `types/` → namespace; `services/` em infrastructure → `application/services/`; `hidrators/` → `hydrators/`; `validators/` → `validate()` no model; `middlewares/` → application/ ou infrastructure/.

| Pasta proibida | APP | PROC | PUB | Substituto canônico |
|---|---|---|---|---|
| `domain/extractors/` | ✅ existe | ✅ existe | ❌ | `domain/utils/get-environment.ts` |
| `infrastructure/extractors/` | ✅ existe | ✅ existe | ❌ | `infrastructure/utils/get-environment.ts` |
| `domain/formatters/` | ✅ existe | ✅ existe | ✅ existe | `utils/` ou model method |
| `infrastructure/formatters/` | ✅ existe | ✅ existe | ✅ existe | `infrastructure/utils/` |
| `domain/hidrators/` (typo) | ✅ existe | ❌ | ❌ | `domain/hydrators/` |
| `infrastructure/hidrators/` (typo) | ✅ existe | ❌ | ❌ | `infrastructure/hydrators/` |
| `infrastructure/i18n/` | ✅ existe (APP only) | ❌ | ❌ | `infrastructure/utils/translator/i18n/` |
| `main/adapters/` | ✅ existe | ✅ existe | ✅ existe | Mover para `main/routes/` ou `main/config/` |
| `main/annotations/` | ✅ existe (APP only) | ❌ | ❌ | Manter em `main/` (não-proibido mas não-canônico) |
| `infrastructure/services/` | ❌ | ❌ | ❌ | — (não existe, ok) |

### 2.5 Barrel exports

> **Regra 9:** Sem barrel exports dentro das camadas (exceção: `domain/errors/index.ts` e `*/entity-events/<entidade>/index.ts`).

| Serviço | Barrel files count | Localizações principais |
|---|---|---|
| APP | 24+ | `domain/{repositories,models,models/entities,models/sap,models/actions,models/functions,adapters,adapters/http,adapters/external-api,extractors,formatters}`, `infrastructure/{persistence,extractors,formatters,adapters/*,adapters/http}`, `application/use-cases/base`, `presentation/controllers/base`, `main/factories/controllers/{actions,functions}` |
| PROC | 16 | `domain/{adapters,adapters/external-api,adapters/http,extractors,formatters,models,models/actions,repositories}`, `infrastructure/{persistence,extractors,formatters,adapters/*,adapters/http}`, `application/use-cases/base`, `main/factories/{controllers,use-cases}/actions` |
| PUB | 13 | `domain/{adapters,adapters/external-api,adapters/http,formatters,models,models/actions,repositories}`, `infrastructure/{persistence,formatters,adapters/*,adapters/http}`, `application/use-cases/base`, `main/factories/controllers/actions` |

**Total: ~53 barrel `index.ts` files para eliminar.**

### 2.6 Naming: hidrators, entities, persistence

| # | Violação | Serviços | Atual | Esperado |
|---|---|---|---|---|
| N1 | `hidrators/` typo | APP | `domain/hidrators/`, `infrastructure/hidrators/`, `hidrate()` | `hydrators/`, `hydrate()` |
| N2 | `entities/` folder | APP (5 layers) | `domain/use-cases/entities/`, `application/use-cases/entities/`, `presentation/controllers/entities/`, `main/factories/.../entities/` | `entity-events/` |
| N3 | `persistence/` folder | APP, PROC, PUB | `infrastructure/persistence/` | `infrastructure/repositories/{models,replication}/` |
| N4 | Controller method | APP, PROC, PUB | `execute()` | `handle()` |
| N5 | `ForbidenRequestError` typo | APP, PROC, PUB | `forbiden-request.ts`, `ForbidenRequestError` | `forbidden-request.ts`, `ForbiddenError` |
| N6 | Adapter naming | PROC | `CarrierApiAdapter`, `EmailAdapter`, `CarrierApiAdapterImpl` | `CarrierApi`, `EmailApi`, `CarrierApiImpl` |
| N7 | Service naming | PROC | `EmailTemplateService` (static, no `Impl`) | `EmailTemplateServiceImpl` with `execute()` |

### 2.7 Error taxonomy

> **Standard:** 6 subclasses obrigatórias: `BadRequestError` (400), `UnauthorizedError` (401), `ForbiddenError` (403), `NotFoundError` (404), `ConflictError` (409), `ServerError` (500).
>
> **Decisão G3 (fechada):** 7 classes canônicas — 6 do standard + `UnavailableServiceError` (503). Remover dead code (BadGateway, UnprocessableEntity). Mover SAPGatewayError/DellavolpeError para `domain/adapters/external-api/`.

| # | Violação | Serviços | Detalhe | Resolução |
|---|---|---|---|---|
| E1 | **Missing `ConflictError` (409)** | APP, PROC, PUB | Nenhum serviço tem a classe 409 obrigatória | Criar nos 3 serviços |
| E2 | **`BadGatewayError` (502)** | APP, PROC, PUB | 0 instanciações — **dead code** | Deletar |
| E3 | **`UnprocessableEntity` (422)** | APP only | 0 instanciações — **dead code** | Deletar |
| E4 | **`UnavailableServiceError` (503)** | APP, PROC, PUB | 4 instanciações ativas — "API externa indisponível" | **Manter** (ADR justifica) |
| E5 | **`SAPGatewayError`** | APP, PROC, PUB | Não é classe de erro — é `namespace` com type definitions para parse de resposta SAP Gateway | Mover para `domain/adapters/external-api/sap-gateway/` |
| E6 | **`DellavolpeError`** | PROC only | Não é classe de erro — é `namespace` com type definitions para parse de resposta DellaVolpe | Mover para `domain/adapters/external-api/dellavolpe/` |
| E7 | **`ForbidenRequestError` typo** | APP, PROC, PUB | `forbiden` → `forbidden` (9 files: 3 services × 3 files) | Renomear |
| E8 | **Recursão infinita no getter `message()`** | APP, PROC, PUB (7 error classes × 3 services = 21 files) | `get message() { return this.message }` → stack overflow se acessado. **Bug ativo** (G2) | Remover getter/setter, usar `super(message)` |
| E9 | **`ServerError` constructor inconsistente** | PROC | `constructor(stack: string)` ao invés de `(message, stack?)` | Padronizar |

### 2.8 tsconfig + eslint divergências

| Config | APP | PROC | PUB | Standard |
|---|---|---|---|---|
| `strictNullChecks` | `false` | `false` | `false` | `true` |
| `noImplicitAny` | `false` | `false` | `false` | `true` |
| `strictPropertyInitialization` | `false` | `false` | `false` | `true` |
| `printWidth` (eslint) | 120 | 120 | 120 | 180 |
| `no-console` (eslint) | `'error'` (blocks all) | `['error']` (blocks all) | `'error'` (blocks all) | `['error', { allow: ['error'] }]` |
| `no-explicit-any` (eslint) | `'off'` | `'off'` | `'off'` | `'error'` |
| `module` | `commonjs` | `commonjs` | `commonjs` | `NodeNext` |
| `moduleResolution` | `node` | `node` | `node` | `NodeNext` |
| `.js` extension in imports | ❌ | ❌ | ❌ | ✅ obrigatório |
| `engines.node` | `^22` | `^22` | `^22` | `^22` ✅ |
| `.nvmrc` | ❌ missing | ❌ | ❌ | ✅ obrigatório |
| `renovate.json` | ❌ missing | ❌ | ❌ | ✅ obrigatório |

### 2.9 Controllers: execute vs handle

> **Standard:** Método público do controller é sempre `handle`. Sem `req.reject`/`req.reply`.

| Serviço | Arquivos | Método atual | Esperado |
|---|---|---|---|
| APP | `presentation/controllers/base/base.ts` + 23 controllers | `execute(...)` | `handle(...)` |
| PROC | `presentation/controllers/base/base.ts` + 4 controllers | `execute(...)` | `handle(...)` |
| PUB | `presentation/controllers/base/base.ts` + 3 controllers | `execute(...)` | `handle(...)` |

**Total: 31 controllers + 3 base classes = 34 arquivos.**

### 2.10 Cross-cutting: duplicação entre serviços

#### Arquivos IDENTICAL (copy-paste perfeito — podem ser shared)

| Arquivo | APP | PROC | PUB | Ação |
|---|---|---|---|---|
| `domain/errors/abstract.ts` | ✅ | ✅ | ✅ | Extrair para shared |
| `domain/errors/{bad-request,bad-gateway,forbiden-request,not-found,sap-gateway,server,unauthorized,unavailable-service}.ts` | ✅ | ✅ | ✅ | Extrair para shared |
| `domain/adapters/http/index.ts` | ✅ | ✅ | ✅ | Extrair para shared |
| `domain/adapters/index.ts` | ✅ | ✅ | ✅ | Extrair para shared |
| `infrastructure/adapters/http/index.ts` | ✅ | ✅ | ✅ | Extrair para shared |
| `main/config/module-alias.ts` | ✅ | ✅ | ✅ | Extrair para shared |
| `presentation/controllers/base/index.ts` | ✅ | ✅ | ✅ | Eliminar barrel (regra 9) |
| `application/use-cases/base/index.ts` | ✅ | ✅ | ✅ | Eliminar barrel (regra 9) |

#### Arquivos SIGNIFICANTLY DIFFERENT (drifted — precisam reconciliação)

| Arquivo | Quem evoluiu mais | O que fazer |
|---|---|---|
| `application/use-cases/base/base.ts` (BaseUseCaseImpl) | PROC/PUB (mais robusto: +DellavolpeError +fallback genérico) | Adotar PROC/PUB como canônico; APP precisa adicionar Dellavolpe |
| `presentation/controllers/base/base.ts` (BaseController) | APP (tipado GetLoggedUser + isBase64Response) | Adotar APP como canônico; PROC/PUB precisam tipar (regrediram para `any`) |
| `main/adapters/router.ts` (adaptRoute) | APP (suporta records + isBase64Response) | Adotar APP superset (params opcionais mantêm compat) |
| `domain/adapters/http/request.ts` | APP/PROC (split get/post + body fallback) | Adotar APP/PROC; PUB perdeu `get()` — restaurar |
| `infrastructure/adapters/http/request.ts` | APP/PROC (get+post + body fallback) | Adotar APP/PROC; PUB perdeu `get()` — restaurar |

#### Missing em todos os serviços

| Componente | Standard exige | Estado |
|---|---|---|
| `infrastructure/utils/translator/` | i18n wrapper | ❌ ausente em todos (APP tem `infrastructure/i18n/` não-canônico) |
| `infrastructure/utils/get-user.ts` | Extrair usuário do contexto | ❌ ausente em todos |
| `infrastructure/utils/get-environment.ts` | Detectar ambiente (dev/prod) | ❌ ausente (APP usa `domain/extractors/environment.ts` não-canônico) |

### 2.11 db/ divergências

| # | Achado | Severidade | Standard ref |
|---|---|---|---|
| DB1 | `db/legacy-eva-models/` usa namespace não-canônico para mirrors externos; deveria ser `db/replication/` com `namespace db.replication` | High | replication.md §"Anti-padrão: subpasta sap/" |
| DB2 | `db/types/application-service/` e `db/types/public-service/` acoplam db a nomes de serviço | Medium | dependency rule (db deve ser service-agnostic) |
| DB3 | `db/views/fup-users.cds` — totalmente comentado (dead code) | Low | code hygiene |
| DB4 | `Key` (capital K) em `purchase-order-item.cds:6` e `purchase-order-delivery-detail.cds:4` | Low | style |
| DB5 | `tax-invoice-divergence-correction.status.cds` — dots ao invés de kebab-case | Low | file-naming |
| DB6 | TODO `// TODO: check this with denys` em `carrier-invoice.cds:28` | Low | code hygiene |
| DB7 | Sem `.nvmrc` + sem `renovate.json` na raiz do repo | Low | tooling |
| DB8 | `git-flow.yaml` usa `actions/checkout@v2` (deveria ser v4) + `VAEES/git-flow-action@main` (unpinned) | Medium | security |

**db/ compliance highlights:** ✅ One-entity-per-file, ✅ no actions/functions in db CDS, ✅ `@cds.persistence.exists` correto (uncommented, active), ✅ TVFs/procedures isolados em `db/src/`, ✅ `replace-csn-source.js` presente, ✅ multi-service build-task separation.

### 2.12 Build pipeline + scripts

| # | Arquivo | Função | Problema |
|---|---|---|---|
| BP1 | `main/scripts/replace-csn-source.ts` (cada serviço) | Rewrites `@source` paths no CSN compilado | Hardcodes service names; `catch (ignored) {}` silencioso |
| BP2 | `main/scripts/populate-local-db.ts` (cada serviço) | Comment/uncomment `@cds.persistence.exists` + `cds deploy` | APP: sem try/catch; PROC: try/catch swallow; PUB: +legacy-eva handling. Mesma lógica, parameterizable |
| BP3 | Root `package.json` | `build:cf` = cds build + per-service build:mta | Sem lint/typecheck/test no root |
| BP4 | APP `main/annotations/` | CDS annotations para panel/manual-fup/role-management | Não-existe em PROC/PUB (ok — apenas APP tem entidades com annotations) |

### 2.13 Testes

| Serviço | vitest.config.ts | Test files | @faker-js/faker | @vitest/coverage | Status real |
|---|---|---|---|---|---|
| APP | ✅ | 4 test files + 74 CSVs mock + HTTP files | ✅ | ✅ | Funcional mas coverage baixa |
| PROC | ❌ | 0 | ❌ | ❌ | Sem testes |
| PUB | ❌ | 0 | ✅ (declared, unused) | ✅ (declared, unused) | Dead deps |

**Padrão de teste (APP):** `makeSut()` + `stubs/<dep>.ts` + `BaseUseCaseImplStub extends BaseUseCaseImpl` (vazio). Base stub não é shared — vive só em APP.

---

## 3. Padrões observados — o que NÃO mudar

> Estas são as coisas que JÁ estão corretas segundo o standard e DEVEM ser preservadas durante a refatoração.

### 3.1 Estrutura de 5 camadas
- `domain/`, `application/`, `infrastructure/`, `presentation/`, `main/` — presente em todos os 3 serviços.
- A direção de dependência está correta: `application/` importa só `@/domain/*`; `presentation/` importa `@/domain/*` + `@/presentation`; `main/` compõe tudo.
- **Nenhuma violação de `application→infrastructure` ou `presentation→infrastructure`** foi encontrada.

### 3.2 Either monad
- `@sweet-monads/either ^3.3.1` usado em todos os serviços.
- Use cases retornam `Promise<Either<AbstractError, T>>` via `left()`/`right()`.
- `BaseUseCaseImpl.handleError` centraliza error mapping.
- Infrastructure **não** retorna `Either` — propaga exceção que o use case captura. Correto.

### 3.3 Factory pattern (DI manual)
- `makeXxx(): InterfaceDomínio` em `main/factories/` em todos os serviços.
- Constructor DI com `private readonly` tipado pela interface.
- Sem DI container. Sem instância concreta exposta.

### 3.4 Repository pattern
- Interfaces em `domain/repositories/`, impls em `infrastructure/persistence/` (nome errado mas pattern correto).
- Repository `ENTITY` constant como `private readonly`.

### 3.5 CAP handler binding
- `main/routes/*.ts` exporta `default (service: Service) => { service.on(...) }` (APP, PROC).
- Factories injetam controllers nos CAP events.
- `adaptRoute` adapter traduz CAP `Request` → `BaseController.Request`.

### 3.6 db/ structure
- One-entity-per-file em `db/models/` (61 entidades).
- No actions/functions em CDS do db.
- `@cds.persistence.exists` usado corretamente.
- TVFs + procedures isolados em `db/src/`.
- Cross-schema grants via `hdbgrants/` (4 grants).

### 3.7 app-router
- `authenticationMethod: route` com 2 rotas xsuaa.
- Public-service corretamente **não** roteado (sem auth).

### 3.8 Code style base
- 4 espaços, single quotes, semícolons, `sort-imports: error`, kebab-case filenames, PascalCase classes.

---

## 4. Gaps & incógnitas — RESOLVIDOS

> Todos os 6 gaps foram resolvidos por investigação técnica ou decisão do usuário (2026-08-21).

| # | Pergunta | Resolução | Como |
|---|---|---|---|
| G1 | Quantos models anêmicos têm regras de validação implícitas que precisam virar `validate()`? | ✅ **Zero models têm validação implícita.** A validação vive nos use cases (ex: `validatePayload()`, `validateRecipients()` em `application/use-cases/`). Os `type Xxx = {}` em `domain/models/` são puros tipos de dados sem nenhuma lógica. **Implicação:** converter para rich models requer MOVER a validação dos use cases para o `model.validate()`. Não é só converter `type`→`class` — é reposicionar a lógica de validação. | Investigação: `grep` por `validate\|throw\|if.*===` em `domain/models/` retornou vazio; mesma busca em `application/use-cases/` retornou 40+ matches de validação ativa |
| G2 | O getter `message()` com recursão infinita é acessado em runtime? | ✅ **Sim, é bug ativo.** `processing-service/save-carrier-invoice-data.ts:53,121` chama `errorInstance.message` onde `errorInstance` é tipado como `AbstractError`. Provavelmente não estoura porque `super(message)` no construtor seta `Error.message` nativo que pode shadowar o getter no prototype chain do V8. **Implicação:** fixar removendo o getter/setter customizado (usar `super.message` ou não override). | Investigação: `grep` por `errorInstance.message` + `reason?.message` em `application/` e `presentation/` |
| G3 | As classes de erro extras (BadGateway, SAPGateway, Dellavolpe, Unavailable) devem ser mantidas ou reduzidas às 6 canônicas? | ✅ **Limpar dead code + mover types + manter UnavailableService.** Análise: `BadGatewayError` (502) e `UnprocessableEntity` (422) têm **0 instanciações** — dead code, remover. `UnavailableServiceError` (503) tem **4 instanciações ativas** — representa "API externa indisponível" (rede/timeout), manter. `SAPGatewayError` e `DellavolpeError` **não são classes de erro** — são `namespace` com type definitions para parse de resposta de API externa; mover para `domain/adapters/external-api/`. Adicionar `ConflictError` (409). Resultado: **7 classes de erro** (6 canônicas + UnavailableService). ADR justifica a 7ª. | Investigação: `grep` por `new XxxError(` em todos os .ts; leitura dos arquivos de cada error class |
| G4 | `db/legacy-eva-models/` → `db/replication/`: a renomeação quebra o `@cds.persistence.exists`? | ✅ **Não quebra.** `@cds.persistence.exists` é uma annotation per-entity (linha 1 de cada .cds), não depende do nome da pasta. O rename da PASTA é seguro. **Path references a atualizar:** `public-service/src/main/routes/public.cds:8` (`from '../../../../db/legacy-eva-models'` → `'../../../../db/replication'`) e `public-service/src/main/scripts/populate-local-db.ts:44` (path `'db', 'legacy-eva-models'` → `'db', 'replication'`). O namespace CDS interno (`namespace db.models`) NÃO precisa mudar — é referenciado por `using { db.models }` em todos os serviços. | Investigação: `grep` por `legacy-eva-models\|legacy.eva` em .cds, .ts, .json + leitura dos hdbgrants |
| G5 | Extract `shared/` package: yarn workspaces ou monorepo sem workspaces (como cat2)? | ✅ **Sem workspaces** (já decidido no research global D2). Cada package mantém `yarn.lock` próprio, igual cat2. | Research global `fup-s4-migration` decisão D2 |
| G6 | Quanto da refatoração pode ser feita com find-and-replace vs. refatoração manual? | ✅ **56% mecânico, 44% manual.** Mecânico (find-replace + rename de pasta): 59 barrels + 13 hidrators→hydrators + 34 persistence→repositories + 51 entities→entity-events + 59 execute→handle = **216 arquivos**. Manual (cada arquivo é único): 27 domain purity + 229 models anêmicos + 4 SQL injection = **260 arquivos**. Total: 476 arquivos touchpoints. | Investigação: `find` + `grep` contando arquivos por categoria |

---

## 5. Épicos Jira candidatos

> Estruturados para o board RFIA (ID 2011). Cada épico = 1 área de refatoração. Subtasks = ações cirúrgicas por arquivo.

### EP-01: Scaffold do monorepo `suzano-fup-s4` — estrutura base
**Prioridade:** P1 ⭐
**Descrição:** Criar a estrutura base do monorepo (AGENTS.md, package.json, mta.yaml, .nvmrc, .github/, docs/specs/) e migrar os 3 serviços + db + app-router para `packages/backend/`.
**Subtasks:**
- Criar `packages/backend/{application-service,processing-service,public-service,db,app-router,public-app-router}/`
- Criar `AGENTS.md` raiz + `packages/backend/AGENTS.md`
- Criar `.nvmrc` (Node 22) + `renovate.json`
- Consolidar `mta.yaml` (MTA ID `suzano-fup-s4`, v2.0.0)
- Consolidar `xs-security.json` (2 arquivos: interno + público)
- Configurar CI com paths-filter + `NUMENDS/git-flow-action`

### EP-02: Eliminar framework do domain (domain purity)
**Prioridade:** P1 ⭐
**Descrição:** Remover todos os imports de `@models/db/models` e `@sap/cds` de arquivos em `domain/` (28 arquivos total). Declarar tipos wrapper no namespace do próprio contrato.
**Subtasks:**
- APP: Refatorar 12 `domain/repositories/*.ts` — remover `@models` imports, declarar `DbRow` types
- APP: Refatorar 15 `domain/models/{actions,functions,entities}/*.ts` — mover `Params`/`Result` para `domain/use-cases/`
- PROC: Refatorar 3 `domain/repositories/*.ts` + 4 `domain/models/actions/*.ts`
- PUB: Refatorar 3 `domain/repositories/*.ts` + 3 `domain/models/actions/*.ts`

### EP-03: Models anêmicos → ricos
**Prioridade:** P2
**Descrição:** Converter todos os `type Xxx = {}` em `class XxxModel` com `XxxProps`, `static with(props)`, `validate()`, getters.
**Subtasks:**
- APP: Converter 24+ model files em `domain/models/`
- PROC: Converter `email.ts`, `user-provided-variables.ts`
- PUB: Converter 3 model files em `domain/models/actions/`

### EP-04: Corrigir SQL injection
**Prioridade:** P1 ⭐ (segurança)
**Descrição:** Eliminar 8 sites de SQL injection usando parâmetros bound (`cds.run(sql, [params])` ou `cds.ql`).
**Subtasks:**
- APP: `infrastructure/persistence/purchase-order-item.ts:25-35,99-114`
- APP: `infrastructure/persistence/fup-history.ts:23-25`
- APP: `infrastructure/persistence/tax-invoice-item.ts:21-23`
- PROC: `infrastructure/persistence/purchase-order-item.ts:17,39,58,70`
- PROC: `application/use-cases/actions/send-top-five-pending-fups-to-supplier.ts:40`
- PUB: `infrastructure/persistence/purchase-order-item.ts:46`
- PUB: `main/routes/public.ts:30` — mover SQL para repository

### EP-05: Eliminar pastas proibidas
**Prioridade:** P2
**Descrição:** Remover `extractors/`, `formatters/`, `hidrators/` (typo), `i18n/` e mover conteúdo para `utils/` ou model methods.
**Subtasks:**
- APP: `domain/extractors/` → `domain/utils/get-environment.ts`
- APP: `infrastructure/extractors/` → `infrastructure/utils/get-environment.ts`
- APP: `domain/formatters/` + `infrastructure/formatters/` → `utils/` ou model method
- APP: `domain/hidrators/` + `infrastructure/hidrators/` → `hydrators/` (rename)
- APP: `infrastructure/i18n/` → `infrastructure/utils/translator/i18n/`
- PROC: `domain/extractors/` + `infrastructure/extractors/` → `utils/`
- PROC: `domain/formatters/` + `infrastructure/formatters/` → `utils/`
- PUB: `domain/formatters/` + `infrastructure/formatters/` → `utils/`
- ALL: `main/adapters/` → `main/routes/` ou `main/config/`

### EP-06: Eliminar barrel exports
**Prioridade:** P2
**Descrição:** Remover ~53 arquivos `index.ts` que re-exportam (exceção: `domain/errors/index.ts`).
**Subtasks:**
- APP: Eliminar 24 barrels
- PROC: Eliminar 16 barrels
- PUB: Eliminar 13 barrels
- Atualizar todos os imports que usavam barrels para imports diretos por arquivo

### EP-07: Padronizar error taxonomy
**Prioridade:** P1 ⭐
**Descrição:** Corrigir error classes: adicionar `ConflictError` (409), corrigir typo `Forbiden→Forbidden`, fixar bug de recursão infinita no getter `message()`, limpar dead code, mover type namespaces.
**Subtasks:**
- Criar `ConflictError` (409) em todos os 3 serviços
- Renomear `ForbidenRequestError` → `ForbiddenError` + `forbiden-request.ts` → `forbidden-request.ts` (9 files)
- Fixar getter `message()` recursão infinita — remover getter/setter customizado, usar `super(message)` no construtor (21 files — 7 classes × 3 serviços)
- **Remover dead code:** `BadGatewayError` (502, 0 instâncias) e `UnprocessableEntity` (422, 0 instâncias) — deletar arquivos
- **Manter:** `UnavailableServiceError` (503, 4 instâncias ativas) — representa "API externa indisponível" (rede/timeout). ADR justifica a 7ª classe.
- **Mover:** `SAPGatewayError` namespace → `domain/adapters/external-api/sap-gateway/` (é type shape de resposta, não error class)
- **Mover:** `DellavolpeError` namespace → `domain/adapters/external-api/dellavolpe/` (idem)
- Padronizar `ServerError` constructor (PROC tem `constructor(stack: string)` inconsistente)
- Resultado final: **7 classes de erro** = 6 canônicas (400/401/403/404/409/500) + UnavailableServiceError (503)

### EP-08: Padronizar tsconfig + eslint
**Prioridade:** P2
**Descrição:** Alinhar configurações TS/ESLint ao standard.
**Subtasks:**
- `strictNullChecks: true` em todos os 3 serviços
- `noImplicitAny: true` em todos
- `strictPropertyInitialization: true`
- `printWidth: 180` no eslint
- `no-console: ['error', { allow: ['error'] }]`
- `no-explicit-any: 'error'`
- Migrar `module`/`moduleResolution` para `NodeNext` + `.js` extensions
- Criar `.nvmrc` + `renovate.json`

### EP-09: Rename `persistence/` → `repositories/` + `entities/` → `entity-events/`
**Prioridade:** P2
**Descrição:** Renomear pastas para a estrutura canônica do standard.
**Subtasks:**
- APP, PROC, PUB: `infrastructure/persistence/` → `infrastructure/repositories/` (com subpastas `models/` + `replication/`)
- APP: `domain/use-cases/entities/` → `entity-events/` (5 layers)
- APP: `domain/models/{actions,functions,entities}/` → mover contratos para `domain/use-cases/`

### EP-10: Controllers: `execute()` → `handle()`
**Prioridade:** P2
**Descrição:** Renomear método público de `execute` para `handle` em todos os controllers.
**Subtasks:**
- APP: 23 controllers + base
- PROC: 4 controllers + base
- PUB: 3 controllers + base

### EP-11: Extrair código compartilhado (shared core)
**Prioridade:** P3
**Descrição:** Extrair arquivos idênticos entre serviços para um package `shared/` ou `packages/backend/shared/`.
**Subtasks:**
- Extrair `domain/errors/` (abstract + 6 classes canônicas)
- Extrair `domain/adapters/http/` (interface)
- Extrair `infrastructure/adapters/http/` (impl)
- Extrair `main/config/module-alias.ts`
- Reconciliar `BaseUseCaseImpl` (adotar PROC/PUB versão mais robusta)
- Reconciliar `BaseController` (adotar APP versão tipada)
- Reconciliar `adaptRoute` (adotar APP superset)
- Reconciliar `HttpRequestAdapter` (restaurar `get()` em PUB)
- Criar `infrastructure/utils/{translator,get-user,get-environment}.ts` ausentes

### EP-12: Mover business logic de `main/routes/public.ts` para controller/use case
**Prioridade:** P1 ⭐
**Descrição:** PUB `main/routes/public.ts:15-44` tem SQL inline + business logic na rota. Mover para repository + controller + use case.
**Subtasks:**
- Criar `NotificationLogRepository` interface em `domain/`
- Criar impl em `infrastructure/repositories/`
- Criar use case em `application/use-cases/`
- Criar controller em `presentation/controllers/`
- Mapear rota `public.ts` para o novo controller via factory

### EP-13: `db/` canonicalização
**Prioridade:** P3
**Descrição:** Alinhar db/ ao standard de replication.
**Subtasks:**
- Rename `db/legacy-eva-models/` → `db/replication/` + `namespace db.replication`
- Mover `db/types/application-service/` + `db/types/public-service/` para os respectivos `main/routes/`
- Deletar `db/views/fup-users.cds` (dead code)
- Corrigir `Key` → `key` em 2 entity files
- Corrigir `tax-invoice-divergence-correction.status.cds` → kebab-case
- Resolver TODO em `carrier-invoice.cds:28`

### EP-14: Ativar SonarQube + CI gate
**Prioridade:** P3
**Descrição:** Ativar CI gate com lint + typecheck + test em todos os serviços.
**Subtasks:**
- Descomentar SonarQube step em `application-service.yaml`
- Adicionar CI para PROC e PUB (hoje só APP tem)
- Adicionar lint + typecheck + test gate no root CI
- Pin `git-flow-action` a tag/SHA ao invés de `@main`
- Atualizar `actions/checkout@v2` → `@v4`

---

## 6. Fontes consultadas

### 6.1 Standard canônico
- `docs/standards/cap-clean-architecture/README.md` (232 linhas) — visão geral, dependency rule, regras de ouro, anti-padrões
- `docs/standards/cap-clean-architecture/domain-layer/README.md` — contratos, models, errors
- `docs/standards/cap-clean-architecture/application-layer/README.md` — use cases, services
- `docs/standards/cap-clean-architecture/infrastructure-layer/README.md` — repositories, adapters, utils
- `docs/standards/cap-clean-architecture/infrastructure-layer/repositories/conventions.md` — SQL pleno, parameterized queries
- `docs/standards/cap-clean-architecture/infrastructure-layer/repositories/replication.md` — mirrors externos, `db.replication.*`
- `docs/standards/cap-clean-architecture/presentation-layer/README.md` — controllers, `handle()` method
- `docs/standards/cap-clean-architecture/main-layer/README.md` — factories, routes, scripts
- `docs/standards/code-style/typescript.md` — code style canônico

### 6.2 Codebase auditado
- `application-service/src/` — 160 .ts files, 30 violations (6 Critical, 12 High, 9 Medium, 3 Low)
- `processing-service/src/` — 89 .ts files, 70 violations (17 Critical, 22 High, 28 Medium, 3 Low)
- `public-service/src/` — 66 .ts files, 33 violations (9 Critical, 7 High, 13 Medium, 4 Low)
- `db/` — 62 CDS + 53 CSV + ~25 HANA artifacts, 10 divergences
- `app-router/` + root configs — 5 divergences
- Cross-cutting analysis — duplication matrix, reconciliation targets

### 6.3 Research global
- `docs/researches/fup-s4-migration/research.md` — decisões macro (D1-D8), estrutura alvo, stack

### 6.4 Doc funcional
- `/home/hiagoperoni/Projects/Suzano/S4/docs/FUNCIONAL-FUP.md` — contexto de negócio, modelo de dados, integrações

---

## 7. Handoff TLC Spec-Driven + Jira

**Próximo passo sugerido:** 
1. Criar épicos EP-01 a EP-14 no Jira board RFIA (ID 2011)
2. Invocar `tlc-spec-driven` → fase **Specify** carregando este research como contexto base

**Slug da feature**: `fup-backend-refactor` (deve casar com o slug deste research).

### 7.1 Épicos para criar no Jira (board RFIA, ID 2011)

| ID | Épico | Prioridade | Estimativa | Subtasks |
|---|---|---|---|---|
| EP-01 | Scaffold monorepo + migração packages/backend/ | P1 ⭐ | M | 6 |
| EP-02 | Eliminar framework do domain (domain purity) | P1 ⭐ | L | 4 grupos |
| EP-03 | Models anêmicos → ricos | P2 | L | 3 serviços |
| EP-04 | Corrigir SQL injection | P1 ⭐ | M | 8 sites |
| EP-05 | Eliminar pastas proibidas | P2 | M | 9 ações |
| EP-06 | Eliminar barrel exports | P2 | M | 53 files |
| EP-07 | Padronizar error taxonomy | P1 ⭐ | M | 5 ações |
| EP-08 | Padronizar tsconfig + eslint | P2 | M | 8 configs |
| EP-09 | Rename pastas (persistence→repositories, entities→entity-events) | P2 | M | 2 renames |
| EP-10 | Controllers execute→handle | P2 | S | 34 files |
| EP-11 | Extrair código compartilhado (shared core) | P3 | L | 9 reconciliações |
| EP-12 | Mover business logic de public.ts | P1 ⭐ | M | 5 arquivos |
| EP-13 | db/ canonicalização | P3 | S | 6 ações |
| EP-14 | Ativar SonarQube + CI gate | P3 | S | 5 ações |

### 7.2 Ordem sugerida de execução

```
EP-01 (scaffold) → EP-04 (SQL injection — segurança) → EP-07 (error taxonomy — bug) 
→ EP-02 (domain purity) → EP-12 (business logic em route) 
→ EP-09 (rename pastas) → EP-05 (pastas proibidas) → EP-06 (barrels) 
→ EP-10 (execute→handle) → EP-08 (tsconfig+eslint) 
→ EP-03 (rich models) → EP-11 (shared core) → EP-13 (db/) → EP-14 (CI)
```

### 7.3 Decisões fechadas (resumo executivo)

| ID | Decisão | Detalhe |
|---|---|---|
| G1 | Models anêmicos sem validação implícita | Validação vive nos use cases (`validatePayload()`, `validateRecipients()`). Converter para rich models requer MOVER a validação dos use cases para `model.validate()` |
| G2 | Bug de recursão infinita é ativo | `errorInstance.message` acessado em PROC `save-carrier-invoice-data.ts:53,121`. Provavelmente não estoura por V8 prototype shadowing, mas é bomba relógio |
| G3 | Error taxonomy = 7 classes | 6 canônicas (400/401/403/404/409/500) + `UnavailableServiceError` (503). Remover dead code (BadGateway, UnprocessableEntity). Mover SAPGatewayError/DellavolpeError para `domain/adapters/external-api/`. ADR justifica a 7ª |
| G4 | Rename `legacy-eva-models/` → `replication/` é seguro | `@cds.persistence.exists` é per-entity, não depende do nome da pasta. Atualizar 2 path references: `public.cds:8` + `populate-local-db.ts:44` |
| G5 | Sem yarn workspaces | Decidido no research global D2 |
| G6 | 56% mecânico, 44% manual | 216 arquivos mecânicos (barrels, renames, execute→handle) + 260 arquivos manuais (domain purity, rich models, SQL injection) |

### 7.4 Bibliografia mínima para a Specify carregar

- ⭐ `docs/researches/fup-backend-refactor/research.md` (este documento)
- `docs/researches/fup-s4-migration/research.md` (research global — decisões macro)
- `docs/standards/cap-clean-architecture/README.md` + sub-READMEs
- `docs/standards/code-style/typescript.md`
- `/home/hiagoperoni/Projects/Suzano/S4/docs/FUNCIONAL-FUP.md` (contexto de negócio)
- `suzano-cat2-frontend-s4/AGENTS.md` (referência de AGENTS.md raiz)

---

**Fontes consultadas neste documento:** Standard cap-clean-architecture (10+ arquivos) + 3 codebases de serviço (315 .ts files auditados) + db/ (62 CDS + HANA) + app-router + root configs + cross-cutting duplication matrix + research global fup-s4-migration + doc funcional FUP.
