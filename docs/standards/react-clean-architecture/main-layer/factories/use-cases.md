# Main layer — Factories de Use Cases

Factories de use case são funções puras responsáveis por **instanciar o repositório concreto e injetá-lo no use case**. Recebem o `httpClient` como parâmetro, constroem o grafo de dependências mínimo necessário e devolvem a instância pronta para uso. Não contêm lógica de negócio.

## Estrutura

```
src/main/factories/use-cases/
└── <kebab-name>.ts   → make<UseCase>(httpClient)
```

O arquivo da factory usa kebab-case **sem sufixo** (`load-customers.ts`), espelhando o nome do use case correspondente em `application/use-cases/` (que mantém o sufixo `-use-case.ts`).

Exemplos:

```
src/main/factories/use-cases/
├── load-customers.ts
├── load-sales-orders.ts
├── load-sales-order-by-id.ts
└── clone-sales-order.ts
```

## Exemplo 1 — Factory simples (um repositório)

```typescript
// src/main/factories/use-cases/load-customers.ts

import { FetchHttpClient } from '@/infra/http/fetch-http-client.js';
import { CustomerRepositoryImpl } from '@/infra/repositories/customer-repository-impl.js';
import { LoadCustomersUseCase } from '@/application/use-cases/customers/load-customers-use-case.js';
import type { LoadCustomers } from '@/domain/use-cases/customers/load-customers.js';

export const makeLoadCustomers = (httpClient: FetchHttpClient): LoadCustomers => {
    const repository = new CustomerRepositoryImpl(httpClient);
    return new LoadCustomersUseCase(repository);
};
```

## Exemplo 2 — Factory com múltiplos repositórios

Quando o use case depende de mais de um repositório, cada um é instanciado separadamente e injetado na ordem esperada pelo constructor.

```typescript
// src/main/factories/use-cases/create-sales-order.ts

import { FetchHttpClient } from '@/infra/http/fetch-http-client.js';
import { CustomerRepositoryImpl } from '@/infra/repositories/customer-repository-impl.js';
import { SalesOrderRepositoryImpl } from '@/infra/repositories/sales-order-repository-impl.js';
import { CreateSalesOrderUseCase } from '@/application/use-cases/sales-orders/create-sales-order-use-case.js';
import type { CreateSalesOrder } from '@/domain/use-cases/sales-orders/create-sales-order.js';

export const makeCreateSalesOrder = (httpClient: FetchHttpClient): CreateSalesOrder => {
    const salesOrderRepository = new SalesOrderRepositoryImpl(httpClient);
    const customerRepository = new CustomerRepositoryImpl(httpClient);
    return new CreateSalesOrderUseCase(salesOrderRepository, customerRepository);
};
```

## Regras de ouro

1. A factory recebe `httpClient: FetchHttpClient` como parâmetro — nunca o instancia internamente.
2. Somente `export const makeXxx` — **sem `export default`**.
3. Uma factory por use case — cada arquivo instancia exatamente um use case.
4. A factory **não contém lógica de negócio** — apenas instancia repositórios e injeta dependências.
5. O arquivo segue kebab-case **sem sufixo** (`load-customers.ts`) e a função é `make<UseCase>` (`makeLoadCustomers`), espelhando o nome do use case.

## Anti-padrões

❌ **Instanciar o `httpClient` dentro da factory:**
```typescript
export const makeLoadCustomers = (): LoadCustomers => {
    const httpClient = new FetchHttpClient(); // ← quem cria o httpClient é a page factory
    const repository = new CustomerRepositoryImpl(httpClient);
    return new LoadCustomersUseCase(repository);
};
```

❌ **Adicionar lógica de negócio na factory:**
```typescript
export const makeLoadCustomers = (httpClient: FetchHttpClient, user: User): LoadCustomers => {
    if (!user.isActive) {
        throw new Error('Usuário inativo'); // ← lógica de negócio não vai na factory
    }
    const repository = new CustomerRepositoryImpl(httpClient);
    return new LoadCustomersUseCase(repository);
};
```

❌ **Usar `export default`:**
```typescript
export default (httpClient: FetchHttpClient) => { // ← sempre named export
    const repository = new CustomerRepositoryImpl(httpClient);
    return new LoadCustomersUseCase(repository);
};
```
