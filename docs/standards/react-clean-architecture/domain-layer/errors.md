# Domain layer — Errors

Os erros de domínio formam uma **hierarquia tipada** que permite identificar o tipo de falha sem depender de strings ou magic numbers. São usados em toda a cadeia como `left()` no `Either<AbstractError, T>` — da infra que lança até o hook que exibe a mensagem.

## Estrutura

```
src/domain/errors/
├── abstract.ts       → base abstrata com code + message
├── bad-request-error.ts    → 400
├── unauthorized-error.ts   → 401
├── forbidden-error.ts      → 403
├── not-found-error.ts      → 404
├── conflict-error.ts       → 409
└── server-error.ts         → 500
```

Esses 7 arquivos são **gerados automaticamente pelo MCP** ao criar o projeto e não devem ser removidos.

## A classe base — `AbstractError`

```typescript
// src/domain/errors/abstract.ts

export abstract class AbstractError extends Error {
    public abstract readonly code: number;

    constructor(message: string) {
        super(message);
        this.name = this.constructor.name;
    }
}
```

Toda classe de erro herda `AbstractError`. A propriedade `code` é **obrigatória** e corresponde ao status HTTP equivalente. `this.name = this.constructor.name` garante que o nome da classe apareça corretamente no stack trace.

## As 6 subclasses HTTP

Todas geradas pelo MCP com o mesmo padrão:

```typescript
// src/domain/errors/bad-request-error.ts
import { AbstractError } from '@/domain/errors/abstract.js';

export class BadRequestError extends AbstractError {
    public readonly code = 400;
    constructor(message = 'Bad request') {
        super(message);
    }
}
```

```typescript
// src/domain/errors/unauthorized-error.ts
import { AbstractError } from '@/domain/errors/abstract.js';

export class UnauthorizedError extends AbstractError {
    public readonly code = 401;
    constructor(message = 'Unauthorized') {
        super(message);
    }
}
```

```typescript
// src/domain/errors/forbidden-error.ts
import { AbstractError } from '@/domain/errors/abstract.js';

export class ForbiddenError extends AbstractError {
    public readonly code = 403;
    constructor(message = 'Forbidden') {
        super(message);
    }
}
```

```typescript
// src/domain/errors/not-found-error.ts
import { AbstractError } from '@/domain/errors/abstract.js';

export class NotFoundError extends AbstractError {
    public readonly code = 404;
    constructor(message = 'Not found') {
        super(message);
    }
}
```

```typescript
// src/domain/errors/conflict-error.ts
import { AbstractError } from '@/domain/errors/abstract.js';

export class ConflictError extends AbstractError {
    public readonly code = 409;
    constructor(message = 'Conflict') {
        super(message);
    }
}
```

Use para operações que falham por conflito de estado — ex.: criar um recurso que já existe, duplicidade de registro.

```typescript
// src/domain/errors/server-error.ts
import { AbstractError } from '@/domain/errors/abstract.js';

export class ServerError extends AbstractError {
    public readonly code = 500;
    constructor(message = 'Internal server error') {
        super(message);
    }
}
```

## Onde cada erro é produzido e consumido

| Camada | Papel |
|---|---|
| `infra/http/fetch-http-client.ts` | **Produz** erros mapeando status HTTP → `XxxError` |
| `application/use-cases/` | **Captura** e retorna via `left(new ServerError(...))` |
| `presentation/controllers/` | **Recebe** via `Either.isLeft()` e repassa o `AbstractError` |
| `presentation/hooks/` | **Expõe** `error: AbstractError | null` no estado React |
| `presentation/pages/` | **Renderiza** a mensagem com `<ErrorMessage>` |

## Exemplo — Fluxo completo de um erro

```typescript
// 1. infra: mapeia status HTTP → erro de domínio
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

// 2. application: captura e retorna como Either — preserva o tipo do erro de domínio
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

// 3. presentation/controller: repassa o erro de domínio
public handleResult<T>(result: Either<AbstractError, T>): BaseControllerState<T> {
    if (result.isLeft()) {
        return { data: null, error: result.value }; // ← AbstractError
    }
    return { data: result.value, error: null };
}

// 4. presentation/hook: expõe no estado
result.fold(
    (error) => setState({ isLoading: false, error }), // ← AbstractError
    (data) => setState({ isLoading: false, data, error: null })
)
```

## Quando criar erros customizados

Em geral, os 6 erros gerados pelo MCP cobrem todos os cenários. Crie uma subclasse customizada apenas quando precisar de **código HTTP diferente** (ex.: 422 Unprocessable Entity) ou quando quiser distinguir semanticamente erros do mesmo código:

```typescript
// src/domain/errors/validation-error.ts
import { AbstractError } from '@/domain/errors/abstract.js';

export class ValidationError extends AbstractError {
    public readonly code = 422;
    constructor(message: string) {
        super(message);
    }
}
```

## Anti-padrões

❌ **Lançar `Error` genérico fora da hierarquia:**
```typescript
throw new Error('something went wrong'); // ← sem code, sem tipagem
```

❌ **Usar string para identificar o tipo de erro:**
```typescript
if (error.type === 'NOT_FOUND') { ... } // ← use instanceof ou .code
```

✅ **Correto — identificar por `instanceof` ou `code`:**
```typescript
if (result.isLeft()) {
    const err = result.value;
    if (err instanceof UnauthorizedError) {
        // redirecionar para login
    }
}
```

## Regras de ouro

1. Todo erro da aplicação herda `AbstractError` — nunca lançar `Error` nativo em código de negócio.
2. Os 6 erros HTTP gerados pelo MCP são **obrigatórios** — não remover, não renomear.
3. O `code` corresponde ao status HTTP equivalente — facilita logging e diagnóstico.
4. A mensagem padrão é em inglês; a mensagem exibida ao usuário vem dos arquivos de i18n.
5. Erros são lançados na `infra/` e **retornados** (via `Either`) pela `application/` — nunca relançados pela `presentation/`.
