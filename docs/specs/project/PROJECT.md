# Suzano FUP S4

**Vision:** Consolidar 3 repositórios FUP (2 frontends + 1 backend com 3 serviços CAP) em um único monorepo `suzano-fup-s4` no padrão IAtização, refatorando o backend para conformidade total com o standard `cap-clean-architecture`.

**For:** Time de desenvolvimento Suzano S4 (analistas FUP, devs, code reviewers).

**Solves:** Code duplication entre 3 repos, 148 violações de Clean Architecture no backend (32 critical), 8 sites de SQL injection, error taxonomy inconsistente, configs desalinhadas, ausência de AGENTS.md/docs/CI gate.

## Goals

- [ ] Monorepo `suzano-fup-s4` com `packages/backend/` contendo os 3 serviços + db + 2 app-routers
- [ ] Backend 100% conforme `cap-clean-architecture` standard (zero violations critical/high)
- [ ] MTA unificado (`suzano-fup-s4` v2.0.0) com deploy único `yarn bd`
- [ ] AGENTS.md em hierarquia (raiz + packages/backend)
- [ ] CI com paths-filter + lint/typecheck/test gate
- [ ] Funcionalidade de produção preservada (sistema em uso ativo)

## Tech Stack

**Core:**

- Framework: SAP CAP/CDS `^9.3.1` (backend) + SAPUI5 `1.120.39` (frontends, fase posterior)
- Language: TypeScript `^5.9.x` strict
- Database: SAP HANA Cloud (HDI Container, schema `FUP_BACKEND_DB_S4`)
- Runtime: Node.js `22`
- Cloud: SAP BTP Cloud Foundry

**Key dependencies:**

- `@sap/cds ^9.3.1` — CAP framework
- `@sweet-monads/either ^3.3.1` — Either monad (use case return type)
- `@cap-js/hana ^2.2.0` — HANA adapter
- `@cap-js/cds-typer ^0.37.0` — Type generation from CDS
- `vitest 3.1.3` — Test runner
- `eslint ^9.35.0` (flat config) + `prettier ^3.3.3` — Lint/format
- `mbt ^1.2.31` — MTA build tool

## Scope

**v1 includes (backend refactor):**

- Scaffold do monorepo + migração dos 3 serviços para `packages/backend/`
- Eliminar framework do domain (28 arquivos com `@models` imports)
- Corrigir 8 sites de SQL injection
- Padronizar error taxonomy (7 classes canônicas + fix bug recursão infinita)
- Eliminar pastas proibidas (`extractors/`, `formatters/`, `hidrators/`, `i18n/`)
- Eliminar 53 barrel exports
- Rename `persistence/` → `repositories/`, `entities/` → `entity-events/`
- Controllers `execute()` → `handle()`
- Padronizar tsconfig + eslint (strict, printWidth 180, NodeNext)
- Mover business logic de `main/routes/public.ts` para controller/use case
- db/ canonicalização (`legacy-eva-models/` → `replication/`)
- CI gate com paths-filter + SonarQube

**Explicitly out of scope:**

- Frontend refactoring (fase posterior — research individual por app frontend)
- Models anêmicos → ricos (P2, exige mover validação de use cases)
- Extrair código compartilhado `shared/` package (P3)
- Testes unitários completos (P2 — P1 inclui apenas smoke test de build)
- yarn workspaces (decisão D2: sem workspaces)

## Constraints

- Timeline: não especificada (baby steps)
- Technical: código em produção — refatoração deve preservar comportamento; SAP BTP Cloud Foundry; CAP/CDS 9.3.1; HANA Cloud HDI; cross-schema grants (4 schemas externos)
- Resources: 1 dev (Hiago) + IA agents
- Integrações: SAP S/4HANA (via Replication Hub), Monitor Fiscal EVA (cross-db), DellaVolpe (REST API), SAP Gateway (OData), Azure AD, SAP Job Scheduler

## Research base

- `docs/researches/fup-s4-migration/research.md` — decisions macro (D1-D8), estrutura alvo, stack
- `docs/researches/fup-backend-refactor/research.md` — auditoria arquivo a arquivo (148 violations, 14 épicos Jira)

## Referências arquiteturais

- Standard canônico: `docs/standards/cap-clean-architecture/README.md`
- Code style: `docs/standards/code-style/typescript.md`
- Projeto de referência: `suzano-cat2-frontend-s4/` (monorepo padrão IAtização)
- Scaffold prompt: `docs/obsidian/Obsidian/IAtizacao/Prompt - Scaffold Monorepo.md`
