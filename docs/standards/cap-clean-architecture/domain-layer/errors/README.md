# Errors

`domain/errors/` define a hierarquia de erros do negócio: uma classe `AbstractError` base + 6 subclasses obrigatórias correspondentes aos códigos HTTP mais comuns. Toda função que pode falhar de forma esperada (validação, recurso ausente, conflito, autorização) lança ou retorna instância de uma dessas classes — nunca `Error` cru, `string`, ou objeto plain.

Esses erros são o **lado esquerdo do `Either`** retornado pelos use cases (`Either<AbstractError, T>`). A presentation layer mapeia o `code` HTTP para a resposta; a application layer captura via `handleError` da `BaseUseCaseImpl`.

## Estrutura canônica

```
src/domain/errors/
├── abstract.ts                → AbstractError (classe base)
├── bad-request.ts             → BadRequestError (400)
├── unauthorized.ts            → UnauthorizedError (401)
├── forbidden.ts               → ForbiddenError (403)
├── not-found.ts               → NotFoundError (404)
├── conflict.ts                → ConflictError (409)
├── server.ts                  → ServerError (500)
└── index.ts                   → barrel: reexporta todas as classes
```

> **6 subclasses obrigatórias.** Cada projeto pode adicionar `BadGatewayError` (502) e `ServiceUnavailableError` (503) quando integra com sistemas externos. Não fundir 401 (`Unauthorized`) com 403 (`Forbidden`) — anti-padrão histórico de LE44, MRO e RVE.

## `AbstractError` — template canônico

`AbstractError` é uma classe abstrata simples. Cada service do projeto **duplica** essa classe (e as 6 subclasses) — não existe pacote compartilhado entre services. Use o código abaixo como template canônico para reduzir desvios.

```typescript
// src/domain/errors/abstract.ts
export abstract class AbstractError extends Error {
    public readonly code: number;
    public readonly args?: string[];

    constructor(message: string, code: number, stack?: string, args?: string[]) {
        super(message);
        this.name = this.constructor.name;
        this.code = code;
        this.args = args;
        if (stack) {
            this.stack = stack;
        }
    }
}
```

Pontos do template:

- **`abstract` class** — não pode ser instanciada diretamente; força uso de subclasse.
- **`public readonly code`** — código HTTP imutável após construção.
- **`args?: string[]`** — argumentos para interpolação de mensagem (i18n) — opcional.
- **`this.name = this.constructor.name`** — `error.name` reflete o nome da subclasse (`BadRequestError`, `NotFoundError`, etc.) automaticamente.
- **`stack?: string`** — sobrescritível para preservar stack original quando o erro é re-lançado.
- **Sem `details: unknown`** — não polua a base com campos opcionais não utilizados; quando precisar, a subclasse específica adiciona.
- **Sem `toErrorDetails()` na base** — formatação de resposta é responsabilidade da presentation layer, não do erro.

## Subclasses obrigatórias

Cada subclasse fixa o código HTTP no `super(...)` — o consumidor **nunca** passa o código.

### `BadRequestError` — 400

```typescript
// src/domain/errors/bad-request.ts
import { AbstractError } from './abstract.js';

export class BadRequestError extends AbstractError {
    constructor(message: string, stack?: string, args?: string[]) {
        super(message, 400, stack, args);
    }
}
```

Uso típico: payload inválido, regra de entrada violada, validação de model falhou.

### `UnauthorizedError` — 401

```typescript
// src/domain/errors/unauthorized.ts
import { AbstractError } from './abstract.js';

export class UnauthorizedError extends AbstractError {
    constructor(message: string, stack?: string, args?: string[]) {
        super(message, 401, stack, args);
    }
}
```

Uso típico: usuário não autenticado, token ausente ou inválido.

> **Nunca** use `ForbiddenError` (403) para falta de autenticação — semanticamente são distintos. 401 = "não sei quem você é"; 403 = "sei quem você é mas você não pode".

### `ForbiddenError` — 403

