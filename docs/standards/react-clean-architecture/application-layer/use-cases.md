# Application layer — Use Cases

Implementações concretas dos contratos declarados em `domain/use-cases/`. Cada classe implementa exatamente uma interface de domain, recebe suas dependências via constructor e retorna `Either<AbstractError, T>`.

## Estrutura

```
src/application/use-cases/
└── <feature>/
    └── <kebab-name>-use-case.ts
```

| Caminho no domain | Caminho correspondente na application |
|---|---|
| `domain/use-cases/customers/load-customers.ts` | `application/use-cases/customers/load-customers-use-case.ts` |
| `domain/use-cases/sales-orders/load-sales-orders.ts` | `application/use-cases/sales-orders/load-sales-orders-use-case.ts` |
| `domain/use-cases/sales-orders/clone-sales-order.ts` | `application/use-cases/sales-orders/clone-sales-order-use-case.ts` |

## Padrão de implementação

```typescript
// src/application/use-cases/customers/load-customers-use-case.ts

import { left, right } from '@sweet-monads/either';

import type { CustomerRepository } from '@/domain/repositories/customer-repository.js';
import type { LoadCustomers } from '@/domain/use-cases/customers/load-customers.js';
import { AbstractError } from '@/domain/errors/abstract.js';
import { ServerError } from '@/domain/errors/server-error.js';

export class LoadCustomersUseCase implements LoadCustomers {
    constructor(private readonly repository: CustomerRepository) {}

    public async execute(): Promise<LoadCustomers.Result> {
        try {
            return right(await this.repository.findAll());
        } catch (error) {
            if (error instanceof AbstractError) {
                return left(error);
            }
            const err = error as Error;
            return left(new ServerError(err.message));
        }
    }
}
```

## Exemplos

### Exemplo 1 — Use case com parâmetro

```typescript
// src/application/use-cases/sales-orders/load-sales-order-by-id-use-case.ts

import { left, right } from '@sweet-monads/either';

import type { SalesOrderRepository } from '@/domain/repositories/sales-order-repository.js';
import type { LoadSalesOrderById } from '@/domain/use-cases/sales-orders/load-sales-order-by-id.js';
import { AbstractError } from '@/domain/errors/abstract.js';
import { ServerError } from '@/domain/errors/server-error.js';

export class LoadSalesOrderByIdUseCase implements LoadSalesOrderById {
    constructor(private readonly repository: SalesOrderRepository) {}

    public async execute(id: string): Promise<LoadSalesOrderById.Result> {
        try {
            return right(await this.repository.findById(id));
        } catch (error) {
            if (error instanceof AbstractError) {
                return left(error);
            }
            const err = error as Error;
            return left(new ServerError(err.message));
        }
    }
}
```

### Exemplo 2 — Use case de ação sem retorno de dados

```typescript
// src/application/use-cases/sales-orders/clone-sales-order-use-case.ts

import { left, right } from '@sweet-monads/either';

import type { SalesOrderRepository } from '@/domain/repositories/sales-order-repository.js';
import type { CloneSalesOrder } from '@/domain/use-cases/sales-orders/clone-sales-order.js';
import { AbstractError } from '@/domain/errors/abstract.js';
import { ServerError } from '@/domain/errors/server-error.js';

export class CloneSalesOrderUseCase implements CloneSalesOrder {
    constructor(private readonly repository: SalesOrderRepository) {}

    public async execute(id: string): Promise<CloneSalesOrder.Result> {
        try {
            await this.repository.clone(id);
            return right(true);
        } catch (error) {
            if (error instanceof AbstractError) {
                return left(error);
            }
            const err = error as Error;
            return left(new ServerError(err.message));
        }
    }
}
```

### Exemplo 3 — Use case com múltiplos repositórios

```typescript
// src/application/use-cases/sales-orders/create-sales-order-use-case.ts

import { left, right } from '@sweet-monads/either';

import type { CustomerRepository } from '@/domain/repositories/customer-repository.js';
import type { SalesOrderRepository } from '@/domain/repositories/sales-order-repository.js';
import type { CreateSalesOrder } from '@/domain/use-cases/sales-orders/create-sales-order.js';
import { AbstractError } from '@/domain/errors/abstract.js';
import { NotFoundError } from '@/domain/errors/not-found-error.js';
import { ServerError } from '@/domain/errors/server-error.js';

export class CreateSalesOrderUseCase implements CreateSalesOrder {
    constructor(
        private readonly salesOrderRepository: SalesOrderRepository,
        private readonly customerRepository: CustomerRepository
    ) {}

    public async execute(params: CreateSalesOrder.Params): Promise<CreateSalesOrder.Result> {
        try {
            const customer = await this.customerRepository.findById(params.customerId);
            if (!customer) {
                return left(new NotFoundError('Cliente não encontrado'));
            }

            const order = await this.salesOrderRepository.create(params);
            return right(order);
        } catch (error) {
            if (error instanceof AbstractError) {
                return left(error);
            }
            const err = error as Error;
            return left(new ServerError(err.message));
        }
    }
}
```

## Como o `Either` funciona na prática

```
try {
    return right(data);   // ← sucesso: data fica em result.value quando result.isRight()
} catch (error) {
    return left(new XxxError()); // ← falha: erro fica em result.value quando result.isLeft()
}
```

O controller chama `result.isLeft()` ou `result.fold()` para desfazer o Either e atualizar o estado React.

## Anti-padrões

❌ **Fazer fetch diretamente no use case:**
```typescript
async execute(): Promise<LoadCustomers.Result> {
    const res = await fetch('/api/customers'); // ← deve passar pelo repositório
    return right(await res.json());
}
```

❌ **Lançar `throw` fora do try/catch:**
```typescript
async execute(id: string): Promise<...> {
    if (!id) throw new BadRequestError('id obrigatório'); // ← retorne left()
    // ...
}
```

✅ **Correto:**
```typescript
public async execute(id: string): Promise<...> {
    if (!id) {
        return left(new BadRequestError('id obrigatório'));
    }
    try {
        return right(await this.repository.findById(id));
    } catch (error) {
        if (error instanceof AbstractError) {
            return left(error);
        }
        const err = error as Error;
        return left(new ServerError(err.message));
    }
}
```

❌ **Declarar tipos auxiliares no arquivo:**
```typescript
type OrderData = { id: string; total: number }; // ← pertence ao domain
export class LoadOrderUseCase { ... }
```

## Regras de ouro

1. Um arquivo por use case, nome espelhando o contrato do domain + sufixo `-use-case`.
2. `execute` é o único método público — mesmo nome e assinatura da interface.
3. Todo caminho feliz retorna `right(value)`, toda falha retorna `left(new XxxError(...))`.
4. O `catch` captura exceções lançadas pela infra (que usa `throw`) e as converte em `left()`.
5. A classe **nunca** importa de `infra/`, `presentation/`, `main/` — só de `domain/` e `@sweet-monads/either`.
