# Presentation layer

A presentation layer é a **fronteira entre a lógica de negócio e a interface do usuário**. Ela contém tudo que é React: componentes, páginas, layouts, hooks e controllers. Não contém regras de negócio — apenas orquestra a apresentação dos dados e traduz interações do usuário em chamadas aos use cases.

> **Regra de isolamento:** a presentation layer importa de `domain/` (tipos, modelos, erros) e de `infra/` (stores, i18n). **Não importa** diretamente de `application/` — o acesso aos use cases acontece via controllers injetados por `main/factories/`.

## Estrutura canônica

```
src/presentation/
├── components/         → componentes reutilizáveis (dumb/presentational)
│   ├── <categoria>/
│   │   └── <nome>.tsx
│   └── index.ts        → barrel export de todos os componentes
├── controllers/        → orquestração de use cases e estado de apresentação
│   ├── base/
│   │   ├── protocols.ts        → BaseController + BaseControllerState
│   │   ├── implementation.ts   → BaseControllerImpl
│   │   └── index.ts
│   └── <feature>/
│       └── <feature>-controller.ts
├── hooks/              → custom hooks que conectam controller ao React state
│   └── <feature>/
│       └── use-<feature>.ts
├── layouts/            → wrappers de layout de página inteira
│   └── app-shell/
│       └── app-shell.tsx
└── pages/              → componentes de rota (nível de página)
    └── <feature>/
        ├── components/         → componentes locais da página
        └── <feature>-page.tsx
```

## Responsabilidades por subpasta

| Subpasta | Responsabilidade |
|---|---|
| `controllers/` | Orquestra use cases, gerencia estado de apresentação, expõe métodos tipados |
| `hooks/` | Conecta controller ao React state (`useState`, `useCallback`, `useEffect`) |
| `components/` | Componentes reutilizáveis e puramente visuais — recebem dados via props |
| `pages/` | Componentes de rota — compõem hooks e components para uma tela completa |
| `layouts/` | Wrappers de layout globais (sidebar, header, outlet) |

## Fluxo de dados

```
main/factories/
    ↓ injeta controller via props
pages/<feature>-page.tsx
    ↓ passa controller para hook
hooks/use-<feature>.ts
    ↓ chama métodos do controller
controllers/<feature>-controller.ts
    ↓ executa use cases e retorna Either
application/use-cases/...
    ↓ resultado volta como estado React
hooks/use-<feature>.ts
    ↓ expõe { data, isLoading, error } para a página
pages/<feature>-page.tsx
    ↓ renderiza components com os dados
components/...
```

## Regras de ouro

1. **Nenhuma chamada direta a use cases ou repositórios** — a página recebe o controller via props e delega ao hook.
2. **Controllers são injetados via props** — nunca instanciados dentro de componentes.
3. **Hooks gerenciam estado local de feature** — não estado global (esse fica em stores na infra).
4. **Components são puramente visuais** — recebem apenas props, sem lógica de negócio.
5. **Barrel exports em `components/index.ts`** — centraliza imports dos componentes reutilizáveis.
6. **Sem imports de `application/`** diretamente — sempre via controller.

## Naming

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo de componente | `kebab-case.tsx` | `sales-order-table.tsx` |
| Arquivo de hook | `use-kebab-case.ts` | `use-sales-orders.ts` |
| Arquivo de controller | `kebab-case-controller.ts` | `sales-orders-controller.ts` |
| Arquivo de página | `kebab-case-page.tsx` | `sales-orders-page.tsx` |
| Componente (função) | `PascalCase` | `SalesOrderTable`, `SalesOrdersPage` |
| Hook | `use` + `PascalCase` | `useSalesOrders` |
| Controller (classe) | `PascalCase` + `Controller` | `SalesOrdersController` |
| Factory de página | `PascalCase` + `Factory` | `SalesOrdersPageFactory` |

## Documentos desta seção

- [controllers.md](./controllers.md) — `BaseControllerImpl`, injeção de use cases, `handleResult()`
- [hooks.md](./hooks.md) — custom hooks, `useState + useCallback + useEffect`, assinatura `use*(controller)`
- [components.md](./components.md) — componentes reutilizáveis, barrel exports, organização por categoria
- [pages.md](./pages.md) — componentes de rota, recebem controller via props, composição de hooks + components
- [layouts.md](./layouts.md) — wrappers de layout, `AppShell`, `Outlet`
