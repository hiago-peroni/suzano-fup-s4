# Domain layer — Models

Models são as **entidades de negócio da aplicação**. Cada `XxxModel` encapsula os dados de uma entidade, expõe getters tipados e concentra o mapeamento entre o formato da resposta da API e o formato que o restante da aplicação usa. Nenhum I/O ocorre dentro de um model.

## Estrutura

```
src/domain/models/
└── <entidade>.ts    → XxxProps + XxxResponse + class XxxModel
```

Um arquivo de model contém três partes obrigatórias:

1. **`XxxResponse`** — tipo que espelha o formato bruto retornado pela API
2. **`XxxProps`** — tipo interno do model (campos já transformados/normalizados)
3. **`class XxxModel`** — classe com propriedades privadas, getters e factory methods

## Factory canônica `with` + transformadores `withFrom*`

Alinhado ao [CAP Clean Architecture](../../cap-clean-architecture/domain-layer/models/README.md), o model tem **uma única porta de construção pública**: `static with(props)`, que recebe props já normalizadas e é a única que chama o constructor privado. Toda construção a partir de outra origem (resposta de API, etc.) usa o prefixo **`withFrom*`**: a variante normaliza a origem e delega para `with()`.

> O prefixo `withFrom*` mantém `with` como porta única — apenas adiciona **transformações** explícitas para cada fonte. Continua sendo "construir o model **com** algo", agora com adjetivo de origem.

## Convenções de nomenclatura

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo | `kebab-case.ts` | `customer.ts`, `sales-order-header.ts` |
| Tipo de resposta | `PascalCase` + `Response` | `CustomerResponse`, `SalesOrderHeaderResponse` |
| Tipo de props | `PascalCase` + `Props` | `CustomerProps`, `SalesOrderHeaderProps` |
| Classe | `PascalCase` + `Model` | `CustomerModel`, `SalesOrderHeaderModel` |
| Factory canônica (props) | `static with(props)` | `CustomerModel.with({ id, name })` |
| Factory de item (API) | `static withFromResponse(item)` | `CustomerModel.withFromResponse(responseItem)` |
| Factory de lista (API) | `static withFromResponseList(list)` | `CustomerModel.withFromResponseList(response.value)` |

## Arquivos permitidos / proibidos

| Permitido | Proibido |
|---|---|
| `XxxProps`, `XxxResponse`, `class XxxModel` | Funções fora da classe |
| Getters tipados | `async` em qualquer método |
| `static with()`, `withFromResponse()`, `withFromResponseList()` | Factories `from()`/`fromList()` (porta única é `with`) |
| Enums e tipos auxiliares da entidade | Imports de `react`, `infra/`, `application/` |
| `toObject()` para serialização simples | Lógica de I/O (fetch, localStorage), chamadas a repositórios |

## Exemplos

### Exemplo 1 — Model simples

```typescript
// src/domain/models/customer.ts

export type CustomerResponse = {
    ID: string;
    name: string;
    email?: string;
}

export type CustomerProps = {
    id: string;
    name: string;
    email: string;
}

export class CustomerModel {
    private constructor(private readonly props: CustomerProps) {}

    public static with(props: CustomerProps): CustomerModel {
        return new CustomerModel(props);
    }

    public static withFromResponse(response: CustomerResponse): CustomerModel {
        return CustomerModel.with({
            id: response.ID,
            name: response.name,
            email: response.email ?? ''
        });
    }

    public static withFromResponseList(list: CustomerResponse[]): CustomerModel[] {
        return list.map(CustomerModel.withFromResponse);
    }

    public get id(): string {
        return this.props.id;
    }

    public get name(): string {
        return this.props.name;
    }

    public get email(): string {
        return this.props.email;
    }
}
```

### Exemplo 2 — Model com enum, campos opcionais e normalização