```typescript
// src/domain/errors/forbidden.ts
import { AbstractError } from './abstract.js';

export class ForbiddenError extends AbstractError {
    constructor(message: string, stack?: string, args?: string[]) {
        super(message, 403, stack, args);
    }
}
```

Uso típico: usuário autenticado mas sem permissão para a operação (`PermissionChecker.checkPermission` retornou `false`).

### `NotFoundError` — 404

```typescript
// src/domain/errors/not-found.ts
import { AbstractError } from './abstract.js';

export class NotFoundError extends AbstractError {
    constructor(message: string, stack?: string, args?: string[]) {
        super(message, 404, stack, args);
    }
}
```

Uso típico: `findById` retornou `null` e o use case considera ausência um erro (em vez de retornar `right(null)`).

> **Quando `null` é resultado válido**, o use case retorna `right(null)` — **não** lança `NotFoundError`. A escolha depende da semântica do contrato.

### `ConflictError` — 409

```typescript
// src/domain/errors/conflict.ts
import { AbstractError } from './abstract.js';

export class ConflictError extends AbstractError {
    constructor(message: string, stack?: string, args?: string[]) {
        super(message, 409, stack, args);
    }
}
```

Uso típico: tentativa de criar registro com chave já existente, transição de estado incompatível, edição concorrente detectada.

### `ServerError` — 500

```typescript
// src/domain/errors/server.ts
import { AbstractError } from './abstract.js';

export class ServerError extends AbstractError {
    constructor(stack?: string, message: string = 'internalServerError', args?: string[]) {
        super(message, 500, stack, args);
    }
}
```

Uso típico: erro inesperado, falha de I/O, exceção genérica capturada por `handleError` da `BaseUseCaseImpl`. A ordem dos parâmetros é **`(stack, message)`** intencionalmente — `stack` primeiro porque o caso comum é `new ServerError(error.stack)` com mensagem default.

## Subclasses opcionais — integrações externas

Quando o service integra com APIs externas (S/4, REST, AWS), as subclasses 502/503 aparecem para sinalizar que o erro veio "de fora":

```typescript
// src/domain/errors/bad-gateway.ts
import { AbstractError } from './abstract.js';

export class BadGatewayError extends AbstractError {
    constructor(message: string, stack?: string, args?: string[]) {
        super(message, 502, stack, args);
    }
}
```

```typescript
// src/domain/errors/service-unavailable.ts
import { AbstractError } from './abstract.js';

export class ServiceUnavailableError extends AbstractError {
    constructor(message: string, stack?: string, args?: string[]) {
        super(message, 503, stack, args);
    }
}
```

> **Critério de inclusão:** adicionar 502/503 **só quando o service tem adapter externo**. Não criar por antecipação — mantém o conjunto de erros enxuto.

## Barrel `index.ts`

`errors/index.ts` é o **único barrel obrigatório** dentro de `src/domain/`. Reexporta todas as classes para imports simples no resto do código.

```typescript
// src/domain/errors/index.ts
export { AbstractError } from './abstract.js';
export { BadRequestError } from './bad-request.js';
export { UnauthorizedError } from './unauthorized.js';
export { ForbiddenError } from './forbidden.js';
export { NotFoundError } from './not-found.js';
export { ConflictError } from './conflict.js';
export { ServerError } from './server.js';
```

Quando o service inclui as opcionais, adicionar após `ServerError`:

```typescript
export { BadGatewayError } from './bad-gateway.js';
export { ServiceUnavailableError } from './service-unavailable.js';
```

Uso no consumidor (application layer):

```typescript
// src/application/use-cases/actions/save-theme.ts
import { BadRequestError, NotFoundError } from '@/domain/errors/index.js';
```

## Uso com `Either<AbstractError, T>`

Use cases retornam `Either<AbstractError, T>` — o `left` é uma instância da hierarquia. A application layer importa `left`/`right` de `@sweet-monads/either` (o domain só declara o tipo `Result = Either<AbstractError, T>`).

### Contrato no domain

