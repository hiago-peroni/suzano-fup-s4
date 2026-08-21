# Domain layer — Repositories

Repositories definem os **contratos de acesso a dados** que o domínio precisa. A interface declara o que pode ser feito (findAll, findById, save…); a implementação concreta fica em `infra/repositories/`. Nenhuma SQL, fetch ou lógica de rede vive aqui.

## Estrutura

```
src/domain/repositories/
└── <entidade>-repository.ts    → interface XxxRepository + namespace
```

Um arquivo por entidade principal da aplicação.

## Convenções de nomenclatura

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo | `kebab-case.ts` (entidade no singular) | `customer-repository.ts` |
| Interface | `PascalCase` + `Repository` | `CustomerRepository` |
| Namespace colateral | Mesmo nome da interface | `namespace CustomerRepository { FindAll }` |
| Tipos de retorno | Dentro do namespace | `CustomerRepository.FindAll`, `SalesOrderRepository.FindById` |

## Padrão de interface + namespace

O namespace colateral concentra todos os tipos auxiliares do repositório — evita poluir o escopo global e mantém os tipos coesos com o contrato que os usa.

```typescript
// src/domain/repositories/customer-repository.ts

import { CustomerModel } from '@/domain/models/customer.js';

export interface CustomerRepository {
    findAll(): Promise<CustomerRepository.FindAll>;
}

export namespace CustomerRepository {
    export type FindAll = CustomerModel[];
}
```

## Exemplos

### Exemplo 1 — Repository com múltiplos métodos

```typescript
// src/domain/repositories/sales-order-repository.ts

import { SalesOrderHeaderModel } from '@/domain/models/sales-order-header.js';

export interface SalesOrderRepository {
    findAll(): Promise<SalesOrderRepository.FindAll>;
    findById(id: string): Promise<SalesOrderRepository.FindById>;
    clone(id: string): Promise<void>;
}

export namespace SalesOrderRepository {
    export type FindAll = SalesOrderHeaderModel[];
    export type FindById = SalesOrderHeaderModel;
}
```

### Exemplo 2 — Implementação correspondente na infra (referência)

```typescript
// src/infra/repositories/sales-order-repository-impl.ts

import { SalesOrderHeaderModel, SalesOrderHeaderResponse } from '@/domain/models/sales-order-header.js';
import type { SalesOrderRepository } from '@/domain/repositories/sales-order-repository.js';
import type { HttpClient } from '@/domain/protocols/http-client.js';

export class SalesOrderRepositoryImpl implements SalesOrderRepository {
    private readonly BASE_URL = '/sales-order/SalesOrderHeaders';

    constructor(private readonly httpClient: HttpClient) {}

    public async findAll(): Promise<SalesOrderRepository.FindAll> {
        const response = await this.httpClient.request<{ value: SalesOrderHeaderResponse[] }>({
            url: `${this.BASE_URL}?$expand=customer,items($expand=product),status`,
            method: 'GET'
        });
        return SalesOrderHeaderModel.withFromResponseList(response.body.value);
    }

    public async findById(id: string): Promise<SalesOrderRepository.FindById> {
        const response = await this.httpClient.request<SalesOrderHeaderResponse>({
            url: `${this.BASE_URL}(${encodeURIComponent(id)})?$expand=customer,items($expand=product),status`,
            method: 'GET'
        });
        return SalesOrderHeaderModel.withFromResponse(response.body);
    }

    public async clone(id: string): Promise<void> {
        await this.httpClient.request({
            url: `${this.BASE_URL}(${encodeURIComponent(id)})/cloneSalesOrder`,
            method: 'POST'
        });
    }
}
```

### Exemplo 3 — Factory que conecta o repositório ao use case (referência)

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

## Arquivos permitidos / proibidos

| Permitido | Proibido |
|---|---|
| `interface XxxRepository` com métodos tipados | Implementação (classe, async, fetch) |
| `namespace XxxRepository { ... }` com tipos de retorno | Imports de `infra/`, `react`, `zustand` |
| Imports de `domain/models/` | Tipos de resposta da API (ficam em `models/`) |

## Anti-padrões

❌ **Implementação dentro da interface:**
```typescript
export interface CustomerRepository {
    async findAll(): Promise<CustomerModel[]> { // ← proibido
        return fetch('/api/customers').then(r => r.json());
    }
}
```

❌ **Tipo de retorno usando `any` ou tipo genérico sem model:**
```typescript
export interface CustomerRepository {
    findAll(): Promise<any>; // ← sem tipagem de domínio
}
```

❌ **Tipos soltos fora do namespace:**
```typescript
export type CustomerList = CustomerModel[]; // ← deve estar no namespace
export interface CustomerRepository {
    findAll(): Promise<CustomerList>;
}
```

## Regras de ouro

1. Um arquivo de repository por entidade principal — não criar um repositório "genérico".
2. Todos os tipos de retorno vivem no `namespace XxxRepository { ... }` do mesmo arquivo.
3. A interface **nunca** importa de `infra/`, `application/`, `presentation/` ou `main/`.
4. A implementação concreta (`XxxRepositoryImpl`) mora em `src/infra/repositories/` e é injetada via factory na `main/`.
5. `findById` retorna o model diretamente — se o registro não existir, a infra lança `NotFoundError` que propaga via `Either`.

> **Semântica de `findById`**
>
> No contexto React, a infra lança `NotFoundError` diretamente quando o recurso não é encontrado, em vez de retornar `null`. Isso simplifica o use case: ele não precisa verificar `null` e decidir se é um erro — a infra já garante que, se executou sem lançar exceção, o recurso existe.
>
> Diferença com o CAP: no CAP, `findById` retorna `null` e o use case decide se ausência é um erro. Para React, o repositório opera via HTTP, onde um 404 já é uma falha explícita do servidor — faz sentido propagá-la como `NotFoundError` imediatamente.
