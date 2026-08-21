# Main layer — Factories de Controllers

Factories de controller são funções puras de **composição**: recebem o `httpClient`, chamam as factories de use case correspondentes e retornam uma instância pronta do controller. Não há lógica de negócio, condicional ou efeito colateral — apenas instanciação e conexão.

## Estrutura canônica

```
src/main/factories/
└── controllers/
    ├── sales-orders-controller.ts
    ├── customers-controller.ts
    └── products-controller.ts
```

Cada arquivo contém **uma única factory** nomeada `make<Feature>Controller`. O arquivo segue kebab-case com sufixo `-controller.ts` (sem prefixo `make-`).

## Shape canônico — um use case

```typescript
// src/main/factories/controllers/customers-controller.ts

import { FetchHttpClient } from '@/infra/http/fetch-http-client.js';
import { CustomersController } from '@/presentation/controllers/customers/customers-controller.js';
import { makeLoadCustomers } from '@/main/factories/use-cases/load-customers.js';

export const makeCustomersController = (httpClient: FetchHttpClient): CustomersController => {
    const useCase = makeLoadCustomers(httpClient);
    return new CustomersController(useCase);
};
```

O tipo de retorno é sempre a **classe concreta do controller** — nunca `any` ou uma interface genérica.

## Shape com múltiplos use cases

```typescript
// src/main/factories/controllers/sales-orders-controller.ts

import { FetchHttpClient } from '@/infra/http/fetch-http-client.js';
import { SalesOrdersController } from '@/presentation/controllers/sales-orders/sales-orders-controller.js';
import { makeLoadSalesOrders } from '@/main/factories/use-cases/load-sales-orders.js';
import { makeLoadSalesOrderById } from '@/main/factories/use-cases/load-sales-order-by-id.js';
import { makeCloneSalesOrder } from '@/main/factories/use-cases/clone-sales-order.js';

export const makeSalesOrdersController = (httpClient: FetchHttpClient): SalesOrdersController => {
    const loadSalesOrders = makeLoadSalesOrders(httpClient);
    const loadSalesOrderById = makeLoadSalesOrderById(httpClient);
    const cloneSalesOrder = makeCloneSalesOrder(httpClient);
    return new SalesOrdersController(loadSalesOrders, loadSalesOrderById, cloneSalesOrder);
};
```

Cada use case é criado por sua própria factory, e o controller recebe todos via constructor — nunca os instancia diretamente.

## Convenções

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo | `<kebab-feature>-controller.ts` | `sales-orders-controller.ts` |
| Função | `make<PascalFeature>Controller` | `makeSalesOrdersController` |
| Export | named export (`export const`) | `export const makeXxxController = ...` |
| Parâmetro | `httpClient: FetchHttpClient` | sempre nomeado `httpClient` |
| Retorno | tipo explícito da classe concreta | `: SalesOrdersController` |

## Regras de ouro

1. **Named export obrigatório** — nunca `export default`. O nome da função deve ser `make<Feature>Controller` para facilitar busca e rastreabilidade.
2. **O controller nunca cria repositórios** — a responsabilidade de instanciar `XxxRepositoryImpl` pertence às factories de use case. A factory de controller apenas chama `makeXxx(httpClient)`.
3. **Uma factory por controller** — cada arquivo exporta exatamente uma função factory. Não agrupe factories de controllers distintos em um mesmo arquivo.
4. **Sem lógica** — a factory não contém condicionais, cálculos ou chamadas assíncronas. Se precisar de lógica condicional, ela pertence ao use case ou ao controller, não à factory.
5. **`httpClient` é passado, não instanciado** — a factory nunca chama `new FetchHttpClient()` internamente; quem cria o `httpClient` é a page factory.

## Anti-padrões

❌ **Factory instanciando repositório diretamente:**
```typescript
export const makeCustomersController = (httpClient: FetchHttpClient): CustomersController => {
    const repository = new CustomerRepositoryImpl(httpClient); // ← responsabilidade da use case factory
    const useCase = new LoadCustomersUseCase(repository);
    return new CustomersController(useCase);
};
```

❌ **Usar `export default`:**
```typescript
export default (httpClient: FetchHttpClient) => { // ← dificulta busca e refatoração
    return new CustomersController(makeLoadCustomers(httpClient));
};
```

❌ **Lógica condicional na factory:**
```typescript
export const makeSalesOrdersController = (httpClient: FetchHttpClient, role: string) => {
    if (role === 'admin') { // ← lógica de negócio não pertence à factory
        return new AdminSalesOrdersController(makeLoadSalesOrders(httpClient));
    }
    return new SalesOrdersController(makeLoadSalesOrders(httpClient));
};
```
