# Infra layer — HTTP

O `FetchHttpClient` é a implementação concreta da interface `HttpClient` do domain. Encapsula a Fetch API, serializa/desserializa JSON e mapeia status HTTP em erros de domínio tipados.

## Estrutura

```
src/infra/http/
└── fetch-http-client.ts    → FetchHttpClient implements HttpClient
```

## Implementação gerada pelo MCP

```typescript
// src/infra/http/fetch-http-client.ts

import type { AbstractError } from '@/domain/errors/abstract.js';
import { BadRequestError } from '@/domain/errors/bad-request-error.js';
import { ConflictError } from '@/domain/errors/conflict-error.js';
import { ForbiddenError } from '@/domain/errors/forbidden-error.js';
import { NotFoundError } from '@/domain/errors/not-found-error.js';
import { ServerError } from '@/domain/errors/server-error.js';
import { UnauthorizedError } from '@/domain/errors/unauthorized-error.js';
import type { HttpClient } from '@/domain/protocols/http-client.js';

export class FetchHttpClient implements HttpClient {
    public async request<T>(params: HttpClient.Request): Promise<HttpClient.Response<T>> {
        let response: Response;

        try {
            response = await fetch(params.url, {
                method: params.method,
                headers: { 'Content-Type': 'application/json', ...params.headers },
                body: params.body ? JSON.stringify(params.body) : undefined
            });
        } catch {
            throw new ServerError('Erro ao conectar com o servidor');
        }

        const body = await response.json().catch(() => null);

        if (!response.ok) {
            throw this.mapError(response.status, body);
        }

        return { statusCode: response.status, body: body as T };
    }

    private mapError(status: number, body: unknown): AbstractError {
        const message = this.extractMessage(body);
        switch (status) {
            case 400: return new BadRequestError(message);
            case 401: return new UnauthorizedError(message);
            case 403: return new ForbiddenError(message);
            case 404: return new NotFoundError(message);
            case 409: return new ConflictError(message);
            default:  return new ServerError(message);
        }
    }

    private extractMessage(body: unknown): string | undefined {
        if (body && typeof body === 'object') {
            const err = (body as Record<string, unknown>).error;
            if (err && typeof err === 'object') {
                const msg = (err as Record<string, unknown>).message;
                return typeof msg === 'string' ? msg : undefined;
            }
        }
        return undefined;
    }
}
```

## Mapeamento de status HTTP → erros de domínio

| Status HTTP | Erro de domínio | Significado |
|---|---|---|
| 400 | `BadRequestError` | Requisição malformada, validação falhou |
| 401 | `UnauthorizedError` | Sessão expirada, token inválido |
| 403 | `ForbiddenError` | Sem permissão para o recurso |
| 404 | `NotFoundError` | Registro não existe |
| 409 | `ConflictError` | Conflito de estado: recurso já existe, duplicidade |
| outros (4xx, 5xx) | `ServerError` | Erro interno, timeout, gateway |
| falha de rede (catch) | `ServerError` | Sem conexão, CORS, timeout de rede |

## Como usar em um repositório

```typescript
// src/infra/repositories/customer-repository-impl.ts

import type { HttpClient } from '@/domain/protocols/http-client.js';
import type { CustomerRepository } from '@/domain/repositories/customer-repository.js';
import { CustomerModel, type CustomerResponse } from '@/domain/models/customer.js';

export class CustomerRepositoryImpl implements CustomerRepository {
    private readonly BASE_URL = '/sales-order/Customers';

    constructor(private readonly httpClient: HttpClient) {} // ← tipado pela interface

    public async findAll(): Promise<CustomerRepository.FindAll> {
        const response = await this.httpClient.request<{ value: CustomerResponse[] }>({
            url: this.BASE_URL,
            method: 'GET'
        });
        return CustomerModel.withFromResponseList(response.body.value);
    }
}
```

## Como é instanciado na main layer

```typescript
// src/main/factories/pages/sales-orders-page-factory.tsx

import { useMemo } from 'react';
import { FetchHttpClient } from '@/infra/http/fetch-http-client.js';
import { makeSalesOrdersController } from '@/main/factories/controllers/sales-orders-controller.js';
import { SalesOrdersPage } from '@/presentation/pages/sales-orders/sales-orders-page.js';

const httpClient = new FetchHttpClient(); // ← instância única por módulo, fora do componente

export const SalesOrdersPageFactory: React.FC = () => {
    const controller = useMemo(() => makeSalesOrdersController(httpClient), []);
    return <SalesOrdersPage controller={controller} />;
};
```

## Adicionando headers de autenticação

Quando a aplicação precisar enviar token de autenticação, estenda o `FetchHttpClient` ou crie uma subclasse:

```typescript
// src/infra/http/authenticated-http-client.ts

import { FetchHttpClient } from '@/infra/http/fetch-http-client.js';
import type { HttpClient } from '@/domain/protocols/http-client.js';

export class AuthenticatedHttpClient extends FetchHttpClient {
    public async request<T>(params: HttpClient.Request): Promise<HttpClient.Response<T>> {
        const token = sessionStorage.getItem('token');
        const headers = token
            ? { ...params.headers, Authorization: `Bearer ${token}` } // ← só envia o header quando há token
            : { ...params.headers };
        return super.request({ ...params, headers });
    }
}
```

> **Trade-off de segurança.** O `sessionStorage` é legível por qualquer script da página — um único XSS exfiltra o token. Para dados sensíveis, prefira um cookie `httpOnly`/`Secure` (gerenciado pelo backend) ou mantenha o token apenas em memória. Se usar `sessionStorage`, garanta uma CSP estrita. Note também que o header só é adicionado quando o token existe — nunca envie `Authorization: Bearer null`.

## Anti-padrões

❌ **Usar fetch diretamente nos repositórios (bypassar o HttpClient):**
```typescript
public async findAll(): Promise<CustomerModel[]> {
    const res = await fetch('/api/Customers'); // ← deve usar this.httpClient
    return CustomerModel.withFromResponseList(await res.json());
}
```

❌ **Retornar `Either` do `request()` (infra usa throw, não Either):**
```typescript
public async request<T>(params): Promise<Either<AbstractError, HttpClient.Response<T>>> {
    // ← Either é responsabilidade da application layer, não da infra
}
```

## Regras de ouro

1. `FetchHttpClient` é o **único lugar** onde `fetch` é chamado na aplicação.
2. Erros são **lançados** (`throw`) — não retornados como `Either`. A application layer captura e converte.
3. A classe é tipada pela interface `HttpClient` — repositórios dependem da interface, não da classe.
4. `extractMessage` tenta extrair uma mensagem legível do corpo da resposta de erro; se não encontrar, retorna `undefined` e o erro usa a mensagem padrão.
