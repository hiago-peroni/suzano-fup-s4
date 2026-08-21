# Roadmap

**Current Milestone:** M1 — Backend Refactor
**Status:** In Progress

---

## M1 — Backend Refactor (Scaffold + Critical Fixes)

**Goal:** Monorepo scaffolded + backend com zero violations critical/high de cap-clean-architecture
**Target:** 14 épicos Jira (RFIA-3 a RFIA-16) completos

### Features

**fup-backend-refactor** - IN PROGRESS

- EP-01: Scaffold do monorepo + migração packages/backend/ (RFIA-3)
- EP-04: Corrigir SQL injection — 8 sites (RFIA-6)
- EP-07: Padronizar error taxonomy — bug recursão + dead code + ConflictError (RFIA-9)
- EP-02: Eliminar framework do domain — 28 arquivos (RFIA-4)
- EP-12: Mover business logic de public.ts (RFIA-14)
- EP-09: Rename persistence→repositories + entities→entity-events (RFIA-11)
- EP-05: Eliminar pastas proibidas (RFIA-7)
- EP-06: Eliminar barrel exports — 53 files (RFIA-8)
- EP-10: Controllers execute→handle — 34 files (RFIA-12)
- EP-08: Padronizar tsconfig + eslint (RFIA-10)

**Ordem de execução:**

```
EP-01 → EP-04 → EP-07 → EP-02 → EP-12 → EP-09 → EP-05 → EP-06 → EP-10 → EP-08
```

---

## M2 — Backend Polish (P2/P3)

**Goal:** Backend com models ricos, código compartilhado, db/ canonicalizado, CI completo

### Features

**fup-backend-polish** - PLANNED

- EP-03: Models anêmicos → ricos (RFIA-5) — exige mover validação de use cases
- EP-11: Extrair código compartilhado shared/ (RFIA-13) — reconciliar 4 arquivos drifted
- EP-13: db/ canonicalização (RFIA-15) — rename legacy-eva-models → replication
- EP-14: Ativar SonarQube + CI gate (RFIA-16) — paths-filter + checkout@v4

---

## M3 — Frontend Refactor

**Goal:** Migrar 4 apps frontend (fup-panel, manual-fups, role-management, dynamic-forms) para `packages/frontend/`

### Features

**fup-frontend-refactor** - PLANNED

- Deep research individual por app frontend (auditoria vs react-clean-architecture standard)
- Padronizar TS 5.9.x + UI5 1.120.39 + @ui5/cli v3
- Refatorar role-management para Clean Architecture 5 camadas
- Mover 2 frontends para packages/frontend/

---

## M4 — MTA Unificado + Deploy

**Goal:** MTA único `suzano-fup-s4` v2.0.0 com frontend + backend, deploy único

### Features

**mta-consolidation** - PLANNED

- Consolidar 3 MTAs em 1
- 2 app-routers (interno XSUAA + público sem auth)
- 2 xs-security.json
- Deploy validado em QAS + PRD

---

## Future Considerations

- @cds-models gerado automaticamente do backend para frontends no monorepo
- Testes unitários completos (vitest) em todos os módulos
- Playwright E2E para fluxos críticos (fornecedor confirmando/rejeitando datas)
- Anexação ao monorepo `suzano-eva-frontend-s4` (consolidação futura)
