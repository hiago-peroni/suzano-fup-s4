# Pesquisa: Refatoração FUP S4 para Monorepo IAtização

> Mapeamento detalhado dos 3 repositórios FUP S4 (2 frontends + 1 backend com 3 serviços CAP), preparando a consolidação em um único monorepo `suzano-fup-s4` no padrão IAtização, com deploy MTA unificado e AGENTS.md em hierarquia.

| Item | Valor |
|---|---|
| Slug | `fup-s4-migration` |
| Data | 2026-08-20 |
| Modo | Profundo |
| Consumidor previsto | `tlc-spec-driven` (Specify) |
| Fontes utilizadas | Codebase atual (3 projetos) + Codebase modelo (`suzano-cat2-frontend-s4`, `numen-software-house`) + Docs internas (`docs/FUNCIONAL-FUP.md`, scaffold prompt, standards) |
| Status | Final |
| Autor | Agente Kilo |

---

## Sumário

1. [Visão geral comparativa](#1-visão-geral-comparativa)
2. [Achados por dimensão](#2-achados-por-dimensão)
   - 2.1 [Estrutura de repositórios](#21-estrutura-de-repositórios)
   - 2.2 [Stack e versões](#22-stack-e-versões)
   - 2.3 [Clean Architecture](#23-clean-architecture)
   - 2.4 [Configurações (tsconfig, eslint, vitest, ui5)](#24-configurações-tsconfig-eslint-vitest-ui5)
   - 2.5 [Deploy MTA e Cloud Foundry](#25-deploy-mta-e-cloud-foundry)
   - 2.6 [Autenticação e segurança](#26-autenticação-e-segurança)
   - 2.7 [Integrações e dependências cruzadas](#27-integrações-e-dependências-cruzadas)
   - 2.8 [Testes](#28-testes)
   - 2.9 [Padrão monorepo alvo (IAtização)](#29-padrão-monorepo-alvo-iatização)
3. [Padrões observados](#3-padrões-observados)
4. [Gaps & incógnitas](#4-gaps--incógnitas)
5. [Recomendações como hipóteses](#5-recomendações-como-hipóteses)
6. [Fontes consultadas](#6-fontes-consultadas)
7. [Handoff TLC Spec-Driven](#7-handoff-tlc-spec-driven)

---

## 1. Visão geral comparativa

### 1.1 Estado atual — 3 repositórios separados

| Projeto | MTA ID | Versão | Tipo | # Módulos | Auth | Deploy |
|---|---|---|---|---|---|---|
| `suzano-fup-panel-frontend-s4` | `fup-panel-frontend-s4` | 1.0.0 | Frontend UI5 | 3 apps (fup-panel, manual-fups, role-management) | XSUAA | MTA + CF |
| `suzano-fup-public-frontend-s4` | `fup-public-frontend-s4` | 1.0.2 | Frontend UI5 | 1 app (dynamic-forms) + app-router | **Nenhuma** | MTA + CF |
| `suzano-fup-panel-backend-s4` | `fup-backend-s4` | 1.0.8 | Backend CAP | 3 serviços (application, processing, public) + db-deployer + app-router | XSUAA + Job Scheduler | MTA + CF |

### 1.2 Estrutura alvo — 1 monorepo consolidado

| Package | Origem | Apps/Serviços | Path alvo |
|---|---|---|---|
| `packages/frontend/` | fup-panel-frontend + fup-public-frontend | fup-panel, manual-fups, role-management, dynamic-forms | `packages/frontend/<app-name>/` |
| `packages/backend/` | fup-panel-backend | application-service, processing-service, public-service, db, app-router | `packages/backend/<service-name>/` |

### 1.3 Tabela mestra de módulos

| Módulo | Repo origem | App ID / Service | OData Path | Stack | Clean Arch? | Tests? |
|---|---|---|---|---|---|---|
| fup-panel | fup-panel-frontend | `fup.fuppanels4` | `/fup/panel/` | UI5 1.120.39 + TS 5.1 | ✅ 5 camadas | ❌ stub only |
| manual-fups | fup-panel-frontend | `fup.manuals4` | `/fup/manual/` | UI5 1.120.39 + TS 5.1 | ✅ 5 camadas | ❌ nenhum |
| role-management | fup-panel-frontend | `fup.rolemanagements4` | `/fup/role-management/` | UI5 1.130.11 + TS 5.1 | ❌ flat FE template | ❌ nenhum |
| dynamic-forms | fup-public-frontend | `fup.dynamicformss4` | `/fup/public/` | UI5 1.120.x + TS 5.9.3 | ✅ 5 camadas | ❌ nenhum |
| application-service | fup-panel-backend | PanelService | `/fup/panel` | CAP 9.3.1 + TS 5.9.2 | ✅ 5 camadas | ⚠️ scripts existem, dir ausente |
| processing-service | fup-panel-backend | ProcessingService | `/fup-panel-processing` | CAP 9.3.1 + TS 5.9.2 | ✅ 5 camadas | ❌ sem test scripts |
| public-service | fup-panel-backend | PublicService | `/fup/public` | CAP 9.3.1 + TS 5.9.2 | ✅ 5 camadas | ⚠️ scripts existem, status incerto |
| db (HDI) | fup-panel-backend | — | — | HANA HDI + CDS | — | — |

### Conclusões iniciais

- Os 3 repositórios formam um sistema coeso (FUP) dividido artificialmente em 3 repos. A consolidação em monorepo é natural e reduz fricção de versionamento.
- **4 dos 7 módulos já seguem Clean Architecture** (fup-panel, manual-fups, dynamic-forms, backend services). Apenas `role-management` é um flat FE template sem camadas.
- **Versões de TS e UI5 divergem entre frontends**: fup-panel/manual-fups usam TS 5.1 + UI5 1.120.39; dynamic-forms usa TS 5.9.3 + UI5 1.120.x (types 1.143); role-management usa UI5 1.130.11. Padronizar é necessário.
- **Backend usa CAP 9.3.1 + TS 5.9.2** — mais recente que os frontends. A consolidação deve alinhar versões.
- **3 MTA IDs distintos** precisam ser unificados em um único `mta.yaml` na raiz do monorepo.
- **2 xs-security.json** (backend com scopes JOBSCHEDULER + uaa.user; frontend panel com uaa.user; public sem auth) precisam ser consolidados.
- **2 app-routers** (backend roteia `/fup-panel-application` + `/fup-panel-processing`; public serve `/fupdynamicformss4` sem auth) — decisão de arquitetura sobre mantê-los separados ou unificar.

---

## 2. Achados por dimensão

### 2.1 Estrutura de repositórios

| # | Tipo | Achado | Evidência | Impacto |
|---|---|---|---|---|
| E1 | Padrão consistente | 3 repos seguem o mesmo pattern: root orquestrador (package.json com scripts `bd/build/deploy/undeploy/clean` + mta.yaml + yarn.lock) + subpastas para apps | `suzano-fup-panel-frontend-s4/package.json:1-16`, `suzano-fup-public-frontend-s4/package.json:1-16`, `suzano-fup-panel-backend-s4/package.json:1-23` | Alto — facilita migração |
| E2 | Divergência | Frontend panel NÃO tem workspaces config; apps são independentes com yarn.lock próprio | `suzano-fup-panel-frontend-s4/` (sem `workspaces` no root package.json) | Médio — monorepo workspaces exigirá ajuste |
| E3 | Divergência | Backend tem 4 subpastas com package.json próprio (application-service, processing-service, public-service, app-router) + db/ com package.json próprio, mas SEM workspaces config | `suzano-fup-panel-backend-s4/package.json:1-23` (root sem `workspaces`) | Alto —迁移需要配置 workspaces |
| E4 | Lacuna | Nenhum dos 3 repos tem `AGENTS.md` | `ls suzano-fup-panel-frontend-s4/AGENTS.md` → não existe; idem outros 2 | Alto — padrão IAtização exige AGENTS.md |
| E5 | Lacuna | Nenhum dos 3 repos tem `docs/` com standards/specs/researches | verificação via `find` — nenhum tem pasta `docs/` | Alto — padrão IAtização exige docs/ |
| E6 | Padrão consistente | Todos têm `.github/workflows/git-flow.yaml` com `VAEES/git-flow-action` | `suzano-fup-panel-frontend-s4/.github/workflows/git-flow.yaml:14`, `suzano-fup-panel-backend-s4/.github/workflows/git-flow.yaml` | Baixo — já alinhado |
| E7 | Divergência | Git flow action diverge: frontend usa `VAEES/git-flow-action`, cat2 usa `NUMENDS/git-flow-action` | `suzano-fup-panel-frontend-s4/.github/workflows/git-flow.yaml:14` vs `suzano-cat2-frontend-s4/.github/workflows/git-flow.yaml:16` | Médio — padronizar org |
| E8 | Divergência | `suzano-fup-s4` (destino) já tem git repo com branch `feat/hiago-project-base`, `main`, `quality`; standards copiados, mas sem package.json/mta.yaml/AGENTS.md | `git log` mostra 2 commits; `find` mostra apenas docs/standards + README.md | Alto — scaffold base faltante |

### 2.2 Stack e versões

| # | Tipo | Achado | Evidência | Impacto |
|---|---|---|---|---|
| S1 | Divergência | **TypeScript**: frontends panel = 5.1.6; public = 5.9.3; backend = 5.9.2 | `suzano-fup-panel-frontend-s4/fup-panel/package.json`, `suzano-fup-public-frontend-s4/dynamic-forms/package.json`, `suzano-fup-panel-backend-s4/application-service/package.json` | Alto — padronizar para 5.9.x |
| S2 | Divergência | **UI5 versão**: fup-panel/manual-fups = 1.120.39; role-management = 1.130.11; dynamic-forms = 1.120.x (types 1.143) | manifest.json de cada app | Alto — definir versão canônica |
| S3 | Divergência | **@ui5/cli**: frontends panel = ^3.11.13; public = ^4.0.38 | package.json de cada app | Alto — major version bump (3→4) |
| S4 | Divergência | **Vitest**: frontends panel = ^2.1.9; public = ^4.0.16; backend = 3.1.3 | package.json de cada app | Médio — 3 versões diferentes |
| S5 | Divergência | **ESLint**: frontends panel = ^9.12.0; public = ^9.39.2; backend = ^9.35.0 | package.json de cada app | Baixo — todas 9.x flat config |
| S6 | Padrão consistente | **@sweet-monads/either ^3.3.1** em TODOS os 7 módulos (frontends + backend) | package.json de cada módulo | — — já alinhado |
| S7 | Divergência | **Node**: backend = `engines.node: ^22`; frontends panel = sem engines; public = `engines.node: ^20\|\|^22` (app-router) | package.json de cada módulo | Alto — definir Node 22 |
| S8 | Padrão consistente | **@sapui5/types** em todos frontends; **@cap-js/cds-typer** no backend | package.json de cada módulo | — |
| S9 | Divergência | **printWidth ESLint**: frontends panel = 180 (fup-panel) / 150 (role-mgmt); public = 180; backend = 120 | eslint.config.mjs de cada app | Médio — padronizar para 180 |
| S10 | Divergência | **Prettier**: frontends panel = instalado e wired; public = instalado mas **NÃO wired** (sem .prettierrc, não carregado no eslint); backend = instalado e wired | eslint.config.mjs + .prettierrc de cada app | Médio |
| S11 | Divergência | **moduleResolution**: frontends = `node`; backend = sem explicito (commonjs) | tsconfig.json de cada app | Médio |

### 2.3 Clean Architecture

| # | Tipo | Achado | Evidência | Impacto |
|---|---|---|---|---|
| C1 | Padrão consistente | **5 camadas idênticas** em 6 dos 7 módulos: `domain/`, `application/`, `infrastructure/`, `presentation/`, `main/` | estrutura de pastas em fup-panel, manual-fups, dynamic-forms, application-service, processing-service, public-service | — — já alinhado com cap-clean-arch standard |
| C2 | Divergência | `role-management` é flat FE template (apenas Component.ts + i18n), sem camadas Clean Arch | `suzano-fup-panel-frontend-s4/role-management/webapp/` | Médio — decidir se refatora ou mantém |
| C3 | Padrão consistente | **Either monad** em use cases: `Either<AbstractError, T>` com `right()`/`left()` | todos os módulos com Clean Arch | — |
| C4 | Padrão consistente | **Factory pattern** (DI manual): `make<Name>(): Interface` + singleton exportado em `main/factories/` | todos os módulos com Clean Arch | — |
| C5 | Padrão consistente | **Repository interface + Impl**: domain/interfaces + infrastructure/Impl | todos os módulos com Clean Arch | — |
| C6 | Padrão consistente | **AbstractError hierarchy** mapeada de HTTP codes | `domain/errors/abstract.ts` em todos os módulos | — |
| C7 | Divergência | **Backend tem camadas extras** não presentes nos frontends: `extractors/`, `hidrators/` (CAP query modifiers), `controllers/` (presentation) | `suzano-fup-panel-backend-s4/application-service/src/domain/hidrators/`, `presentation/controllers/` | Baixo — específico de CAP, documentado no standard |
| C8 | Padrão consistente | **Namespace-typed models**: `export namespace X { Params, Result }` ao invés de arquivos `.types.ts` | `domain/models/actions/update-purchase-order-item.ts:7-10` (frontend), `domain/models/` (backend) | — |
| C9 | Divergência | **Barrel exports**: backend usa `index.ts` extensivamente; frontends NÃO usam (proibido no padrão) | `suzano-fup-panel-backend-s4/application-service/src/*/index.ts` vs eslint `no-barrel` nos frontends | Médio — divergência entre padrão frontend vs backend |
| C10 | Divergência | **Import aliases**: frontends = `fup/<app>/*` (namespace UI5); backend = `@/*`, `@models/*`, `@tests/*` (module-alias) | tsconfig.json de cada módulo | Médio — monorepo precisa de estratégia unificada |
| C11 | Anti-padrão | **`any` pervasivo**: ~30+ ocorrências em fup-panel, ~28 em dynamic-forms, extensivo no backend (BaseController, SAP request models) | grep `any` em cada módulo; `no-explicit-any: off` em todos os eslint configs | Alto — débito técnico |
| C12 | Anti-padrão | **SQL injection risk** no backend: string-interpolated SQL em repositories | `infrastructure/persistence/purchase-order-item.ts:31-33,99-101`, `public-service/src/main/routes/public.ts:30` | Alto — segurança |

### 2.4 Configurações (tsconfig, eslint, vitest, ui5)

| # | Tipo | Achado | Evidência | Impacto |
|---|---|---|---|---|
| CF1 | Divergência | **tsconfig strictness**: frontends = `strict: true` mas `strictPropertyInitialization: false`; backend = `strict: true` mas `strictNullChecks: false`, `strictPropertyInitialization: false`, `noImplicitAny: false` | tsconfig.json de cada módulo | Alto — backend menos strict |
| CF2 | Divergência | **eslint max-len/printWidth**: fup-panel = 180; role-management = 150; dynamic-forms = 180; backend = 120 | eslint.config.mjs de cada módulo | Médio — padronizar |
| CF3 | Lacuna | **dynamic-forms tem Prettier instalado mas não configurado** (sem .prettierrc, não carregado no eslint.config.mjs) | `suzano-fup-public-frontend-s4/dynamic-forms/eslint.config.mjs` | Médio |
| CF4 | Padrão consistente | **eslint flat config** em todos os módulos (ESLint 9.x) | eslint.config.mjs em todos | — |
| CF5 | Padrão consistente | **4 espaços, single quotes, semícolons, sort-imports: error, no-console: error** em todos | eslint.config.mjs em todos | — |
| CF6 | Divergência | **vitest.config.ts**: apenas fup-panel e dynamic-forms têm; backend application-service tem; processing-service não tem | vitest.config.ts em cada módulo | Médio |
| CF7 | Divergência | **ui5-deploy.yaml/ui5-local.yaml**: todos frontends têm, mas versões de tasks divergem (ui5-tooling-transpile 3.3.7 vs 3.9.2; ui5-tooling-modules 3.16.6 vs 3.34.0) | ui5.yaml de cada app | Médio |
| CF8 | Lacuna | **Sem .nvmrc** em nenhum dos 3 repos | `find .nvmrc` → não existe | Alto — padrão IAtização exige |
| CF9 | Lacuna | **Sem renovate.json** no backend; frontends têm mas com configs divergentes | `find renovate.json` | Médio |

### 2.5 Deploy MTA e Cloud Foundry

| # | Tipo | Achado | Evidência | Impacto |
|---|---|---|---|---|
| M1 | Divergência | **3 MTA IDs distintos**: `fup-panel-frontend-s4` (v1.0.0), `fup-public-frontend-s4` (v1.0.2), `fup-backend-s4` (v1.0.8) | mta.yaml de cada repo | Alto — unificar em 1 MTA |
| M2 | Divergência | **MTA schema**: frontends = 3.2; backend = 3.3.0 | mta.yaml header de cada repo | Baixo — 3.3.0 é compatível |
| M3 | Divergência | **deploy_mode**: frontends = `html5-repo`; backend = sem deploy_mode (nodejs modules) | mta.yaml de cada repo | Médio — MTA unificado precisa de ambos |
| M4 | Padrão consistente | **Recursos CF compartilhados**: destination (lite, UI5 CDN), html5-apps-repo (app-host), xsuaa (application) | mta.yaml de cada repo | — |
| M5 | Divergência | **Backend requer recursos extras**: HANA HDI, connectivity, job-scheduler, + 4 cross-service grants (s4-replication-db, eva-cap-db, eva-legacy-db, s4hana-remote-source-access-ecc) | `suzano-fup-panel-backend-s4/mta.yaml` resources | Alto — MTA unificado deve incluir todos |
| M6 | Divergência | **Backend build:cf** usa script complexo: `cds build` + per-service `build:mta` + `replace-csn-source.js` | `suzano-fup-panel-backend-s4/package.json:7-8` | Alto — script de build precisa preservar lógica CAP |
| M7 | Padrão consistente | **Scripts raiz idênticos**: `bd`, `build`, `deploy`, `undeploy`, `clean` | package.json de todos os 3 repos | — — facilita unificação |
| M8 | Divergência | **app-router**: backend tem 1 (roteia application + processing); public tem 1 (serve dynamic-forms sem auth); frontend panel **não tem** | app-router/ em cada repo | Alto — decisão de arquitetura |

### 2.6 Autenticação e segurança

| # | Tipo | Achado | Evidência | Impacto |
|---|---|---|---|---|
| A1 | Divergência | **3 configs XSUAA distintas**: backend (xsappname `fup-backend-s4`, scopes JOBSCHEDULER + uaa.user); frontend panel (xsappname `fup-panel-frontend-s4`, scope uaa.user); public (**sem xs-security.json**, auth none) | xs-security.json de cada repo | Alto — consolidar |
| A2 | Padrão consistente | **Token Exchange role template** presente em backend e frontend panel | xs-security.json de ambos | — |
| A3 | Divergência | **oauth2-configuration redirect-uris** divergem: backend inclui `localhost:5000`, `hanacloud-*`; frontend panel inclui `cfapps.*` | xs-security.json de cada repo | Médio |
| A4 | Anti-padrão | **SQL injection**: string interpolation em queries do backend (não usa parâmetros bound consistentemente) | `infrastructure/purchase-order-item.ts:31-33`, `public.ts:30` | Alto — segurança |
| A5 | Lacuna | **Public service sem auth**: access control apenas por UUID v4 no referer (expira em 5 dias) | `docs/FUNCIONAL-FUP.md:187` | Médio — documentar como constraint |

### 2.7 Integrações e dependências cruzadas

| # | Tipo | Achado | Evidência | Impacto |
|---|---|---|---|---|
| I1 | Padrão consistente | **Frontend → Backend via destinations**: `SUZANO_FUP_PANEL_APPLICATION_SERVICE_S4` (frontend panel), `/fup/public/` (public) | xs-app.json + environment.ts de cada frontend | — |
| I2 | Divergência | **Backend app-router** roteia `/fup-panel-application/(.*)` → `application-api` e `/fup-panel-processing/(.*)` → `processing-api`, mas **public-service não tem rota no app-router** (acessado via link UUID direto) | `suzano-fup-panel-backend-s4/app-router/xs-app.json` | Médio — arquitetura de roteamento |
| I3 | Padrão consistente | **Cross-schema HANA grants**: backend acessa 4 schemas externos (s4-replication, eva-cap, eva-legacy, s4hana-remote-source-access-ecc) via hdbgrants + synonyms | `db/src/hdbgrants/` (4 arquivos) | Alto — preservar no MTA unificado |
| I4 | Padrão consistente | **@cds-models/ tipos compartilhados**: frontends usam `@cds-models/*` gerados do backend CAP (entidades PurchaseOrderItems, Suppliers, etc.) | tsconfig.json `@models/*` path de cada frontend | Alto — monorepo pode gerar tipos do backend para os frontends |
| I5 | Divergência | **Destinations backend**: `GTW_PU` (SAP OData), `SUZANO_AZURE_AD` (odata-v4), `SUZANO_FUP_DELLAVOLPE_INTEGRATION_API` (rest) | `.cdsrc.json` prod config | Médio — preservar |
| I6 | Padrão consistente | **db/ compartilhado**: 62 entidades CDS + views + table-functions + procedures + synonyms, servindo todos os 3 serviços backend | `db/models/index.cds:1-62`, `db/src/` | Alto — db/ deve ficar em `packages/backend/db/` |

### 2.8 Testes

| # | Tipo | Achado | Evidência | Impacto |
|---|---|---|---|---|
| T1 | Anti-padrão | **Testes praticamente inexistentes**: fup-panel tem 1 arquivo stub (7 linhas); manual-fups, role-management, dynamic-forms = zero testes; backend application-service tem scripts mas dirs ausentes; processing-service sem scripts | `find tests/` em cada módulo | Alto — débito crítico |
| T2 | Lacuna | **Sem threshold de cobertura** configurado em nenhum módulo | vitest.config.ts de cada módulo | Médio |
| T3 | Padrão consistente | **Vitest + jsdom** (frontends) / **Vitest + node** (backend) quando presente | vitest.config.ts | — |
| T4 | Lacuna | **CI não roda testes** em nenhum dos frontends public; backend application-service roda `test:coverage` | `.github/workflows/` de cada repo | Alto |

### 2.9 Padrão monorepo alvo (IAtização)

| # | Tipo | Achado | Evidência | Impacto |
|---|---|---|---|---|
| P1 | Padrão consistente | **suzano-fup-s4 (destino)** já tem: git repo (branches main/quality/feat), docs/standards/ copiados (cap-clean-arch + code-style + react-clean-arch), docs/obsidian symlink | `find suzano-fup-s4` + `git log` | — |
| P2 | Lacuna | **suzano-fup-s4 NÃO tem**: AGENTS.md, README.md (apenas stub), package.json, mta.yaml, xs-security.json, .nvmrc, .gitignore, renovate.json, .github/, scripts/, docs/specs/, packages/ | `find suzano-fup-s4 -type f` | Alto — scaffold base faltante |
| P3 | Padrão consistente | **Scaffold prompt** já escrito em `docs/obsidian/Obsidian/IAtizacao/Prompt - Scaffold Monorepo.md` com 16 templates de arquivos | leitura do arquivo | — |
| P4 | Padrão consistente | **Codebase modelo** `suzano-cat2-frontend-s4` é a referência canônica: monorepo com packages/frontend/, AGENTS.md em hierarquia, mta.yaml, .nvmrc, renovate.json, CI + git-flow | `suzano-cat2-frontend-s4/` estrutura completa | — |
| P5 | Divergência | **cat2 usa NUMENDS/git-flow-action; FUP usa VAEES/git-flow-action** — org diverge | git-flow.yaml de cada repo | Médio — decidir org canônica |

---

## 3. Padrões observados

### 3.1 Padrões consistentes

- **Root orquestrador**: Todos os 3 repos têm package.json raiz com scripts `bd/build/deploy/undeploy/clean` + mta.yaml + yarn.lock. A raiz não tem código — apenas orquestra build/deploy.
- **Clean Architecture 5 camadas**: `domain/` → `application/` → `infrastructure/` → `presentation/` → `main/`. Presente em 6 dos 7 módulos. Única exceção: `role-management` (flat FE template).
- **Either monad**: `@sweet-monads/either ^3.3.1` em todos os 7 módulos. Use cases retornam `Either<AbstractError, T>`.
- **Factory DI**: `make<Name>(): Interface` + singleton em `main/factories/`. Sem DI container.
- **Repository pattern**: Interface em `domain/repositories/` + Impl em `infrastructure/persistence/`.
- **AbstractError hierarchy**: Classes de erro mapeadas de HTTP codes (400 BadRequest, 401 Unauthorized, 403 Forbiden [sic], 404 NotFound, 500 Server).
- **Namespace-typed models**: `export namespace X { Params, Result }` ao invés de arquivos `.types.ts`.
- **ESLint 9 flat config**: 4 espaços, single quotes, semícolons, `sort-imports: error`, `no-console: error`.
- **Git flow**: `feature/*` → PR `quality` → merge `main` via git-flow-action.

### 3.2 Anti-padrões / débitos

- **`any` pervasivo**: ~60+ ocorrências somadas entre frontends + extensivo no backend. `no-explicit-any: off` em todos os eslint configs tolera o débito. Detectado em: `infrastructure/adapters/http/request.ts`, `presentation/utils/dialog.ts`, `BaseController`, SAP request models.
- **Testes inexistentes**: 1 arquivo stub em fup-panel (7 linhas); zero nos demais frontends; backend application-service tem scripts mas diretórios de teste ausentes; processing-service sem scripts de teste.
- **SQL injection risk**: String interpolation em queries SQL do backend (`infrastructure/persistence/purchase-order-item.ts:31-33,99-101`, `public.ts:30`). Apenas alguns parâmetros usam bound `?`.
- **Typo**: `ForbidenRequestError` / `forbiden-request.ts` (deveria ser "Forbidden"). Presente em dynamic-forms.
- **role-management sem Clean Arch**: Flat FE template quebrando o padrão consistente dos demais módulos.
- **Duplicate package names**: `fup-panel/package.json:2` e `manual-fups/package.json:2` ambos com `name: "fup-panel-frontend-s4"`.
- **UI5 version drift**: 1.120.39 (fup-panel/manual-fups), 1.130.11 (role-management), 1.120.x com types 1.143 (dynamic-forms).
- **Prettier não wired no dynamic-forms**: Instalado mas não carregado no eslint.config.mjs.
- **`populate-local-db.ts` hack**: Mutates `db/models` source files in place (comenta/descomenta `@cds.persistence.exists`).
- **`replace-csn-source.ts`**: Hardcodes service names, silent `catch (ignored)`.
- **SonarQube CI comentado**: `.github/workflows/application-service.yaml:49-164` — step inteiro comentado.
- **`strictNullChecks: false` + `noImplicitAny: false`** no backend (todos os 3 serviços).
- **Sem .nvmrc** em nenhum dos 3 repos.
- **Sem `engines`** nos package.json dos frontends panel.

---

## 4. Gaps & incógnitas — RESOLVIDOS

> Todos os gaps foram resolvidos por investigação técnica ou decisão do usuário (2026-08-20).

| # | Pergunta | Resolução | Como |
|---|---|---|---|
| G1 | Unificar os 2 app-routers ou manter separados? | ✅ **Manter 2 separados** | Decisão do usuário |
| G2 | Consolidar 3 xs-security.json em 1? | ✅ **Manter 2** (interno com XSUAA + público sem auth) | Segue de G1 — público não tem xs-security.json |
| G3 | Padronizar UI5 para qual versão? | ✅ **UI5 1.120.39** | Investigação: é o minUI5Version comum a todos os apps (fup-panel, manual-fups, dynamic-forms min=1.120.39). role-management usa 1.130.11 mas será refatorado (G5) e downgraded |
| G4 | Padronizar @ui5/cli v3 ou v4? | ✅ **@ui5/cli v3 (^3.11.13)** | Investigação: 3 de 4 apps usam v3; cat2 (referência canônica) usa v3; dynamic-forms (único com v4) será downgraded |
| G5 | role-management: refatorar para Clean Arch? | ✅ **Refatorar para Clean Arch 5 camadas** | Decisão do usuário |
| G6 | 1 MTA unificado ou sub-MTAs? | ✅ **1 MTA unificado** (`suzano-fup-s4`, v2.0.0) | Decisão do usuário |
| G7 | Configurar yarn workspaces? | ✅ **NÃO usar workspaces** | Investigação: cat2 (referência canônica) NÃO usa workspaces; apps UI5 com @ui5/cli não foram testados com workspaces; manter pattern de cat2 (cada package com yarn.lock próprio) |
| G8 | @cds-models/ no monorepo? | ✅ **Gerar do backend via cds-typer** | Investigação: frontends têm `@cds-models/` estática hoje (PanelService, ManualFupService, PublicService). No monorepo, configurar script que roda `cds-typer` em `packages/backend/db/` → gera tipos para `packages/frontend/<app>/@cds-models/`. P1: manter cópia estática; P2: automatizar geração |
| G9 | Backend `db/` path? | ✅ **`packages/backend/db/`** | Decisão do usuário |
| G10 | Versão inicial do MTA? | ✅ **2.0.0** | Decisão do usuário (major bump indicando breaking change de consolidação) |
| G11 | Git flow action: NUMENDS ou VAEES? | ✅ **NUMENDS** | Investigação: todos os 3 repos FUP estão sob `github.com/NUMENDS/`; cat2 também usa NUMENDS/git-flow-action |
| G12 | CI: 1 workflow ou por package? | ✅ **1 workflow com paths-filter** | Decisão do usuário |

---

## 5. Decisões consolidadas

> Todas as hipóteses foram confirmadas por investigação técnica ou decisão do usuário (2026-08-20). Estas são agora **decisões fechadas** que a fase Specify deve herdar.

### D1: 1 MTA unificado — `suzano-fup-s4` v2.0.0

- **Decisão**: Consolidar 3 MTAs em 1 único na raiz do monorepo. MTA ID `suzano-fup-s4`, versão `2.0.0` (major bump indicando breaking change de consolidação).
- **Racional**: 3 MTAs separados para um sistema coeso gera fricção de deploy. Um MTA unificado permite `yarn bd` único.
- **Confirmado por**: Decisão do usuário + Seção 2.1 (E1) + Seção 2.5 (M1).

### D2: Estrutura `packages/frontend/<app>/` + `packages/backend/<service>/` + `packages/backend/db/`

- **Decisão**: Seguir padrão IAtização. Frontends em `packages/frontend/{fup-panel,manual-fups,role-management,dynamic-forms}/`. Backend em `packages/backend/{application-service,processing-service,public-service,db,app-router,public-app-router}/`.
- **Sem yarn workspaces** — cada package mantém `yarn.lock` próprio (igual cat2, a referência canônica).
- **Confirmado por**: Decisão do usuário (db/ path) + Investigação (cat2 não usa workspaces) + Seção 2.9 (P4).

### D3: Padronizar stack — TS 5.9.x + UI5 1.120.39 + @ui5/cli v3

- **Decisão**: TypeScript `^5.9.x`, SAPUI5 `1.120.39`, `@ui5/cli ^3.11.13` em todos os frontends. Backend mantém TS 5.9.2 + CAP 9.3.1.
- **Ações necessárias**:
  - fup-panel/manual-fups: upgrade TS 5.1→5.9.x
  - dynamic-forms: downgrade @ui5/cli v4→v3
  - role-management: downgrade UI5 1.130.11→1.120.39 + refatorar Clean Arch
- **Confirmado por**: Investigação (3 de 4 apps + cat2 usam v3; 1.120.39 é minUI5Version comum) + Seção 2.2 (S1, S2, S3).

### D4: Manter 2 app-routers + 2 xs-security.json

- **Decisão**: 1 app-router interno (XSUAA, roteia application+processing) + 1 app-router público (sem auth, serve dynamic-forms). 2 xs-security.json: backend (scopes JOBSCHEDULER + uaa.user) + frontend panel (uaa.user). Público sem xs-security.json.
- **Racional**: Requisitos de segurança fundamentalmente diferentes. Unificar adiciona risco desnecessário.
- **Confirmado por**: Decisão do usuário + Seção 2.6 (A1) + Seção 2.7 (I2).

### D5: role-management refatorado para Clean Architecture

- **Decisão**: Refatorar role-management de flat FE template para Clean Architecture 5 camadas (domain/application/infrastructure/presentation/main), alinhado aos demais módulos.
- **Confirmado por**: Decisão do usuário + Seção 2.3 (C2).

### D6: Padronizar printWidth 180 + Prettier wired em todos

- **Decisão**: printWidth 180 em todos os módulos (frontends + backend). Prettier wired no eslint.config.mjs de todos. Corrigir dynamic-forms (Prettier instalado mas não wired).
- **Racional**: 180 é o valor da maioria + cat2 standard. Backend precisa mudar de 120→180.
- **Confirmado por**: Seção 2.2 (S9, S10) + cat2 code-style standard.

### D7: CI com 1 workflow + paths-filter + NUMENDS/git-flow-action

- **Decisão**: 1 workflow CI com `paths-filter` (só roda lint/typecheck/test do package que mudou). Git flow action padronizada em `NUMENDS/git-flow-action`.
- **Confirmado por**: Decisão do usuário (CI) + Investigação (todos repos sob NUMENDS) + Seção 2.1 (E7).

### D8: Testes como requisito P2 (não P1 MVP)

- **Decisão**: P1 = scaffold + migração preservando código. P2 = adicionar testes unitários (vitest) em todos os módulos. P1 inclui apenas smoke test (validar que build:cf funciona).
- **Confirmado por**: Seção 2.8 (T1 — testes inexistentes).

---

## 6. Fontes consultadas

### 6.1 Codebase atual — `suzano-fup-panel-frontend-s4`

- `package.json:1-16` — root orquestrador (name `fup-panel-frontend-s4`, scripts bd/build/deploy, sem engines)
- `mta.yaml:1-5` — MTA ID `fup-panel-frontend-s4`, v1.0.0, deploy_mode html5-repo
- `xs-security.json` — xsappname `fup-panel-frontend-s4`, scope uaa.user, role Token_Exchange
- `fup-panel/package.json` — TS 5.1.6, @ui5/cli 3.11.13, vitest 2.1.9, @sweet-monads/either 3.3.1
- `fup-panel/webapp/` — Clean Arch 5 camadas (domain/application/infrastructure/presentation/main)
- `fup-panel/eslint.config.mjs` — flat config, 4 spaces, printWidth 180, no-explicit-any: off
- `fup-panel/tests/unit/use-cases/functions/find-user-form-layout.test.ts` — único teste (7 linhas stub)
- `manual-fups/package.json` — name duplicado `fup-panel-frontend-s4`, sem testes
- `role-management/` — flat FE template, UI5 1.130.11, sem Clean Arch
- `.github/workflows/fup-panel.yaml` — CI Node 20.x, yarn, lint (sem typecheck/test)
- `.github/workflows/git-flow.yaml:14` — VAEES/git-flow-action

### 6.2 Codebase atual — `suzano-fup-public-frontend-s4`

- `package.json:1-16` — root orquestrador (name `fup-panel-public-frontend-s4`, sem engines)
- `mta.yaml:1-2` — MTA ID `fup-public-frontend-s4`, v1.0.2
- `app-router/package.json` — @sap/approuter 20.8.6, engines.node ^20||^22
- `app-router/xs-app.json` — authenticationMethod none, welcomeFile /fupdynamicformss4
- `dynamic-forms/package.json` — TS 5.9.3, @ui5/cli 4.0.38, vitest 4.0.16, @sweet-monads/either 3.3.1
- `dynamic-forms/webapp/` — Clean Arch 5 camadas
- `dynamic-forms/eslint.config.mjs` — flat config, printWidth 180, no-explicit-any: off, prettier não wired
- `dynamic-forms/domain/errors/forbiden-request.ts` — typo "Forbiden"
- Sem xs-security.json (app público, sem auth)
- `.github/workflows/git-flow.yaml` — VAEES/git-flow-action, sem CI build/lint/test

### 6.3 Codebase atual — `suzano-fup-panel-backend-s4`

- `package.json:1-23` — root orquestrador (scripts bd/build:cf/deploy, mbt + npm-run-all + rimraf)
- `mta.yaml:1-5` — MTA ID `fup-backend-s4`, v1.0.8, schema 3.3.0
- `xs-security.json` — xsappname `fup-backend-s4`, scopes JOBSCHEDULER + uaa.user
- `application-service/package.json` — @sap/cds 9.3.1, TS 5.9.2, vitest 3.1.3, @sweet-monads/either 3.3.1, engines.node ^22
- `application-service/src/` — Clean Arch 5 camadas + extras (extractors, hidrators, controllers)
- `application-service/src/main/adapters/router.ts:5-33` — adaptRoute bridge BaseController ↔ CAP
- `application-service/eslint.config.mjs` — flat config, printWidth 120, no-explicit-any: off
- `processing-service/package.json` — sem vitest/test scripts (apenas lint)
- `public-service/package.json` — vitest scripts presentes
- `db/models/index.cds:1-62` — 62 entidades CDS
- `db/src/hdbgrants/` — 4 cross-schema grants (eva-cap, eva-legacy, s4-replication, s4hana-remote)
- `db/src/synonyms/` — synonyms para tabelas SAP ECC (EKPO, EKKO, MARC, etc.)
- `app-router/xs-app.json` — 2 rotas (application-api, processing-api), ambas xsuaa auth
- `.cdsrc.json` — destinations: GTW_PU, SUZANO_AZURE_AD, SUZANO_FUP_DELLAVOLPE_INTEGRATION_API
- `.github/workflows/application-service.yaml` — CI Node 22, lint, test:coverage; SonarQube comentado

### 6.4 Codebase modelo — `suzano-cat2-frontend-s4`

- `AGENTS.md:1-78` — índice geral IA (78 linhas, padrão IAtização)
- `package.json:1-20` — root orquestrador, engines.node ^22||^24
- `mta.yaml:1-99` — MTA ID cat2-frontend-s4, path packages/frontend/<app>/
- `packages/frontend/registration-of-the-work-time-sheet/AGENTS.md:1-151` — regras específicas do app (151 linhas)
- `.nvmrc` — Node 22
- `renovate.json` — baseBranches quality, patch-only
- `.github/workflows/cat2-frontend.yml` — CI Node 22, lint + typecheck + test:unit
- `.github/workflows/git-flow.yaml:16` — NUMENDS/git-flow-action
- `docs/standards/` — cap-clean-architecture + code-style + react-clean-architecture
- `docs/specs/project/STATE.md` — memória persistente (decisões, blockers, lessons)
- `docs/specs/project/AGENTS-SPEC.md` — guia canônico de como escrever AGENTS.md

### 6.5 Codebase modelo — `numen-software-house`

- `AGENTS.md:1-295` — índice geral IA (295 linhas, padrão IAtização Node)
- `package.json:1-25` — workspaces, engines.node >=24, packageManager yarn@1.22.22
- `.github/workflows/quality-gate.yml` — reusable workflow (lint + typecheck + test)
- `.github/workflows/ci.yml` — CI com Numen AI Reviewer
- `.github/workflows/git-flow.yml:16` — VAEES/git-flow-action

### 6.6 Documentação interna

- `/home/hiagoperoni/Projects/Suzano/S4/docs/FUNCIONAL-FUP.md:1-269` — doc funcional completa (1264 linhas): visão geral, arquitetura, modelo de dados, autenticação, integrações
- `/home/hiagoperoni/Projects/Suzano/S4/suzano-fup-s4/docs/obsidian/Obsidian/IAtizacao/Prompt - Scaffold Monorepo.md` — scaffold prompt com 16 templates de arquivos
- `/home/hiagoperoni/Projects/Suzano/S4/suzano-fup-s4/docs/standards/cap-clean-architecture/README.md` — standard canônico Clean Architecture CAP
- `/home/hiagoperoni/Projects/Suzano/S4/suzano-fup-s4/docs/standards/code-style/typescript.md` — standard code style TypeScript

### 6.7 Web

- Não foi necessária consulta web — toda a informação necessária estava disponível nos codebases e docs internas.

---

## 7. Handoff TLC Spec-Driven

**Próximo passo sugerido**: invocar a skill `tlc-spec-driven` → fase **Specify** carregando este `research.md` como contexto base.

**Slug da feature**: `fup-s4-migration` (deve casar com o slug deste research).

### 7.1 Requirements candidatos

> Todas as gray areas foram resolvidas (Seção 4 + Seção 5). Os requirements abaixo refletem as decisões fechadas.

| ID provisório | Prioridade | User story (1 linha) | Decisão/Seção de apoio |
|---|---|---|---|
| REQ-01 | P1 ⭐ MVP | Como dev, quero scaffold do monorepo `suzano-fup-s4` com AGENTS.md, package.json, mta.yaml, .nvmrc, .github/ | Seção 2.9 (P2), scaffold prompt |
| REQ-02 | P1 ⭐ MVP | Como dev, quero migrar fup-panel para `packages/frontend/fup-panel/` preservando webapp/ e configs | Seção 2.1 (E1), Seção 2.3 (C1) |
| REQ-03 | P1 ⭐ MVP | Como dev, quero migrar manual-fups para `packages/frontend/manual-fups/` | Seção 2.1 (E1) |
| REQ-04 | P1 ⭐ MVP | Como dev, quero migrar role-management para `packages/frontend/role-management/` | Seção 2.1 (E1) |
| REQ-05 | P1 ⭐ MVP | Como dev, quero migrar dynamic-forms para `packages/frontend/dynamic-forms/` | Seção 2.1 (E1) |
| REQ-06 | P1 ⭐ MVP | Como dev, quero migrar application-service para `packages/backend/application-service/` | Seção 2.1 (E3) |
| REQ-07 | P1 ⭐ MVP | Como dev, quero migrar processing-service para `packages/backend/processing-service/` | Seção 2.1 (E3) |
| REQ-08 | P1 ⭐ MVP | Como dev, quero migrar public-service para `packages/backend/public-service/` | Seção 2.1 (E3) |
| REQ-09 | P1 ⭐ MVP | Como dev, quero migrar db/ para `packages/backend/db/` preservando hdbgrants + synonyms | D2, Seção 2.7 (I3, I6) |
| REQ-10 | P1 ⭐ MVP | Como dev, quero MTA unificado (`suzano-fup-s4` v2.0.0) na raiz com todos os módulos + recursos CF | D1, Seção 2.5 (M1, M5) |
| REQ-11 | P1 ⭐ MVP | Como dev, quero AGENTS.md em hierarquia (raiz + packages/frontend + packages/backend) | Seção 2.1 (E4), Seção 2.9 (P4) |
| REQ-12 | P1 ⭐ MVP | Como dev, quero .nvmrc (Node 22) + renovate.json + .gitignore na raiz | Seção 2.4 (CF8, CF9) |
| REQ-13 | P1 ⭐ MVP | Como dev, quero manter 2 app-routers (interno XSUAA + público sem auth) no MTA | D4, Seção 2.5 (M8) |
| REQ-14 | P1 ⭐ MVP | Como dev, quero manter 2 xs-security.json (backend + frontend panel) no MTA | D4, Seção 2.6 (A1) |
| REQ-15 | P1 ⭐ MVP | Como dev, quero CI com 1 workflow + paths-filter + NUMENDS/git-flow-action | D7, Seção 2.1 (E7) |
| REQ-16 | P2 | Como dev, quero padronizar TS 5.9.x + UI5 1.120.39 + @ui5/cli v3 em todos os frontends | D3, Seção 2.2 (S1, S2, S3) |
| REQ-17 | P2 | Como dev, quero CI com lint + typecheck + test (paths-filter) em todos os packages | D7, Seção 2.8 (T4) |
| REQ-18 | P2 | Como dev, quero printWidth 180 + Prettier wired em todos os módulos | D6, Seção 2.2 (S9, S10) |
| REQ-19 | P2 | Como dev, quero corrigir typo "Forbiden" → "Forbidden" em dynamic-forms | Seção 3.2 |
| REQ-20 | P2 | Como dev, quero refatorar role-management para Clean Architecture 5 camadas | D5, Seção 2.3 (C2) |
| REQ-21 | P2 | Como dev, quero corrigir package name duplicado em manual-fups | Seção 3.2 |
| REQ-22 | P3 | Como dev, quero adicionar testes unitários (vitest) em todos os módulos | D8, Seção 2.8 (T1) |
| REQ-23 | P3 | Como dev, quero corrigir SQL injection no backend (usar bound parameters) | Seção 3.2 |
| REQ-24 | P3 | Como dev, quero ativar SonarQube no CI do backend | Seção 3.2 |
| REQ-25 | P3 | Como dev, quero configurar geração automática de @cds-models do backend para frontends | D2 (G8), Seção 2.7 (I4) |

**Notas**:
- IDs são provisórios; a Specify renumera no padrão da feature.
- P1 = scaffold + migração preservando código + MTA unificado + CI. P2 = padronização de stack + refatoração role-management. P3 = débitos técnicos.

### 7.2 Gray areas — TODAS RESOLVIDAS

> Todas as 12 gray areas foram resolvidas em 2026-08-20 por investigação técnica ou decisão do usuário. A fase Specify herda estas decisões — não precisa re-discutir.

| # | Tema | Decisão | Como |
|---|---|---|---|
| GA-01 | App-router | ✅ Manter 2 separados | Decisão do usuário → D4 |
| GA-02 | xs-security | ✅ Manter 2 (interno + público sem auth) | Segue de GA-01 → D4 |
| GA-03 | UI5 versão | ✅ UI5 1.120.39 | Investigação (minUI5Version comum) → D3 |
| GA-04 | @ui5/cli | ✅ v3 (^3.11.13) | Investigação (3/4 apps + cat2 usam v3) → D3 |
| GA-05 | role-management | ✅ Refatorar para Clean Arch | Decisão do usuário → D5 |
| GA-06 | MTA | ✅ 1 MTA unificado v2.0.0 | Decisão do usuário → D1 |
| GA-07 | Workspaces | ✅ NÃO usar workspaces | Investigação (cat2 não usa) → D2 |
| GA-08 | db/ path | ✅ `packages/backend/db/` | Decisão do usuário → D2 |
| GA-09 | MTA versão | ✅ 2.0.0 | Decisão do usuário → D1 |
| GA-10 | Git flow org | ✅ NUMENDS | Investigação (todos repos sob NUMENDS) → D7 |
| GA-11 | CI strategy | ✅ 1 workflow com paths-filter | Decisão do usuário → D7 |
| GA-12 | @cds-models | ✅ Gerar do backend (P1: estático, P2: automático) | Investigação → D2 (G8) |

### 7.3 ADRs candidatos (registrar retroativamente)

> Com as decisões fechadas, os ADRs abaixo devem ser registrados via skill `create-adr` **antes ou durante** a Specify.

- **ADR-001 — Clean Architecture 5 camadas como padrão**: já implementado em 6 dos 7 módulos; role-management será refatorado (D5). Registrar.
- **ADR-002 — @sweet-monads/either como Either monad canônico**: já usado em todos os módulos. Registrar.
- **ADR-003 — Factory pattern (DI manual) sem container**: já implementado em todos. Registrar.
- **ADR-004 — Monorepo consolidado FUP S4**: consolidação de 3 repos em 1 monorepo `suzano-fup-s4` (D1, D2). Registrar.
- **ADR-005 — Stack padronizada TS 5.9.x + UI5 1.120.39 + @ui5/cli v3**: padronização de versões (D3). Registrar.
- **ADR-006 — 2 app-routers + 2 xs-security (interno vs público)**: manter separação de auth (D4). Registrar.
- **ADR-007 — Sem yarn workspaces (cada package com yarn.lock próprio)**: seguir padrão cat2 (D2). Registrar.

### 7.4 Bibliografia mínima que a Specify deve carregar

A fase Specify deve ler (além do template padrão de `spec.md`):

- ⭐ `docs/researches/fup-s4-migration/research.md` (este documento).
- `docs/researches/fup-s4-migration/prompts/fup-s4-migration.md` (briefing verbatim).
- `docs/obsidian/Obsidian/IAtizacao/Prompt - Scaffold Monorepo.md` (scaffold prompt com 16 templates).
- `/home/hiagoperoni/Projects/Suzano/S4/docs/FUNCIONAL-FUP.md` (doc funcional — 1264 linhas, modelo de dados, integrações).
- `docs/standards/cap-clean-architecture/README.md` (standard canônico).
- `docs/standards/code-style/typescript.md` (code style).
- `suzano-cat2-frontend-s4/AGENTS.md` (referência de AGENTS.md raiz).
- `suzano-cat2-frontend-s4/packages/frontend/registration-of-the-work-time-sheet/AGENTS.md` (referência de AGENTS.md de package).

### 7.5 Decisões fechadas (resumo executivo)

| ID | Decisão | Detalhe |
|---|---|---|
| D1 | 1 MTA unificado | `suzano-fup-s4` v2.0.0 |
| D2 | Estrutura packages/ | `packages/frontend/<app>/` + `packages/backend/<service>/` + `packages/backend/db/`. Sem workspaces |
| D3 | Stack padronizada | TS 5.9.x + UI5 1.120.39 + @ui5/cli v3 |
| D4 | 2 app-routers + 2 xs-security | Interno (XSUAA) + público (sem auth) |
| D5 | role-management refatorado | Clean Arch 5 camadas |
| D6 | printWidth 180 + Prettier | Em todos os módulos |
| D7 | CI + git-flow | 1 workflow paths-filter + NUMENDS/git-flow-action |
| D8 | Testes como P2 | P1 = smoke (build:cf); P2 = unit tests |

### 7.6 Sinais de que a Specify deve voltar para a pesquisa

- Surgir uma dimensão nova não mapeada (ex.: requirement de observabilidade/monitoring não discutido).
- Detectada contradição entre dois achados do research.
- Usuário pedir para considerar uma fonte/tecnologia/repositório ainda não consultado.
- Mais de 30% dos requirements candidatos forem descartados — sinal de escopo mal calibrado.

---

**Fontes consultadas neste documento:** 3 codebases FUP (fup-panel-frontend, fup-public-frontend, fup-panel-backend) + 2 codebases modelo (suzano-cat2-frontend-s4, numen-software-house) + doc funcional FUP (1264 linhas) + scaffold prompt IAtização + standards cap-clean-arch/code-style + investigação de git remotes + investigação de versões UI5/@ui5/cli + investigação de workspaces/@cds-models + decisão do usuário (6 perguntas).
