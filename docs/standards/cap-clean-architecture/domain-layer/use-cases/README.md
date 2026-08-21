# Use Cases

`domain/use-cases/` declara os contratos das operações de negócio que a aplicação expõe. Cada use case é uma `interface XxxUseCase` com um único método público (`execute`) e um `namespace XxxUseCase` colateral que declara `Params` e `Result`. As implementações vivem em `application/use-cases/` (ver [`application-layer/use-cases/`](../../application-layer/use-cases/base.md)); os controllers da presentation layer consomem o contrato via `XxxUseCase.Params`.

> **Regra de ouro:** todo use case retorna `Either<AbstractError, T>`. Caminho feliz = `right(valor)`; erros de domínio = `left(new XxxError(...))`. Nunca usar `throw` no contrato — `throw` é detalhe de implementação que o `handleError` da `BaseUseCaseImpl` converte em `Either`.

## Estrutura de subpastas

A subdivisão espelha o tipo do hook CDS no `index.cds`:

```
src/domain/use-cases/
├── actions/                              → operações de comando (action no index.cds)
│   ├── save-theme.ts                     → SaveThemeUseCase
│   ├── checkout.ts                       → CheckoutUseCase
│   └── loss-provisions/
│       └── mass-update/
│           ├── before-execute.ts         → BeforeExecuteMassUpdateUseCase
│           ├── mass-update.ts            → MassUpdateUseCase
│           └── index.ts                  → barrel das interfaces da pasta
├── functions/                            → operações de consulta (function no index.cds)
│   ├── find-loss-provision-period.ts     → FindLossProvisionPeriodUseCase
│   └── catalog-search.ts                 → CatalogSearchUseCase
└── entity-events/                        → hooks de entidade (before/after CRUD)
    ├── price-lists/
    │   ├── before-create.ts              → BeforeCreatePriceListUseCase
    │   ├── before-update.ts
    │   ├── before-delete.ts
    │   └── index.ts                      → barrel da entidade
    ├── user-preferences/
    │   ├── before-read.ts
    │   └── index.ts
    └── flights/
        ├── before-create.ts
        ├── after-create.ts
        └── index.ts
```

