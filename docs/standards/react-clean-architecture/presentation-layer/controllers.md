# Presentation layer — Controllers

Controllers de apresentação **orquestram use cases e gerenciam o estado de apresentação**. São o ponto de entrada dos dados para a camada React: recebem use cases via constructor, executam operações e retornam `Either<AbstractError, T>`. Não fazem renderização — quem faz são os hooks e componentes.

> Os controllers de apresentação têm papel diferente dos controllers HTTP do backend CAP: aqui, o controller é uma classe TypeScript (não um handler HTTP) que conecta os use cases ao sistema de estado React.

## Estrutura

```
src/presentation/controllers/
├── base/
│   ├── protocols.ts        → BaseController + BaseControllerState<T>
│   ├── implementation.ts   → BaseControllerImpl (abstrata)
│   └── index.ts            → barrel export
└── <feature>/
    └── <feature>-controller.ts
```

## A classe base — `BaseControllerImpl`

Gerada pelo MCP. Todos os controllers concretos herdam dela:

```typescript
// src/presentation/controllers/base/protocols.ts

import type { Either } from '@sweet-monads/either';
import type { AbstractError } from '@/domain/errors/abstract.js';

export interface BaseController {
    handleResult<T>(result: Either<AbstractError, T>): BaseControllerState<T>;
}

export type BaseControllerState<T> = {
    data: T | null;
    error: AbstractError | null;
}
```

```typescript
// src/presentation/controllers/base/implementation.ts

import type { Either } from '@sweet-monads/either';
import type { AbstractError } from '@/domain/errors/abstract.js';
import type { BaseController, BaseControllerState } from '@/presentation/controllers/base/protocols.js';

export abstract class BaseControllerImpl implements BaseController {
    public handleResult<T>(result: Either<AbstractError, T>): BaseControllerState<T> {
        if (result.isLeft()) {
            return { data: null, error: result.value };
        }
        return { data: result.value, error: null };
    }
}
```

`handleResult` é o helper que converte `Either` em estado React: se `left()`, extrai a mensagem de erro; se `right()`, retorna os dados.

## Padrão de controller concreto

```typescript
// src/presentation/controllers/sales-orders/sales-orders-controller.ts

import type { Either } from '@sweet-monads/either';
import type { AbstractError } from '@/domain/errors/abstract.js';
import type { SalesOrderHeaderModel } from '@/domain/models/sales-order-header.js';
import type { CloneSalesOrder } from '@/domain/use-cases/sales-orders/clone-sales-order.js';
import type { LoadSalesOrderById } from '@/domain/use-cases/sales-orders/load-sales-order-by-id.js';
import type { LoadSalesOrders } from '@/domain/use-cases/sales-orders/load-sales-orders.js';
import { BaseControllerImpl } from '@/presentation/controllers/base/implementation.js';

export class SalesOrdersController extends BaseControllerImpl {
    constructor(
        private readonly loadSalesOrders: LoadSalesOrders,
        private readonly loadSalesOrderById: LoadSalesOrderById,
        private readonly cloneSalesOrder: CloneSalesOrder
    ) {
        super();
    }

    public async load(): Promise<Either<AbstractError, SalesOrderHeaderModel[]>> {
        return this.loadSalesOrders.execute();
    }

    public async loadById(id: string): Promise<Either<AbstractError, SalesOrderHeaderModel>> {
        return this.loadSalesOrderById.execute(id);
    }

    public async clone(id: string): Promise<Either<AbstractError, boolean>> {
        return this.cloneSalesOrder.execute(id);
    }
}
```

## Exemplo 2 — Controller com parâmetros de criação

```typescript
// src/presentation/controllers/products/products-controller.ts

import type { LoadProducts } from '@/domain/use-cases/products/load-products.js';
import type { CreateProduct } from '@/domain/use-cases/products/create-product.js';
import { BaseControllerImpl } from '@/presentation/controllers/base/implementation.js';

export class ProductsController extends BaseControllerImpl {
    constructor(
        private readonly loadProducts: LoadProducts,
        private readonly createProduct: CreateProduct
    ) {
        super();
    }

    public async load() {
        return this.loadProducts.execute();
    }

    public async create(params: CreateProduct.Params) {
        return this.createProduct.execute(params);
    }
}
```

## Convenções

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo | `kebab-case-controller.ts` | `sales-orders-controller.ts` |
| Classe | `PascalCase` + `Controller` | `SalesOrdersController` |
| Métodos | `camelCase` descritivos | `load()`, `loadById(id)`, `clone(id)` |
| Constructor params | `private readonly` + nome do use case | `private readonly loadSalesOrders: LoadSalesOrders` |

## Anti-padrões

❌ **Regra de negócio dentro do controller:**
```typescript
async load() {
    const result = await this.loadSalesOrders.execute();
    // filtrar apenas pedidos do usuário logado aqui ← não, vai no use case
    return result;
}
```

❌ **Controller renderizando JSX:**
```typescript
// Controllers são classes TypeScript — nunca retornam JSX
render() {
    return <div>{...}</div>; // ← proibido
}
```

❌ **Instanciar use cases dentro do controller:**
```typescript
constructor() {
    this.loadSalesOrders = new LoadSalesOrdersUseCase(...); // ← DI via constructor externo
}
```

❌ **Acessar store Zustand dentro do controller:**
```typescript
async load() {
    const { user } = useAuthStore(); // ← hooks só em componentes/hooks React
}
```

## Regras de ouro

1. Controllers **estendem `BaseControllerImpl`** — nunca implementam `BaseController` direto.
2. Use cases são **sempre tipados pelas interfaces do domain** no constructor — nunca pela classe de implementação.
3. Cada método público do controller corresponde a **uma operação do use case** — sem lógica intermediária.
4. Controllers são **classes puras TypeScript** — sem JSX, sem `useState`, sem hooks React.
5. A instanciação do controller acontece em `main/factories/` — não dentro de componentes.
