# Main layer — Factories de Pages

Factories de página são **componentes React** responsáveis por montar o controller e injetá-lo na página correspondente. Por serem componentes, podem usar hooks como `useMemo` para garantir que o controller seja criado uma única vez por montagem — sem precisar de container de DI externo.

## Estrutura

```
src/main/factories/pages/
└── <feature>-page-factory.tsx   → <{Feature}PageFactory /> (componente React)
```

Exemplo com múltiplas features:

```
src/main/factories/pages/
├── sales-orders-page-factory.tsx
├── customers-page-factory.tsx
└── products-page-factory.tsx
```

## Shape canônico

```tsx
// src/main/factories/pages/sales-orders-page-factory.tsx

import { useMemo } from 'react';
import { FetchHttpClient } from '@/infra/http/fetch-http-client.js';
import { makeSalesOrdersController } from '@/main/factories/controllers/sales-orders-controller.js';
import { SalesOrdersPage } from '@/presentation/pages/sales-orders/sales-orders-page.js';

const httpClient = new FetchHttpClient();

export const SalesOrdersPageFactory: React.FC = () => {
    const controller = useMemo(() => makeSalesOrdersController(httpClient), []);
    return <SalesOrdersPage controller={controller} />;
};
```

> **Por que `useMemo`?** O controller é uma classe com estado interno; recriá-lo a cada render perderia o estado e causaria loops de re-fetch. `useMemo` com deps `[]` garante uma única instância por montagem do componente.

## Exemplo 2 — Uso da factory nas rotas

A factory é registrada diretamente como `element` da rota. Ela encapsula toda a composição de dependências, e o roteador não precisa saber nada sobre controllers ou use cases:

```tsx
// src/main/app.tsx (ou routes.tsx)

import { Route, Routes } from 'react-router-dom';
import { ROUTES } from '@/shared/constants/routes.js';
import { SalesOrdersPageFactory } from '@/main/factories/pages/sales-orders-page-factory.js';
import { CustomersPageFactory } from '@/main/factories/pages/customers-page-factory.js';

export const AppRoutes: React.FC = () => {
    return (
        <Routes>
            <Route path={ROUTES.SALES_ORDERS} element={<SalesOrdersPageFactory />} />
            <Route path={ROUTES.CUSTOMERS} element={<CustomersPageFactory />} />
        </Routes>
    );
};
```

## Regras de ouro

1. O nome do componente é **PascalCase** com sufixo `PageFactory` — ex.: `SalesOrdersPageFactory` — porque é um componente React válido e pode ser usado diretamente no JSX.
2. Usar **sempre `useMemo`** com deps `[]` para criar o controller — garante instância única por montagem e evita re-fetch desnecessário.
3. O `FetchHttpClient` é instanciado **fora do componente**, no escopo do módulo, como constante — uma instância por arquivo, reutilizada a cada render.
4. **Uma factory por página** — cada rota tem seu próprio arquivo `<feature>-page-factory.tsx`.
5. A factory **não contém lógica de negócio nem de apresentação** — só instancia o `httpClient`, cria o controller via `makeXxxController` e renderiza a página com o controller injetado.

## Anti-padrões

❌ **Factory de página sem `useMemo`:**
```tsx
export const SalesOrdersPageFactory: React.FC = () => {
    const controller = makeSalesOrdersController(httpClient); // ← recria o controller a cada render
    return <SalesOrdersPage controller={controller} />;
};
```

❌ **Instanciar `FetchHttpClient` dentro do componente:**
```tsx
export const SalesOrdersPageFactory: React.FC = () => {
    const controller = useMemo(() => makeSalesOrdersController(new FetchHttpClient()), []); // ← nova instância a cada montagem
    return <SalesOrdersPage controller={controller} />;
};
```