| Subpasta | Quando criar contrato |
|---|---|
| `actions/<nome>.ts` | Para cada `action` declarada no `index.cds` (comando — modifica estado ou aceita objetos/arrays como input) |
| `actions/<nome>/before-execute.ts` + `<nome>.ts` | Quando a action precisa de pré-processamento separado (validação de permissão, enriquecimento de payload). Ver [application-layer/use-cases/actions.md → Variação `before-execute`](../../application-layer/use-cases/actions.md#variação-action-com-before-execute) |
| `functions/<nome>.ts` | Para cada `function` declarada no `index.cds` (consulta com parâmetros escalares) |
| `entity-events/<entidade-plural>/<evento>.ts` | Para cada hook (`before-create`, `before-read`, `before-update`, `before-delete`, `after-create`, `after-read`, `after-update`, `after-delete`) que a entidade realmente usa |
| `entity-events/<entidade-plural>/index.ts` | Barrel **obrigatório** — reexporta os contratos da entidade (alinha com a application layer e a main layer/factories) |

## Shape canônico — interface + namespace

```typescript
// src/domain/use-cases/actions/checkout.ts
import type { Either } from '@sweet-monads/either';

import type { AbstractError } from '@/domain/errors/abstract.js';

export namespace CheckoutUseCase {
    export type Params = {
        tenantId: string;
        userId: string;
        cartId: string;
    };
    export type Result = Either<AbstractError, CheckoutUseCase.CheckoutResult>;

    export type CheckoutResult = {
        orderId: string;
        totalAmount: number;
    };
}

export interface CheckoutUseCase {
    execute(params: CheckoutUseCase.Params): Promise<CheckoutUseCase.Result>;
}
```

Pontos canônicos:

- **Método único `execute(params): Promise<Result>`** — sempre. Sem `executeBatch`, `validate`, `prepare` ou auxiliares públicos.
- **`Params` é um objeto** — nunca lista de argumentos posicionais. Mesmo para um único campo, manter como `{ id: string }`.
- **`Result = Either<AbstractError, T>`** — `T` pode ser `void`, `string`, `XxxModel`, array, struct customizado.
- **`Result` direto, não `Promise<Either<...>>`** — o `Promise` envolve apenas a assinatura de `execute`. Ver anti-padrão #4 abaixo.
- **Tipos auxiliares dentro do namespace** — `CheckoutResult`, `LineItem`, qualquer struct específico do contrato.

## Variação `entity-events/<entidade>/before-create.ts`

Hooks de entidade usam o nome do evento como nome de classe/interface, espelhado em PascalCase com a entidade plural:

```typescript
// src/domain/use-cases/entity-events/operation-flight-types/before-create.ts
import type { Either } from '@sweet-monads/either';

import type { AbstractError } from '@/domain/errors/abstract.js';

export namespace BeforeCreateOperationFlightTypeUseCase {
    export type Params = {
        id: string;
        name: string;
        description?: string;
    };
    export type Result = Either<AbstractError, BeforeCreateOperationFlightTypeUseCase.CreationPayload>;

    export type CreationPayload = {
        id: string;
        name: string;
        description: string | null;
        createdAt: Date;
    };
}

export interface BeforeCreateOperationFlightTypeUseCase {
    execute(
        params: BeforeCreateOperationFlightTypeUseCase.Params,
    ): Promise<BeforeCreateOperationFlightTypeUseCase.Result>;
}
```

> A interface se chama `BeforeCreateOperationFlightTypeUseCase` (entidade no **singular** dentro do nome da classe, mesmo a pasta sendo plural). O padrão é: `<Evento><Ação><EntidadeSingular>UseCase`.

### Barrel da entidade — `index.ts`

```typescript
// src/domain/use-cases/entity-events/operation-flight-types/index.ts
export type { BeforeCreateOperationFlightTypeUseCase } from './before-create.js';
export type { BeforeUpdateOperationFlightTypeUseCase } from './before-update.js';
export type { AfterReadOperationFlightTypeUseCase } from './after-read.js';
```

Esse barrel é consumido pela application layer e pelas factories da main layer:

```typescript
// src/main/factories/use-cases/entity-events/operation-flight-types/before-create.ts (trecho)
import type { BeforeCreateOperationFlightTypeUseCase } from '@/domain/use-cases/entity-events/operation-flight-types/index.js';
```

## Variação `actions/<nome>/` com `before-execute`

Quando uma action no CAP exige dois handlers (`service.before('action')` + `service.on('action')`), o domain declara dois contratos na mesma pasta:

```
src/domain/use-cases/actions/loss-provisions/mass-update/
├── before-execute.ts            → BeforeExecuteMassUpdateUseCase
├── mass-update.ts               → MassUpdateUseCase
└── index.ts                     → barrel
```

```typescript
// src/domain/use-cases/actions/loss-provisions/mass-update/before-execute.ts
export namespace BeforeExecuteMassUpdateUseCase {
    export type Params = { request: BeforeExecuteMassUpdateUseCase.RequestShape };
    export type Result = Either<AbstractError, void>;
    export type RequestShape = { user: { email: string; roles: string[] } };
}

export interface BeforeExecuteMassUpdateUseCase {
    execute(params: BeforeExecuteMassUpdateUseCase.Params): Promise<BeforeExecuteMassUpdateUseCase.Result>;
}
```

```typescript
// src/domain/use-cases/actions/loss-provisions/mass-update/mass-update.ts
export namespace MassUpdateUseCase {
    export type Params = {
        items: MassUpdateUseCase.Item[];
    };
    export type Result = Either<AbstractError, MassUpdateUseCase.UpdatedSummary>;

    export type Item = { id: string; status: string };
    export type UpdatedSummary = { updatedCount: number };
}

export interface MassUpdateUseCase {
    execute(params: MassUpdateUseCase.Params): Promise<MassUpdateUseCase.Result>;
}
```

> O `RequestShape` no namespace evita acoplamento com `@sap/cds.Request` — declara apenas o subset de campos que o use case efetivamente lê.

## Tipos auxiliares — sempre no namespace

Quando o use case manipula structs internas (resultado parcial, item de coleção, accumulator de pipeline), os tipos vão dentro do `namespace XxxUseCase`. Evita poluição do escopo do módulo e permite imports concisos.

```typescript
// src/domain/use-cases/actions/checkout.ts
export namespace CheckoutUseCase {
    export type Params = { /* ... */ };
    export type Result = Either<AbstractError, CheckoutUseCase.CheckoutResult>;

    export type CheckoutResult = {
        orderId: string;
        totalAmount: number;
        items: CheckoutUseCase.LineItem[];
    };

    export type LineItem = {
        sku: string;
        quantity: number;
        unitPrice: number;
    };

    export type PipelineContext = {
        session: CheckoutUseCase.SessionShape;
        accumulatedSteps: string[];
    };

    export type SessionShape = { id: string; userId: string };
}
```

> A application layer usa `CheckoutUseCase.LineItem` e `CheckoutUseCase.PipelineContext` — sem precisar declarar tipos locais no arquivo de implementação. Reforça a regra "[application/ não declara tipos](../../application-layer/use-cases/base.md#anti-padrão-tipos-e-interfaces-declarados-no-arquivo-da-application-layer)".

## Sem `Either<ServerError, ...>` específico

O `left` é **sempre** `AbstractError` — qualquer subclasse cabe. O contrato não restringe o tipo de erro porque diferentes paths do use case podem retornar diferentes erros.

```typescript
// ❌ ERRADO — restringe a um único erro
export type Result = Either<ServerError, T>;

// ❌ ERRADO — restringe a uma união manual
export type Result = Either<BadRequestError | NotFoundError, T>;

// ✅ CORRETO — base abstrata cobre todas as subclasses
export type Result = Either<AbstractError, T>;
```

> Restringir o tipo do erro torna inflexível a evolução: adicionar uma nova classe de erro ao path do use case quebra a tipagem do controller. `AbstractError` no contrato + classes específicas no `left()` da implementação é o equilíbrio canônico.

## `Result = Either<..., void>` — quando o caminho feliz não devolve dado

Use cases que apenas mutam estado (save, delete) retornam `Either<AbstractError, void>` no contrato. A implementação faz `return right(undefined)`.

```typescript
export namespace SaveThemeUseCase {
    export type Params = { tenantId: string; primaryColor: string };
    export type Result = Either<AbstractError, void>;
}

export interface SaveThemeUseCase {
    execute(params: SaveThemeUseCase.Params): Promise<SaveThemeUseCase.Result>;
}
```

## Imports permitidos

| O que pode importar | Exemplo |
|---|---|
| Tipo `Either` (essencial) | `import type { Either } from '@sweet-monads/either';` |
| `AbstractError` | `import type { AbstractError } from '@/domain/errors/abstract.js';` |
| Models (apenas tipos) | `import type { PriceListModel } from '@/domain/models/db/price-list.js';` |
| Outros use cases (raro — só em namespaces compartilhados) | `import type { CheckoutUseCase } from '@/domain/use-cases/actions/checkout.js';` |

| O que **não** pode importar | Motivo |
|---|---|
| Repositories, adapters, hydrators, utils | Use case **declara** o que precisa via `Params`/`Result`; quem injeta é a application layer |
| `@sap/cds`, `@models/*` | Acoplamento com framework — usar wrapper local no namespace |
| `application/`, `infrastructure/`, `presentation/`, `main/` | Quebra de isolamento |

> **Atenção:** o contrato do use case **não importa** o `XxxRepository` que sua implementação usa. O repository é detalhe de **implementação** — fica injetado no `XxxUseCaseImpl` em `application/`. O domain do use case só conhece `Params`/`Result`.

## Tipos auxiliares de request CAP — wrapper local

Quando o use case `entity-event` precisa do shape do `Request` CAP (`request.user`, `request.data`, `request.query.SELECT.where`), declarar wrapper local no namespace — **não importar de `@sap/cds`**:

```typescript
// src/domain/use-cases/entity-events/user-preferences/before-read.ts
export namespace BeforeReadUserPreferencesUseCase {
    export type Params = {
        request: BeforeReadUserPreferencesUseCase.MutableRequest;
        userId: string;
    };
    export type Result = Either<AbstractError, void>;

    export type MutableRequest = {
        query: {
            SELECT: {
                where?: unknown[];
            };
        };
    };
}

export interface BeforeReadUserPreferencesUseCase {
    execute(params: BeforeReadUserPreferencesUseCase.Params): Promise<BeforeReadUserPreferencesUseCase.Result>;
}
```

> O `MutableRequest` documenta o subset do request que o use case toca. Mesma estratégia dos hydrators (ver [hydrators/README.md](../hydrators/README.md)).

## Regras

1. **Método único `execute(params): Promise<XxxUseCase.Result>`.** Sem métodos auxiliares públicos.
2. **`Params` é um objeto** — sempre struct, nunca lista posicional.
3. **`Result = Either<AbstractError, T>`** — `T` pode ser `void`. `AbstractError` é a base — não restringir a subclasses específicas.
4. **`namespace XxxUseCase`** agrupa `Params`, `Result` e tipos auxiliares.
5. **Tipos auxiliares no namespace** — sem declaração top-level no arquivo, sem pasta `types/`.
6. **Barrel obrigatório em `entity-events/<entidade>/index.ts`** — reexporta os contratos da entidade.
7. **Sem framework** — wrapper local quando precisar do shape de `Request`/`Transaction` do CAP.
8. **Sem dependência de outros artefatos do domain** que não sejam `Either`, `AbstractError` e Models (apenas tipos).
9. **Naming:**
   - Arquivo: `kebab-case.ts` (singular para action; nome do evento para entity-event).
   - Pasta de entidade: `kebab-case-plural`.
   - Interface: `<Evento?><Ação><EntidadeSingular?>UseCase` (PascalCase).

## Anti-padrões

### 1. Múltiplos métodos públicos

```typescript
// ❌ ERRADO — interface com mais de um método público
export interface CheckoutUseCase {
    execute(params: CheckoutUseCase.Params): Promise<CheckoutUseCase.Result>;
    validate(params: CheckoutUseCase.Params): Promise<boolean>;
    rollback(orderId: string): Promise<void>;
}
```

```typescript
// ✅ CORRETO — apenas execute; auxiliares são privados na implementação
export interface CheckoutUseCase {
    execute(params: CheckoutUseCase.Params): Promise<CheckoutUseCase.Result>;
}
```

### 2. Tipos top-level no arquivo

```typescript
// ❌ ERRADO — tipos no module-level
export type CheckoutResult = { orderId: string; total: number };
export type LineItem = { sku: string; quantity: number };

export interface CheckoutUseCase {
    execute(params: { items: LineItem[] }): Promise<Either<AbstractError, CheckoutResult>>;
}
```

```typescript
// ✅ CORRETO — tipos no namespace
export namespace CheckoutUseCase {
    export type Params = { items: CheckoutUseCase.LineItem[] };
    export type Result = Either<AbstractError, CheckoutUseCase.CheckoutResult>;
    export type CheckoutResult = { orderId: string; total: number };
    export type LineItem = { sku: string; quantity: number };
}

export interface CheckoutUseCase {
    execute(params: CheckoutUseCase.Params): Promise<CheckoutUseCase.Result>;
}
```

### 3. `Result = Either<ServerError, T>` ou subclasse específica

```typescript
// ❌ ERRADO — restringe ao tipo do ServerError
export namespace ValidatePreFlightStatusUseCase {
    export type Result = Either<ServerError, ValidationOutcome>;
}
```

```typescript
// ✅ CORRETO — sempre AbstractError
export namespace ValidatePreFlightStatusUseCase {
    export type Result = Either<AbstractError, ValidationOutcome>;
}
```

### 4. Dupla `Promise<Either<...>>` aninhada

```typescript
// ❌ ERRADO — Result já é Promise; execute retorna Promise<Result> = Promise<Promise<Either<...>>>
export namespace SaveAircraftUseCase {
    export type Result = Promise<Either<AbstractError, AircraftModel>>;
}

export interface SaveAircraftUseCase {
    execute(params: SaveAircraftUseCase.Params): Promise<SaveAircraftUseCase.Result>;
}
```

```typescript
// ✅ CORRETO — Result é o Either; execute envelopa em Promise
export namespace SaveAircraftUseCase {
    export type Result = Either<AbstractError, AircraftModel>;
}

export interface SaveAircraftUseCase {
    execute(params: SaveAircraftUseCase.Params): Promise<SaveAircraftUseCase.Result>;
}
```

### 5. Use case sem `Either` (retorna `void` plain)

```typescript
// ❌ ERRADO — sem Either, perde semântica de erro
export interface SaveCarrierInvoiceDataUseCase {
    execute(params: SaveCarrierInvoiceDataUseCase.Params): Promise<void>;
}
```

```typescript
// ✅ CORRETO — Either<AbstractError, void>
export namespace SaveCarrierInvoiceDataUseCase {
    export type Result = Either<AbstractError, void>;
}

export interface SaveCarrierInvoiceDataUseCase {
    execute(params: SaveCarrierInvoiceDataUseCase.Params): Promise<SaveCarrierInvoiceDataUseCase.Result>;
}
```

### 6. Use case com `{ hasError, error? }` em vez de `Either`

```typescript
// ❌ ERRADO — desvio de LE44 plant-validation
export namespace PlantValidationUseCase {
    export type Result = { hasError: boolean; error?: AbstractError };
}
```

```typescript
// ✅ CORRETO — Either uniforme
export namespace PlantValidationUseCase {
    export type Result = Either<AbstractError, void>;
}
```

### 7. Acoplamento com `@sap/cds.Request`

```typescript
// ❌ ERRADO — Request do CAP no contrato
import type { Request } from '@sap/cds';

export namespace BeforeReadUseCase {
    export type Params = { request: Request };
}
```

```typescript
// ✅ CORRETO — wrapper local
export namespace BeforeReadUseCase {
    export type Params = {
        request: BeforeReadUseCase.MutableRequest;
        userId: string;
    };
    export type MutableRequest = {
        query: { SELECT: { where?: unknown[] } };
    };
}
```

### 8. Sufixo `-use-case` no arquivo

```
❌  src/domain/use-cases/actions/save-theme-use-case.ts
✅  src/domain/use-cases/actions/save-theme.ts
```

A pasta `use-cases/` já é o contexto.

### 9. Use case importando repository ou adapter

```typescript
// ❌ ERRADO — domain do use case importa repository
import type { PriceListRepository } from '@/domain/repositories/price-lists.js';

export namespace SavePriceListUseCase {
    export type Params = { /* ... */ };
}
```

```typescript
// ✅ CORRETO — use case declara apenas o que entra/sai; quem injeta repo é o impl
export namespace SavePriceListUseCase {
    export type Params = { /* ... */ };
    export type Result = Either<AbstractError, void>;
}
```

> O contrato do use case é a "fronteira de comportamento" — ele declara o que **a aplicação** espera receber e devolver, não como faz isso. Os repositórios são detalhe de implementação.

### 10. Padrão A do Suzano — namespace no model em vez do use case

```typescript
// ❌ ERRADO (Suzano padrão A) — namespace SaveFupAnswer mora no model
// src/domain/models/actions/save-fup-answer.ts
export namespace SaveFupAnswer {
    export type UseCaseParams = { /* ... */ };
    export type UseCaseResult = Either<AbstractError, void>;
}

// src/domain/use-cases/actions/save-fup-answer.ts
export interface SaveFupAnswerUseCase {
    execute(params: SaveFupAnswer.UseCaseParams): Promise<SaveFupAnswer.UseCaseResult>;
}
```

```typescript
// ✅ CORRETO — namespace co-localizado com a interface do use case
// src/domain/use-cases/actions/save-fup-answer.ts
export namespace SaveFupAnswerUseCase {
    export type Params = { /* ... */ };
    export type Result = Either<AbstractError, void>;
}

export interface SaveFupAnswerUseCase {
    execute(params: SaveFupAnswerUseCase.Params): Promise<SaveFupAnswerUseCase.Result>;
}
```

> O contrato (Params/Result) pertence ao **use case**, não ao model. Models declaram apenas `XxxProps` + lógica de negócio.
