# Main layer — Factories

Factories são o **mecanismo de injeção de dependência** da arquitetura React. Não usam container de DI — a composição é manual via funções e componentes. Há três tipos: factories de use case, de controller e de página.

## Estrutura

```
src/main/factories/
├── use-cases/
│   └── <kebab-name>.ts            → make<UseCase>(httpClient)
├── controllers/
│   └── <feature>-controller.ts   → make<Feature>Controller(httpClient)
└── pages/
    └── <feature>-page-factory.tsx → <FeaturePageFactory /> (componente React)
```

## Tipo 1 — Factory de use case

Recebe o `httpClient`, cria o repositório concreto e retorna a instância do use case:

```typescript
// src/main/factories/use-cases/load-customers.ts

import { FetchHttpClient } from '@/infra/http/fetch-http-client.js';
import { CustomerRepositoryImpl } from '@/infra/repositories/customer-repository-impl.js';
import { LoadCustomersUseCase } from '@/application/use-cases/customers/load-customers-use-case.js';

export const makeLoadCustomers = (httpClient: FetchHttpClient) => {
    const repository = new CustomerRepositoryImpl(httpClient);
    return new LoadCustomersUseCase(repository);
};
```

```typescript
// src/main/factories/use-cases/load-sales-orders.ts

import { FetchHttpClient } from '@/infra/http/fetch-http-client.js';
import { SalesOrderRepositoryImpl } from '@/infra/repositories/sales-order-repository-impl.js';
import { LoadSalesOrdersUseCase } from '@/application/use-cases/sales-orders/load-sales-orders-use-case.js';

export const makeLoadSalesOrders = (httpClient: FetchHttpClient) => {
    const repository = new SalesOrderRepositoryImpl(httpClient);
    return new LoadSalesOrdersUseCase(repository);
};
```

```typescript
// src/main/factories/use-cases/clone-sales-order.ts

import { FetchHttpClient } from '@/infra/http/fetch-http-client.js';
import { SalesOrderRepositoryImpl } from '@/infra/repositories/sales-order-repository-impl.js';
import { CloneSalesOrderUseCase } from '@/application/use-cases/sales-orders/clone-sales-order-use-case.js';

export const makeCloneSalesOrder = (httpClient: FetchHttpClient) => {
    const repository = new SalesOrderRepositoryImpl(httpClient);
    return new CloneSalesOrderUseCase(repository);
};
```

## Tipo 2 — Factory de controller

Recebe o `httpClient` e compõe os use cases necessários para o controller:

```typescript
// src/main/factories/controllers/sales-orders-controller.ts

import { FetchHttpClient } from '@/infra/http/fetch-http-client.js';
import { SalesOrdersController } from '@/presentation/controllers/sales-orders/sales-orders-controller.js';
import { makeLoadSalesOrders } from '@/main/factories/use-cases/load-sales-orders.js';
import { makeLoadSalesOrderById } from '@/main/factories/use-cases/load-sales-order-by-id.js';
import { makeCloneSalesOrder } from '@/main/factories/use-cases/clone-sales-order.js';

export const makeSalesOrdersController = (httpClient: FetchHttpClient) => {
    return new SalesOrdersController(
        makeLoadSalesOrders(httpClient),
        makeLoadSalesOrderById(httpClient),
        makeCloneSalesOrder(httpClient)
    );
};
```

## Tipo 3 — Factory de página (componente React)

A page factory é um **componente React** (não uma função simples) porque precisa usar `useMemo` para evitar recriar o controller a cada render:

```typescript
// src/main/factories/pages/sales-orders-page-factory.tsx

import { useMemo } from 'react';
import { FetchHttpClient } from '@/infra/http/fetch-http-client.js';
import { makeSalesOrdersController } from '@/main/factories/controllers/sales-orders-controller.js';
import { SalesOrdersPage } from '@/presentation/pages/sales-orders/sales-orders-page.js';

const httpClient = new FetchHttpClient(); // ← instância única por módulo

export const SalesOrdersPageFactory: React.FC = () => {
    const controller = useMemo(() => makeSalesOrdersController(httpClient), []);
    return <SalesOrdersPage controller={controller} />;
};
```

> **Por que `useMemo`?** O controller é uma classe com estado interno; recriá-lo a cada render perderia o estado e causaria loops de re-fetch. `useMemo` com deps `[]` garante uma única instância por montagem do componente.

## Exemplo completo — cadeia de dependências

```
FetchHttpClient
    └── SalesOrderRepositoryImpl(httpClient)
            └── LoadSalesOrdersUseCase(repository)
            └── LoadSalesOrderByIdUseCase(repository)
            └── CloneSalesOrderUseCase(repository)
                    └── SalesOrdersController(loadAll, loadById, clone)
                                └── SalesOrdersPageFactory → <SalesOrdersPage controller={...} />
```

## Quando o repositório é compartilhado entre use cases

Quando múltiplos use cases de um mesmo controller usam o mesmo repositório, criar uma instância por use case é aceitável (é o padrão atual). Para otimização, compartilhar a instância:

```typescript
// src/main/factories/controllers/sales-orders-controller.ts (com repositório compartilhado)

export const makeSalesOrdersController = (httpClient: FetchHttpClient) => {
    const repository = new SalesOrderRepositoryImpl(httpClient); // ← uma instância
    return new SalesOrdersController(
        new LoadSalesOrdersUseCase(repository),
        new LoadSalesOrderByIdUseCase(repository),
        new CloneSalesOrderUseCase(repository)
    );
};
```

## Anti-padrões

❌ **Instanciar controller dentro do componente de página:**
```typescript
export function SalesOrdersPage() {
    const controller = makeSalesOrdersController(new FetchHttpClient()); // ← na factory
}
```

❌ **Page factory sem `useMemo`:**
```typescript
export const SalesOrdersPageFactory: React.FC = () => {
    const controller = makeSalesOrdersController(httpClient); // ← sem useMemo, recria a cada render
    return <SalesOrdersPage controller={controller} />;
};
```

❌ **Lógica de negócio na factory:**
```typescript
export const makeSalesOrdersController = (httpClient, user) => {
    if (user.role === 'admin') { ... } // ← lógica de negócio não vai na factory
};
```

## Regras de ouro

1. Factories de use case recebem `httpClient` como parâmetro — nunca o instanciam internamente.
2. Factories de controller chamam factories de use case — nunca instanciam `XxxRepositoryImpl` diretamente (a responsabilidade fica nas use case factories).
3. Page factories são **componentes React** com `useMemo` — garantem instância única do controller.
4. O `FetchHttpClient` é instanciado **uma vez por módulo** (fora do componente), na constante do arquivo da page factory.
5. Factories **não contêm lógica de negócio** — só instanciam e conectam.
