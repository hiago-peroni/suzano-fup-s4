# Infra layer — Repositories

Implementações concretas das interfaces `XxxRepository` declaradas em `domain/repositories/`. Cada implementação recebe um `HttpClient`, faz as requisições necessárias e transforma as respostas da API em domain models.

## Estrutura

```
src/infra/repositories/
└── <entidade>-repository-impl.ts    → XxxRepositoryImpl implements XxxRepository
```

| Contrato no domain | Implementação na infra |
|---|---|
| `domain/repositories/customer-repository.ts` | `infra/repositories/customer-repository-impl.ts` |
| `domain/repositories/sales-order-repository.ts` | `infra/repositories/sales-order-repository-impl.ts` |

## Padrão de implementação

```typescript
// src/infra/repositories/customer-repository-impl.ts

import type { HttpClient } from '@/domain/protocols/http-client.js';
import type { CustomerRepository } from '@/domain/repositories/customer-repository.js';
import { CustomerModel, type CustomerResponse } from '@/domain/models/customer.js';

export class CustomerRepositoryImpl implements CustomerRepository {
    private readonly BASE_URL = '/sales-order/Customers';

    constructor(private readonly httpClient: HttpClient) {}

    public async findAll(): Promise<CustomerRepository.FindAll> {
        const response = await this.httpClient.request<{ value: CustomerResponse[] }>({
            url: this.BASE_URL,
            method: 'GET'
        });
        return CustomerModel.withFromResponseList(response.body.value);
    }
}
```

## Exemplos

### Exemplo 1 — Repositório com múltiplos métodos e OData expand

```typescript
// src/infra/repositories/sales-order-repository-impl.ts

import type { HttpClient } from '@/domain/protocols/http-client.js';
import type { SalesOrderRepository } from '@/domain/repositories/sales-order-repository.js';
import { SalesOrderHeaderModel, type SalesOrderHeaderResponse } from '@/domain/models/sales-order-header.js';

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

### Exemplo 2 — Repositório com body em POST

```typescript
public async create(params: SalesOrderRepository.CreateParams): Promise<SalesOrderHeaderModel> {
    const response = await this.httpClient.request<SalesOrderHeaderResponse>({
        url: this.BASE_URL,
        method: 'POST',
        body: {
            customerID: params.customerId,
            items: params.items.map((i) => ({ productID: i.productId, quantity: i.quantity }))
        }
    });
    return SalesOrderHeaderModel.withFromResponse(response.body);
}
```

## Convenções

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo | `kebab-case` + `-repository-impl.ts` | `customer-repository-impl.ts` |
| Classe | `PascalCase` + `RepositoryImpl` | `CustomerRepositoryImpl` |
| URL base | `private readonly BASE_URL` | `'/sales-order/Customers'` |
| Constructor | `private readonly httpClient: HttpClient` | Tipado pela interface, nunca pela impl |

## Arquivos permitidos / proibidos

| Permitido | Proibido |
|---|---|
| `class XxxRepositoryImpl implements XxxRepository` | Regras de negócio (validações, orquestração) |
| `private readonly BASE_URL` | Imports de `application/`, `presentation/` |
| Mapeamento `Response → Model` via `withFromResponse()`/`withFromResponseList()` | `Either` — infra usa `throw` |
| `httpClient.request(...)` para I/O | `fetch()` direto |

## Anti-padrões

❌ **Regra de negócio dentro do repositório:**
```typescript
public async findAll(): Promise<SalesOrderHeaderModel[]> {
    const orders = await this.loadAll();
    return orders.filter((o) => o.status === 'PENDING'); // ← filtro de negócio vai no use case
}
```

❌ **Retornar `Either` do repositório:**
```typescript
public async findAll(): Promise<Either<AbstractError, CustomerModel[]>> {
    // ← repositório usa throw; Either é papel da application layer
}
```

❌ **Instanciar `FetchHttpClient` dentro do repositório:**
```typescript
constructor() {
    this.httpClient = new FetchHttpClient(); // ← deve ser injetado via constructor
}
```

## Regras de ouro

1. Um arquivo por entidade — nome espelha o contrato do domain + sufixo `Impl`.
2. O `HttpClient` é **sempre injetado** — nunca instanciado dentro do repositório.
3. Toda transformação `Response → Model` usa os factory methods do model (`withFromResponse()`, `withFromResponseList()`).
4. Erros são **lançados** (`throw`) — a application layer é responsável por convertê-los em `left()`.
5. Sem regras de negócio — o repositório só executa I/O e mapeia formatos.
6. **Todo valor dinâmico interpolado na URL** (ex.: `id` vindo de `useParams`) passa por `encodeURIComponent` — caracteres como `)`, `?` e `&` reescreveriam a URL OData.