```typescript
// src/domain/models/sales-order-header.ts

export type SalesOrderStatus = 'COMPLETED' | 'PENDING' | 'REJECTED';

export type SalesOrderHeaderResponse = {
    ID: string;
    totalAmount?: number;
    status_id?: string;
    status?: { id: string };
    customer?: { ID: string; name: string };
    items?: unknown[];
    createdAt?: string;
}

export type SalesOrderHeaderProps = {
    id: string;
    totalAmount: number;
    status: SalesOrderStatus;
    customerId: string;
    customerName: string;
    itemCount: number;
    createdAt: string;
}

export class SalesOrderHeaderModel {
    private constructor(private readonly props: SalesOrderHeaderProps) {}

    public static with(props: SalesOrderHeaderProps): SalesOrderHeaderModel {
        return new SalesOrderHeaderModel(props);
    }

    public static withFromResponse(response: SalesOrderHeaderResponse): SalesOrderHeaderModel {
        const statusId = response.status_id ?? response.status?.id ?? 'PENDING';
        return SalesOrderHeaderModel.with({
            id: response.ID,
            totalAmount: response.totalAmount ?? 0,
            status: statusId as SalesOrderStatus,
            customerId: response.customer?.ID ?? '',
            customerName: response.customer?.name ?? '',
            itemCount: response.items?.length ?? 0,
            createdAt: response.createdAt ?? ''
        });
    }

    public static withFromResponseList(list: SalesOrderHeaderResponse[]): SalesOrderHeaderModel[] {
        return list.map(SalesOrderHeaderModel.withFromResponse);
    }

    public get id(): string {
        return this.props.id;
    }

    public get totalAmount(): number {
        return this.props.totalAmount;
    }

    public get status(): SalesOrderStatus {
        return this.props.status;
    }

    public get customerId(): string {
        return this.props.customerId;
    }

    public get customerName(): string {
        return this.props.customerName;
    }

    public get itemCount(): number {
        return this.props.itemCount;
    }

    public get createdAt(): string {
        return this.props.createdAt;
    }

    public toObject(): SalesOrderHeaderProps {
        return { ...this.props };
    }
}
```

### Exemplo 3 — Uso correto do model no repositório (infra layer)

```typescript
// src/infra/repositories/customer-repository-impl.ts

public async findAll(): Promise<CustomerModel[]> {
    const response = await this.httpClient.request<{ value: CustomerResponse[] }>({
        url: '/api/Customers',
        method: 'GET'
    });
    return CustomerModel.withFromResponseList(response.body.value); // ← factory do domain
}
```

## Anti-padrões

❌ **Model anêmico (type alias sem classe):**
```typescript
// Proibido — sem comportamento, sem encapsulamento
export type CustomerModel = {
    id: string;
    name: string;
}
```

❌ **Factories `from()`/`fromList()` como porta de construção:**
```typescript
// Proibido — porta única é `with`; transformadores usam prefixo `withFrom*`
public static from(response: CustomerResponse): CustomerModel { /* ... */ }
public static fromList(list: CustomerResponse[]): CustomerModel[] { /* ... */ }
```

❌ **Lógica de I/O dentro do model:**
```typescript
static async withFromApi(): Promise<CustomerModel> {
    const res = await fetch('/api/customers'); // ← nunca dentro do domain
    return CustomerModel.withFromResponse(await res.json());
}
```

❌ **Importar infra ou React dentro do model:**
```typescript
import { FetchHttpClient } from '@/infra/http/fetch-http-client.js'; // ← proibido
import { useState } from 'react'; // ← proibido
```

## Regras de ouro

1. `XxxResponse` espelha o contrato da API — pode ter campos opcionais e nomes em formato de API (ex.: `ID`, `status_id`).
2. `XxxProps` é o formato interno normalizado — sempre tipado, sem `undefined`.
3. O constructor é **sempre privado** — a única forma de criar um model é via `with()` ou suas variantes `withFrom*`.
4. `with(props)` é a **porta única** de construção; `withFromResponse()`/`withFromResponseList()` normalizam a resposta da API e delegam para `with()`.
5. Toda lógica de default/fallback (ex.: `totalAmount ?? 0`) fica no `withFromResponse()`, nunca espalhada pelos consumers.
