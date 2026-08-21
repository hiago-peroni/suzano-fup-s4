# Domain layer — Protocols

Protocols são **interfaces técnicas que o domínio precisa** mas cuja implementação pertence à infra. O nome "protocol" diferencia esses contratos dos repositórios (que são contratos de dados) e dos use cases (que são contratos de operações). No projeto React, o único protocol gerado pelo MCP é `HttpClient`.

## Estrutura

```
src/domain/protocols/
└── http-client.ts    → interface HttpClient + namespace com tipos
```

## Convenções

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo | `kebab-case.ts` | `http-client.ts` |
| Interface | `PascalCase` descritivo | `HttpClient` |
| Namespace colateral | Mesmo nome da interface | `namespace HttpClient { Method, Request, Response }` |
| Tipos auxiliares | Dentro do namespace | `HttpClient.Method`, `HttpClient.Request` |

## O protocol gerado pelo MCP — `HttpClient`

```typescript
// src/domain/protocols/http-client.ts

export interface HttpClient {
    request<T>(params: HttpClient.Request): Promise<HttpClient.Response<T>>;
}

export namespace HttpClient {
    export type Method = 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';

    export type Request = {
        url: string;
        method: Method;
        body?: unknown;
        headers?: Record<string, string>;
    }

    export type Response<T = unknown> = {
        statusCode: number;
        body: T;
    }
}
```

O domain define **o que** o HTTP client precisa fazer; a `infra/http/fetch-http-client.ts` define **como**.

## Como o protocol é usado nas outras camadas

```typescript
// domain/repositories/customer-repository.ts — contrato usa interface do domain
export interface CustomerRepository {
    findAll(): Promise<CustomerRepository.FindAll>
}

// infra/repositories/customer-repository-impl.ts — implementação recebe HttpClient via DI
export class CustomerRepositoryImpl implements CustomerRepository {
    constructor(private readonly httpClient: HttpClient) {} // ← tipado pela interface

    public async findAll(): Promise<CustomerModel[]> {
        const response = await this.httpClient.request<{ value: CustomerResponse[] }>({
            url: '/api/Customers',
            method: 'GET'
        });
        return CustomerModel.withFromResponseList(response.body.value);
    }
}

// main/factories/use-cases/load-customers.ts — factory injeta a implementação concreta
export const makeLoadCustomers = (httpClient: FetchHttpClient) => {
    const repository = new CustomerRepositoryImpl(httpClient); // ← impl concreta
    return new LoadCustomersUseCase(repository);
};
```

## Exemplo 2 — Adicionando um novo protocol

Se a aplicação precisar de um segundo protocolo técnico (ex.: storage, analytics), o padrão é o mesmo:

```typescript
// src/domain/protocols/storage-client.ts

export interface StorageClient {
    get(key: string): string | null;
    set(key: string, value: string): void;
    remove(key: string): void;
}
```

A implementação concreta (`LocalStorageClient`) vai em `src/infra/storage/local-storage-client.ts`.

## Anti-padrões

❌ **Colocar a implementação dentro do protocol:**
```typescript
// Proibido — domain não pode ter implementação com fetch
export class HttpClientImpl {
    async request(...) {
        return fetch(...); // ← pertence à infra
    }
}
```

❌ **Importar o protocol direto da infra nos use cases:**
```typescript
// Proibido — use case deve depender da interface, não da implementação
import { FetchHttpClient } from '@/infra/http/fetch-http-client.js';
```

✅ **Correto — use case depende da interface do domain:**
```typescript
import type { HttpClient } from '@/domain/protocols/http-client.js';
```

## Regras de ouro

1. Protocols no domain são **sempre interfaces** — zero implementação.
2. O namespace colateral (`HttpClient.Request`, `HttpClient.Response<T>`) mantém os tipos auxiliares coesos com o contrato.
3. A implementação concreta (`FetchHttpClient`) sempre mora em `infra/` e é injetada via `main/factories/`.
4. `HttpClient` é **o único protocol** gerado pelo MCP — novos protocols devem seguir o mesmo padrão de `interface + namespace`.
