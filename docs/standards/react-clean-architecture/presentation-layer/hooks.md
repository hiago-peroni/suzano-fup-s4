# Presentation layer — Hooks

Custom hooks **conectam o controller ao estado React**. Recebem o controller como parâmetro, chamam seus métodos e expõem o estado da feature (`data`, `isLoading`, `error`) e as ações disponíveis. São o único lugar onde `useState`, `useCallback` e `useEffect` são usados para estado de feature.

## Estrutura

```
src/presentation/hooks/
└── <feature>/
    └── use-<feature>.ts
```

## Padrão de hook

```typescript
// src/presentation/hooks/sales-orders/use-sales-orders.ts

import { useState, useCallback, useEffect } from 'react';

import type { AbstractError } from '@/domain/errors/abstract.js';
import type { SalesOrderHeaderModel } from '@/domain/models/sales-order-header.js';
import type { SalesOrdersController } from '@/presentation/controllers/sales-orders/sales-orders-controller.js';

type State = {
    salesOrders: SalesOrderHeaderModel[];
    isLoading: boolean;
    error: AbstractError | null;
}

export const useSalesOrders = (controller: SalesOrdersController) => {
    const [state, setState] = useState<State>({
        salesOrders: [],
        isLoading: true,
        error: null
    });

    const load = useCallback(async () => {
        setState((prev) => ({ ...prev, isLoading: true, error: null }));
        const result = await controller.load();
        result.fold(
            (error) => setState({ salesOrders: [], isLoading: false, error }),
            (salesOrders) => setState({ salesOrders, isLoading: false, error: null })
        );
    }, [controller]);

    const clone = useCallback(async (id: string) => {
        const result = await controller.clone(id);
        if (result.isRight()) {
            await load(); // ← recarrega lista após clonar
        }
        return result;
    }, [controller, load]);

    useEffect(() => { load(); }, [load]);

    return { ...state, reload: load, clone };
};
```

## Exemplo 2 — Hook para detalhe com parâmetro de ID

```typescript
// src/presentation/hooks/sales-orders/use-sales-order-detail.ts

import { useState, useCallback, useEffect } from 'react';

import type { AbstractError } from '@/domain/errors/abstract.js';
import type { SalesOrderHeaderModel } from '@/domain/models/sales-order-header.js';
import type { SalesOrdersController } from '@/presentation/controllers/sales-orders/sales-orders-controller.js';

type State = {
    salesOrder: SalesOrderHeaderModel | null;
    isLoading: boolean;
    error: AbstractError | null;
}

export const useSalesOrderDetail = (controller: SalesOrdersController, id: string) => {
    const [state, setState] = useState<State>({
        salesOrder: null,
        isLoading: true,
        error: null
    });

    const load = useCallback(async () => {
        setState((prev) => ({ ...prev, isLoading: true, error: null }));
        const result = await controller.loadById(id);
        result.fold(
            (error) => setState({ salesOrder: null, isLoading: false, error }),
            (salesOrder) => setState({ salesOrder, isLoading: false, error: null })
        );
    }, [controller, id]);

    useEffect(() => { load(); }, [load]);

    return { ...state, reload: load };
};
```

## Como `result.fold()` funciona

`fold()` é o método do Either que desfaz o wrapper de forma funcional:

```typescript
result.fold(
    (leftValue) => { /* executado se result.isLeft() */ },
    (rightValue) => { /* executado se result.isRight() */ }
);
```

Equivalente usando `isLeft()`:

```typescript
if (result.isLeft()) {
    setState({ error: result.value, isLoading: false });
} else {
    setState({ data: result.value, isLoading: false });
}
```

## Convenções

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo | `use-kebab-case.ts` | `use-sales-orders.ts` |
| Função | `use` + `PascalCase` | `useSalesOrders`, `useSalesOrderDetail` |
| Tipo de estado | `type State = { ... }` (local no arquivo) | — |
| Estado inicial | Sempre com `isLoading: true` e dados vazios | — |
| Retorno | Spread do estado + ações nomeadas | `{ ...state, reload: load, clone }` |

## Padrão de estado

```typescript
type State = {
    <entidade>: XxxModel[] | XxxModel | null;   // dados da feature
    isLoading: boolean;                          // true enquanto busca
    error: AbstractError | null;                 // null quando ok
}
```

## Anti-padrões

❌ **Chamar use case diretamente no hook (bypassar controller):**
```typescript
export const useSalesOrders = (useCase: LoadSalesOrders) => {
    const load = useCallback(() => useCase.execute(), []); // ← deve receber controller
};
```

❌ **Estado global no hook (usar store para isso):**
```typescript
export const useSalesOrders = () => {
    // Sem controller — usando estado global ← erro de arquitetura
    const orders = useSalesOrdersStore((s) => s.orders);
};
```

❌ **Efeito sem dependências corretas:**
```typescript
useEffect(() => {
    load(); // ← load não está nas deps — causa stale closure
}, []);    // ← deve ser [load]
```

❌ **Lógica de negócio dentro do hook:**
```typescript
const processedOrders = useMemo(
    () => orders.filter((o) => o.status === 'PENDING'), // ← filtro de negócio vai no use case
    [orders]
);
```

## Regras de ouro

1. Todo hook recebe **o controller como primeiro parâmetro** — nunca instancia o controller internamente.
2. O estado inicial tem `isLoading: true` — garante que o loader aparece na primeira renderização.
3. `useCallback` com **todas as dependências corretas** — previne chamadas desnecessárias e stale closures.
4. `useEffect(() => { load(); }, [load])` carrega dados na montagem do componente.
5. Ações que modificam dados (clone, delete, create) **recarregam a lista** após sucesso via `await load()`.
6. O hook **não conhece** como o controller foi construído — só chama seus métodos públicos.
