# controllers/base/

Define o contrato base de todos os controllers da presentation layer: tipos de resposta e métodos utilitários de envelope.

## Arquivo: `controllers/base/controller.ts`

```typescript
// src/presentation/controllers/base/controller.ts

export type ErrorDetails = {
    status: number;
    message: string;
    target: string;
};

export type ErrorData = {
    code: string;
    details: ErrorDetails[];
};

export type BaseControllerResponse = {
    status: number;
    data?: unknown;
    errorData?: ErrorData;
};

export class BaseController {
    public success(data: unknown): BaseControllerResponse {
        return { status: 200, data };
    }

    public error(code: number, details: ErrorDetails[]): BaseControllerResponse {
        return {
            status: code,
            errorData: {
                code: 'MULTIPLE_ERRORS',
                details,
            },
        };
    }
}
```

---

## Responsabilidade de cada tipo

| Tipo | Para que serve |
|---|---|
| `ErrorDetails` | Um item de erro individual: código HTTP, mensagem i18n e campo-alvo que disparou o erro |
| `ErrorData` | Envelope de erro: agrupa uma lista de `ErrorDetails` sob um código de erro de grupo |
| `BaseControllerResponse` | Resposta unificada: sempre tem `status`; erros preenchem `errorData`; sucessos preenchem `data` |
| `BaseController` | Classe base concreta; não é abstrata — controllers estendem via `extends BaseController` |

---

## Contrato com `domain/errors/`

Para que `this.error(result.value.code, result.value.toErrorDetails())` funcione, toda classe de erro do domain precisa expor o método `toErrorDetails(targetField?: string): ErrorDetails[]`:

```typescript
// src/domain/errors/abstract-error.ts
import type { ErrorDetails } from '@/presentation/controllers/base/controller.js';

export abstract class AbstractError extends Error {
    public abstract code: number;

    public toErrorDetails(targetField = 'unknown'): ErrorDetails[] {
        return [{ status: this.code, message: this.message, target: targetField }];
    }
}
```

```typescript
// src/domain/errors/bad-request.ts
import { AbstractError } from './abstract-error.js';

export class BadRequestError extends AbstractError {
    public readonly code = 400;

    constructor(message: string) {
        super(message);
        this.name = 'BadRequestError';
    }
}
```

```typescript
// src/domain/errors/not-found.ts
import { AbstractError } from './abstract-error.js';

export class NotFoundError extends AbstractError {
    public readonly code = 404;

    constructor(message: string) {
        super(message);
        this.name = 'NotFoundError';
    }
}
```

---

## Como `routes/index.ts` consome a resposta

```typescript
// src/main/routes/index.ts (trecho)
async function handleConfirmOrder(request: any) {
    const result = await confirmOrderController.handle(request);
    if (result.status >= 400) {
        return request.reject(result.errorData);   // ← errorData vai direto para o CAP
    }
    return result.data;
}
```

---

## Regras

- `BaseController` é **concreta** — não `abstract`. Controllers estendem e não precisam implementar método nenhum da base.
- O `code` em `ErrorData` é sempre a string `'MULTIPLE_ERRORS'` — o código HTTP está dentro de cada `ErrorDetails.status`.
- `BaseControllerResponse.data` é `unknown` para manter a base genérica; o caller (`routes/index.ts`) faz o cast quando necessário.
- Nunca acrescentar lógica de negócio à base. Se sentir necessidade, a lógica pertence ao use case.
