# AGENTS.md — packages/backend

Regras específicas do backend do FUP S4. **Leia o [`AGENTS.md`](../../AGENTS.md) raiz primeiro** — este arquivo complementa com regras do backend.

---

## 1. Overview

Backend CAP do FUP com 3 serviços (application, processing, public) + db (HANA HDI) + 2 app-routers. Refatoração para conformidade total com `cap-clean-architecture`.

## 2. Serviços

| Serviço | CDS Service | OData Path | Auth |
|---------|------------|------------|------|
| application-service | PanelService | `/fup/panel` | XSUAA |
| processing-service | ProcessingService | `/fup-panel-processing` | XSUAA |
| public-service | PublicService | `/fup/public` | Nenhuma (referer) |

## 3. Stack

- **Linguagem:** TypeScript `^5.9.x` strict
- **Framework:** SAP CAP/CDS `^9.3.1`
- **Database:** SAP HANA Cloud (HDI Container, schema `FUP_BACKEND_DB_S4`)
- **Either monad:** `@sweet-monads/either ^3.3.1`
- **Test runner:** Vitest `3.1.3`
- **Lint:** ESLint `^9.35.0` (flat config) + Prettier `^3.3.3` (printWidth 180)
- **Imports:** NodeNext (`.js` obrigatório)

## 4. Clean Architecture (5 camadas)

```
src/
├── domain/          ← contratos, models ricos, errors (sem framework)
├── application/     ← use cases, services auxiliares
├── infrastructure/  ← repositories, adapters, hydrators, UoW
├── presentation/    ← controllers (handle(), não execute())
└── main/            ← factories (DI manual), routes, config, scripts
```

**Dependency rule absoluta:** setas só apontam para `domain/`. Inverter = violação.

## 5. Comandos por serviço

Cada serviço em `packages/backend/{service}/` tem seus próprios scripts:

| Script | Descrição |
|--------|-----------|
| `yarn dev` | Server de desenvolvimento |
| `yarn lint` | ESLint |
| `yarn ts-typecheck` | TypeScript type check |
| `yarn test:unit` | Testes unitários (Vitest) |
| `yarn build:cf` | Build para Cloud Foundry |

## 6. Convenções específicas

- **Error taxonomy:** 6 classes canônicas (400/401/403/404/409/500) + `UnavailableServiceError` (503). Proibido: `BadGatewayError`, `UnprocessableEntity`.
- **Models:** sempre ricos (`static with(props)` + `validate()`), nunca anêmicos.
- **Controllers:** método `handle()`, nunca `execute()`.
- **Use cases:** retornam `Promise<Either<AbstractError, T>>`. Infrastructure propaga exceção.
- **Sem framework no domain:** única dependência permitida é `@sweet-monads/either`.
- **Sem barrel exports:** exceto `domain/errors/index.ts`.
- **console.error apenas:** stdout reservado para CAP/JSON-RPC.

## 7. Zonas de Proteção (Read-Only)

- `docs/standards/cap-clean-architecture/` — standard canônico. NUNCA alterar.
- `docs/researches/fup-backend-refactor/` — pesquisa que embasa a refatoração. NUNCA alterar.
- `docs/specs/` — specs e tasks. Trate como Read-Only.

## 8. Referências

- `docs/standards/cap-clean-architecture/README.md` — Clean Architecture canônica
- `docs/researches/fup-backend-refactor/research.md` — pesquisa com 148 violações auditadas
- `docs/specs/features/fup-backend-refactor/` — spec + tasks (17 tasks, T1-T17)
- `docs/obsidian/Obsidian/suzano-s4/Auditoria Backend.md` — auditoria completa