```typescript
// src/domain/use-cases/actions/save-theme.ts
import type { Either } from '@sweet-monads/either';
import type { AbstractError } from '@/domain/errors/abstract.js';

export namespace SaveThemeUseCase {
    export type Params = {
        tenantId: string;
        primaryColor: string;
    };
    export type Result = Either<AbstractError, void>;
}

export interface SaveThemeUseCase {
    execute(params: SaveThemeUseCase.Params): Promise<SaveThemeUseCase.Result>;
}
```

### Implementação na application layer

```typescript
// src/application/use-cases/actions/save-theme.ts
import { left, right } from '@sweet-monads/either';

import { BaseUseCaseImpl } from '@/application/use-cases/base/base.js';
import { BadRequestError, NotFoundError } from '@/domain/errors/index.js';
import { SaveThemeUseCase } from '@/domain/use-cases/actions/save-theme.js';

export class SaveThemeUseCaseImpl extends BaseUseCaseImpl implements SaveThemeUseCase {
    public async execute(params: SaveThemeUseCase.Params): Promise<SaveThemeUseCase.Result> {
        try {
            if (!params.primaryColor) {
                return left(new BadRequestError('theme.primaryColor.required'));
            }
            const tenant = await this.tenantRepo.findById(params.tenantId);
            if (!tenant) {
                return left(new NotFoundError('tenant.notFound'));
            }
            // ... persistência
            return right(undefined);
        } catch (error) {
            return left(this.handleError(error));
        }
    }
}
```

> O `handleError` da `BaseUseCaseImpl` (ver [`application-layer/use-cases/base.md`](../../application-layer/use-cases/base.md)) preserva instâncias de `AbstractError` e converte qualquer outra exceção em `ServerError`. Essa é a razão de **`handleError(error)` retornar `AbstractError`** — qualquer subclasse cabe.

## Mensagens — chaves i18n vs strings literais

A `message` passada ao construtor pode ser:

1. **Chave i18n** (ex.: `'tenant.notFound'`, `'theme.primaryColor.required'`) — recomendado quando o service tem `Translator` configurado.
2. **String literal traduzida** (ex.: `'Tenant não encontrado'`) — aceitável para services sem i18n ou para mensagens internas.

A escolha é por service, não por erro individual. Decidida no projeto e mantida coerente.

> **`AbstractError` não invoca `Translator`.** A tradução é responsabilidade da presentation layer (ou do controller que serializa a resposta), nunca do próprio erro. Manter o domain livre de dependências.

## Tipo `Either` — único pacote externo permitido

```typescript
import type { Either } from '@sweet-monads/either';
```

Importar `Either` é a única dependência externa aceita no domain. `left()` e `right()` (funções) **não** são importados no domain — apenas o tipo `Either<L, R>`. Quem chama `left/right` é a application layer.

## Regras

1. **`AbstractError` é `abstract`** — nunca instanciar diretamente. Toda exceção em domain é uma subclasse.
2. **Cada subclasse fixa seu `code`** no `super(message, <code>, ...)` — consumidor nunca passa o código.
3. **6 subclasses obrigatórias:** `BadRequest` 400, `Unauthorized` 401, `Forbidden` 403, `NotFound` 404, `Conflict` 409, `Server` 500.
4. **Não fundir 401 com 403** — semânticas distintas.
5. **`ServerError` tem `(stack, message?)` invertido** — caso comum é `new ServerError(error.stack)`; mensagem default = `'internalServerError'`.
6. **`AbstractError` é duplicado por service** — sem pacote compartilhado entre services. Use o template canônico para reduzir desvios.
7. **Sem `Translator` no construtor** — mensagens são strings (chave i18n ou literal). Tradução fica fora do domain.
8. **Barrel `errors/index.ts` obrigatório** — reexporta todas as classes.
9. **Nome do arquivo base:** `abstract.ts` (não `abstract-error.ts`) — alinhado com 3 de 4 projetos modelo (LE44, RVE, Suzano).
10. **`name` setado via `this.constructor.name`** — automático e consistente com a subclasse.

## Anti-padrões

### 1. Nome de subclasse com typo ou inconsistente com HTTP

