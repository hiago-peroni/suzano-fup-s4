# Main layer

A main layer é a **raiz de composição (composition root)** da aplicação: o único lugar que conhece todas as outras camadas e as conecta via injeção de dependência. Nenhum outro módulo importa da `main/` — só a `main/` importa de todos.

> **Equivalência com o CAP:** a `main/` da arquitetura React tem o mesmo papel da `main/` do backend — factories instanciam o grafo de dependências, rotas conectam URLs a componentes.

## Estrutura

```
src/main/
├── factories/
│   ├── controllers/
│   │   └── <feature>-controller.ts          → make<Feature>Controller()
│   ├── pages/
│   │   └── <feature>-page-factory.tsx        → <FeaturePageFactory /> (componente React)
│   └── use-cases/
│       └── <kebab-name>.ts                   → make<UseCase>()
├── app.tsx                                   → App component + BrowserRouter + rotas
└── index.tsx                                 → ReactDOM.createRoot + providers globais
```

## Responsabilidades

| Subpasta/arquivo | Responsabilidade |
|---|---|
| `factories/use-cases/` | Cria instâncias de use cases com suas dependências |
| `factories/controllers/` | Cria instâncias de controllers com use cases compostos |
| `factories/pages/` | Componentes React que montam o controller e passam para a página |
| `app.tsx` | Define o `BrowserRouter`, `Routes` e `Route` com factories como elementos |
| `index.tsx` | Entry point: `ReactDOM.createRoot`, providers globais (`ThemeAppProvider`) |

## Regras de ouro

1. **Nenhuma regra de negócio.** A main apenas instancia e conecta — sem `if` de domínio.
2. **A main conhece todas as camadas** — é o único arquivo que pode importar de `domain/`, `application/`, `infra/` e `presentation/` ao mesmo tempo.
3. **Nada importa da main** — o fluxo de dependências é unidirecional.
4. **Factories são funções puras** para use cases e controllers; **componentes React** para pages.
5. **`useMemo` nas page factories** — garante que o controller não seja recriado a cada render.

## Anti-padrões

- ❌ Importar de `@/domain/` ou `@/application/` direto nas páginas — páginas recebem tudo via props do factory
- ❌ Lógica de negócio nas factories — factories só instanciam e injetam, sem `if`, sem transformações
- ❌ Instanciar o mesmo repositório em múltiplas factories sem compartilhar — usar uma única instância por composição quando o repositório é stateless
- ❌ Exportar `default` de factories — usar named exports (`export const makeXxx = ...`)
- ❌ Criar factories para componentes reutilizáveis (buttons, inputs) — factories são apenas para pages que precisam de injeção de dependência

## Naming

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo de use case factory | `kebab-case.ts` | `load-customers.ts` |
| Arquivo de controller factory | `kebab-case-controller.ts` | `sales-orders-controller.ts` |
| Arquivo de page factory | `kebab-case-page-factory.tsx` | `sales-orders-page-factory.tsx` |
| Função factory | `make` + `PascalCase` | `makeLoadCustomers`, `makeSalesOrdersController` |
| Componente de page factory | `PascalCase` + `PageFactory` | `SalesOrdersPageFactory` |

## Documentos desta seção

- [factories/README.md](./factories/README.md) — padrão `make*`, cadeia de dependências, `useMemo`
- [routes.md](./routes.md) — definição de rotas, `BrowserRouter`, `app.tsx`, `index.tsx`
