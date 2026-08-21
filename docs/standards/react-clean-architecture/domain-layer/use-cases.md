# Domain layer — Use Cases

Use cases no domain são **contratos de operações de negócio**. Cada interface declara o que pode ser executado; a implementação concreta fica em `application/use-cases/`. O tipo de retorno é sempre `Either<AbstractError, T>` — nunca `throw`, nunca `Promise<T>` sem o wrapper.

## Estrutura

```
src/domain/use-cases/
└── <feature>/
    └── <kebab-name>.ts    → interface + namespace
```

Os arquivos são agrupados por feature (ex.: `customers/`, `sales-orders/`), não por tipo de operação como no CAP.

## Convenções de nomenclatura

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo | `kebab-case.ts` | `load-customers.ts`, `clone-sales-order.ts` |
| Interface | Verbo + entidade, sem sufixo | `LoadCustomers`, `CloneSalesOrder`, `LoadSalesOrderById` |
| Namespace colateral | Mesmo nome da interface | `namespace LoadCustomers { Result }` |
| Tipo de resultado | Dentro do namespace | `LoadCustomers.Result` |
| Método da interface | Sempre `execute` | `execute(params?): Promise<XxxUseCase.Result>` |

## Padrão de interface + namespace

```typescript
// src/domain/use-cases/customers/load-customers.ts

import type { Either } from '@sweet-monads/either';
import type { AbstractError } from '@/domain/errors/abstract.js';
import type { CustomerModel } from '@/domain/models/customer.js';

export interface LoadCustomers {
    execute(): Promise<LoadCustomers.Result>;
}

export namespace LoadCustomers {
    export type Result = Either<AbstractError, CustomerModel[]>;
}
```

## Exemplos

### Exemplo 1 — Use case sem parâmetros

```typescript
// src/domain/use-cases/sales-orders/load-sales-orders.ts

import type { Either } from '@sweet-monads/either';
import type { AbstractError } from '@/domain/errors/abstract.js';
import type { SalesOrderHeaderModel } from '@/domain/models/sales-order-header.js';

export interface LoadSalesOrders {
    execute(): Promise<LoadSalesOrders.Result>;
}

export namespace LoadSalesOrders {
    export type Result = Either<AbstractError, SalesOrderHeaderModel[]>;
}
```

### Exemplo 2 — Use case com parâmetros

```typescript
// src/domain/use-cases/sales-orders/load-sales-order-by-id.ts

import type { Either } from '@sweet-monads/either';
import type { AbstractError } from '@/domain/errors/abstract.js';
import type { SalesOrderHeaderModel } from '@/domain/models/sales-order-header.js';

export interface LoadSalesOrderById {
    execute(id: string): Promise<LoadSalesOrderById.Result>;
}

export namespace LoadSalesOrderById {
    export type Result = Either<AbstractError, SalesOrderHeaderModel>;
}
```

### Exemplo 3 — Use case com resultado `boolean` (ação sem retorno de dados)

```typescript
// src/domain/use-cases/sales-orders/clone-sales-order.ts

import type { Either } from '@sweet-monads/either';
import type { AbstractError } from '@/domain/errors/abstract.js';

export interface CloneSalesOrder {
    execute(id: string): Promise<CloneSalesOrder.Result>;
}

export namespace CloneSalesOrder {
    export type Result = Either<AbstractError, boolean>;
}
```

### Exemplo 4 — Implementação correspondente na application layer (referência)

```typescript
// src/application/use-cases/customers/load-customers-use-case.ts

import { left, right } from '@sweet-monads/either';

import type { CustomerRepository } from '@/domain/repositories/customer-repository.js';
import type { LoadCustomers } from '@/domain/use-cases/customers/load-customers.js';
import { ServerError } from '@/domain/errors/server-error.js';

export class LoadCustomersUseCase implements LoadCustomers {
    constructor(private readonly repository: CustomerRepository) {}

    public async execute(): Promise<LoadCustomers.Result> {
        try {
            return right(await this.repository.findAll());
        } catch (error) {
            const err = error as Error;
            return left(new ServerError(err.message));
        }
    }
}
```

## Arquivos permitidos / proibidos

| Permitido | Proibido |
|---|---|
| `interface + namespace` no mesmo arquivo | Classe de implementação |
| `Either<AbstractError, T>` como `Result` | Imports de `infra/`, `react`, `application/` |
| `execute(params?)` como único método | Múltiplos métodos públicos na interface |
| Imports de `domain/models/` e `domain/errors/` | Tipos soltos fora do namespace |

## Anti-padrões

❌ **Múltiplos métodos públicos:**
```typescript
export interface SalesOrderOperations {
    load(): Promise<...>;
    loadById(id: string): Promise<...>;
    clone(id: string): Promise<...>;
}
// ← cada operação deve ser um use case separado
```

❌ **Retorno sem `Either`:**
```typescript
export interface LoadCustomers {
    execute(): Promise<CustomerModel[]>; // ← sem tratamento de erro tipado
}
```

❌ **Parâmetros como objeto quando há apenas um campo:**
```typescript
export interface LoadSalesOrderById {
    execute(params: { id: string }): Promise<...>; // ← prefer execute(id: string)
}
```

❌ **Tipos auxiliares fora do namespace:**
```typescript
export type LoadResult = Either<AbstractError, CustomerModel[]>; // ← deve estar no namespace

export interface LoadCustomers {
    execute(): Promise<LoadResult>;
}
```

## Regras de ouro

1. **Um arquivo por operação** — nomeado com o verbo de ação (`load-customers.ts`, `clone-sales-order.ts`).
2. **`execute` é o único método público** da interface.
3. **`Result` é sempre `Either<AbstractError, T>`** — nunca `Promise<T>` sem o wrapper.
4. **Tipos auxiliares vivem no namespace** — sem `export type` solto no mesmo arquivo.
5. **A interface não conhece a implementação** — quem une os dois é `main/factories/`.