```typescript
// ❌ ERRADO — typo histórico de Suzano
export class ForbidenRequestError extends AbstractError { /* ... */ }
```

```typescript
// ✅ CORRETO — nome em inglês alinhado ao status HTTP
export class ForbiddenError extends AbstractError { /* ... */ }
```

### 2. Fundir 401 em 403

```typescript
// ❌ ERRADO — falta de autenticação retornada como Forbidden (403)
return left(new ForbiddenError('userNotAuthenticated'));
```

```typescript
// ✅ CORRETO — Unauthorized (401) para autenticação
return left(new UnauthorizedError('userNotAuthenticated'));
```

### 3. `code` mutável ou setado pelo consumidor

```typescript
// ❌ ERRADO — code passado pelo consumidor
return left(new BadRequestError('invalid', 422));
```

```typescript
// ✅ CORRETO — código fixo na subclasse
// Para 422: use UnprocessableEntityError (subclasse opcional, se o service precisar)
return left(new BadRequestError('invalid'));
```

### 4. Getter/setter recursivo de `message`

```typescript
// ❌ ERRADO — typo crítico observado em Suzano (bad-request.ts:3-14)
export class BadRequestError extends AbstractError {
    public get message(): string {
        return this.message; // ← infinite recursion → stack overflow
    }
    public set message(value: string) {
        this.message = value;
    }
}
```

```typescript
// ✅ CORRETO — sem getter/setter; message é da Error base
export class BadRequestError extends AbstractError {
    constructor(message: string, stack?: string, args?: string[]) {
        super(message, 400, stack, args);
    }
}
```

### 5. `Translator` injetado na classe de erro

```typescript
// ❌ ERRADO — erro depende do translator
export class BadRequestError extends AbstractError {
    constructor(translator: Translator, key: string) {
        super(translator.translate(key), 400);
    }
}
```

```typescript
// ✅ CORRETO — erro carrega chave; quem traduz é a presentation
export class BadRequestError extends AbstractError {
    constructor(message: string, stack?: string, args?: string[]) {
        super(message, 400, stack, args);
    }
}
```

### 6. `LoadStatusError` ou similar como objeto plain

```typescript
// ❌ ERRADO — objeto plain em vez de subclasse de AbstractError
export type LoadStatusError = {
    code: number;
    message: string;
};
```

```typescript
// ✅ CORRETO — subclasse de AbstractError
// Se o domínio tem caso específico, criar:
export class LoadStatusError extends AbstractError {
    constructor(message: string, stack?: string, args?: string[]) {
        super(message, 409, stack, args); // 409 Conflict — estado incompatível
    }
}
```

### 7. `errors/types.ts` declarando shape de gateway externo

```typescript
// ❌ ERRADO — domain/errors/types.ts com tipos auxiliares
export type GatewayError = { /* ... */ };
export type ErrorDetails = { /* ... */ };
```

```typescript
// ✅ CORRETO — gateway tem subclasse própria; details fica no namespace de quem usa
export class BadGatewayError extends AbstractError { /* ... */ }
// Tipos específicos de payload externo vivem no namespace do adapter:
// domain/adapters/external-api/sap/payment.ts → namespace SapPaymentApi { ... }
```

### 8. Nome de arquivo `abstract-error.ts`

```
❌  src/domain/errors/abstract-error.ts   (padrão MRO — divergente)
✅  src/domain/errors/abstract.ts          (LE44, RVE, Suzano — canônico)
```

### 9. Subclasses sem reexport no barrel

```typescript
// ❌ ERRADO — UnprocessableEntity criado mas fora do index.ts (Suzano)
// errors/unprocessable-entity.ts existe mas index.ts não reexporta
```

```typescript
// ✅ CORRETO — toda subclasse no barrel
// errors/index.ts
export { AbstractError } from './abstract.js';
export { BadRequestError } from './bad-request.js';
export { UnauthorizedError } from './unauthorized.js';
export { ForbiddenError } from './forbidden.js';
export { NotFoundError } from './not-found.js';
export { ConflictError } from './conflict.js';
export { ServerError } from './server.js';
```
