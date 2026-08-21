# Services

`domain/services/` declara contratos de **helpers granulares de use case** — interfaces que servem para extrair etapas reutilizáveis ou semanticamente coesas que múltiplos use cases consomem, sem expô-las como `action`/`function` no `index.cds`.

Um service no domain tem **exatamente o mesmo shape de um use case** (`interface XxxService { execute(params): Promise<XxxService.Result> }` + `namespace XxxService { Params, Result }`), mas **não é um use case**: não corresponde a nenhuma operação CDS, não é chamado pela presentation layer, e não recebe `request` do CAP. É consumido **internamente pelos use cases** via DI.

> **Use case vs service — quando usar cada um:** se a operação é exposta no `index.cds` como `action`/`function`/entity-event hook → **use case**. Se é uma etapa intermediária consumida por um ou mais use cases (regra repetida, decisão complexa, transformação reutilizável) → **service**.

## Estrutura

`domain/services/` é **plano** ou **agrupado por contexto** quando há múltiplos services relacionados. Um arquivo por contrato; nome do arquivo em kebab-case espelhando o conceito do service.

```
src/domain/services/
├── permission-checker.ts                   → PermissionCheckerService (helper transversal)
├── data-load/
│   ├── initialize-data-load.ts             → InitializeDataLoadService
│   ├── import-data-load.ts                 → ImportDataLoadService
│   ├── promote-data-load.ts                → PromoteDataLoadService
│   ├── finalize-data-load.ts               → FinalizeDataLoadService
│   ├── validate-data-load-keys.ts          → ValidateDataLoadKeysService
│   ├── validate-service-number.ts          → ValidateServiceNumberService
│   └── index.ts                            → barrel da pasta (reexport por contexto)
└── build-sap-payload.ts                    → BuildSapPayloadService
```

> Subpastas por contexto (ex.: `data-load/`) seguem a mesma convenção de `use-cases/entity-events/<entidade>/` — agrupam etapas relacionadas a um pipeline. Index.ts da subpasta é permitido (mesma regra dos `entity-events/<entidade>/index.ts`).

## Shape canônico — interface + namespace

```typescript
// src/domain/services/permission-checker.ts
import type { Either } from '@sweet-monads/either';

import type { AbstractError } from '@/domain/errors/index.js';

export namespace PermissionCheckerService {
    export type Params = {
        userGroups: string[];
        operation: string;
    };
    export type Result = Either<AbstractError, boolean>;
}

export interface PermissionCheckerService {
    execute(params: PermissionCheckerService.Params): Promise<PermissionCheckerService.Result>;
}
```

```typescript
// src/domain/services/data-load/initialize-data-load.ts
import type { Either } from '@sweet-monads/either';

import type { AbstractError } from '@/domain/errors/index.js';
import type { DataLoadRepository } from '@/domain/repositories/data-loads.js';

export namespace InitializeDataLoadService {
    export type Params = {
        dataLoadId: string;
    };
    export type Result = Either<AbstractError, DataLoadRepository.DbRow | null>;
}

export interface InitializeDataLoadService {
    execute(params: InitializeDataLoadService.Params): Promise<InitializeDataLoadService.Result>;
}
```

Pontos canônicos:

- **`namespace XxxService`** com `Params` e `Result` — mesma convenção dos use cases.
- **`Result = Either<AbstractError, T>`** — services retornam `Either` (igual a use case), porque participam da pipeline de orquestração que a application layer monta.
- **Método único `execute(params)`** — sem variantes (`run`, `process`, `do`); consistência total com use cases.
- **Implementação vive em `application/services/<mesma-estrutura>/`**, consumindo repositories/adapters via DI. Ver [application-layer/services/README.md](../../application-layer/services/README.md).
- **Sem barrel `services/index.ts` raiz** — só dentro de subpastas de contexto (`data-load/index.ts`).

## Quando criar um service vs deixar como helper privado de use case

| Situação | O que criar |
|---|---|
| Lógica usada por **um** use case e que cabe em método privado da `*UseCaseImpl` | `private` method na própria implementação do use case (em `application/use-cases/`). Não criar service. |
| Lógica usada por **2+ use cases** (mesma decisão, mesma transformação) | **Service no domain** com contrato + impl em `application/services/`. |
| Etapa de pipeline que envolve I/O (chamar repository/adapter) | **Service no domain** — porque a impl precisa receber dependências via DI. |
| Etapa que **só** transforma dados puros (sem I/O, sem decisão de negócio) | Método utilitário em `domain/utils/` (se for genérico) ou método do **model** (se for específico da entidade). Não criar service. |
| "Sub-use-case" que aparece no fluxo de outro use case mas tem `Params`/`Result` próprios | **Service no domain** — eleva a coesão da pipeline. Ver `mro-processing-service` (`InitializeDataLoadService`, `ImportDataLoadService`, `PromoteDataLoadService`, `FinalizeDataLoadService`) consumidos por `process-data-load-queue.ts`. |

## Imports permitidos

| O que pode importar | Exemplo |
|---|---|
| `Either` de `@sweet-monads/either` | `import type { Either } from '@sweet-monads/either';` |
| `AbstractError` do domain | `import type { AbstractError } from '@/domain/errors/index.js';` |
| Models do domain (apenas como tipos) | `import type { PriceListModel } from '@/domain/models/db/price-list.js';` |
| Tipos auxiliares de outros contratos | `import type { DataLoadRepository } from '@/domain/repositories/data-loads.js';` |
| Outros services do domain (composição via tipos) | `import type { ValidateDataLoadKeysService } from '@/domain/services/data-load/validate-data-load-keys.js';` |

| O que **não** pode importar | Motivo |
|---|---|
| `@sap/cds`, `@models/*`, `@cds-models/*` | Acoplamento com framework |
| `application/`, `infrastructure/`, `presentation/`, `main/` | Quebra de isolamento (regra de ouro do domain) |
| SDKs externos (AWS, http clients, drivers de DB) | Se precisa disso, **não é service — é adapter** |
| Implementações concretas (`*Impl`, `*Concrete`) | Domain só conhece contratos |

## Regras

1. **Interface pura.** Só `interface` + `namespace`. Implementação concreta em `domain/services/` é **proibida** (anti-padrão MRO `OciLineBuilders/kit-line-builder.ts`).
2. **Mesmo shape de use case.** `execute(params): Promise<XxxService.Result>` + `namespace { Params, Result }`. Sem variantes de método.
3. **`Result = Either<AbstractError, T>`** — services compõem com use cases na pipeline `Either`.
4. **Sem dependência de framework ou SDK** mesmo no contrato. Se a implementação precisar tocar HTTP/DB/SDK/CAP, o contrato vira **adapter** em `domain/adapters/`, não service.
5. **Sem barrel raiz** (`services/index.ts`). Permitido apenas em subpastas de contexto (`data-load/index.ts`).
6. **Naming `XxxService`** — sufixo obrigatório. `XxxHelper`, `XxxManager`, `XxxHandler` são **proibidos**.
7. **Arquivo em kebab-case singular** descrevendo a operação (`permission-checker.ts`, `validate-service-number.ts`) — sem sufixo `-service` no arquivo (a pasta já é o contexto).

## Anti-padrões

### 1. Implementação concreta em `domain/services/`

```typescript
// ❌ ERRADO — domain/services/oci-line-builders/kit-line-builder.ts (anti-padrão MRO)
export class KitLineBuilder implements OciLineBuilder {
    public buildOciLines(params: OciLineBuilder.Params): OciLineBuilder.Result {
        // ... lógica concreta no domain
    }
}
```

```typescript
// ✅ CORRETO — só contrato no domain; impl em application/services/
// src/domain/services/oci-line-builder.ts
export interface OciLineBuilder {
    execute(params: OciLineBuilder.Params): Promise<OciLineBuilder.Result>;
}

// src/application/services/oci-line-builders/kit-line-builder.ts
export class KitLineBuilderImpl implements OciLineBuilder { /* ... */ }
```

### 2. Service que toca rede/disco/SDK (deveria ser adapter)

```typescript
// ❌ ERRADO — contrato implica que a impl faz HTTP
export interface SapHoursCyclesApi {
    fetchCycle(cycleId: string): Promise<CycleData>;
}

// Localizado em domain/services/sap-hours-cycles-api.ts
```

```typescript
// ✅ CORRETO — vai para domain/adapters/external-api/sap/hours-cycles.ts
// services/ é só para helpers internos de use case
```

> **Regra prática para distinguir service × adapter:** se a implementação consome **apenas outros contratos do domain** (repositories, outros services, models), é **service**. Se toca **rede, disco, SDK ou framework**, é **adapter**.

### 3. Sufixo errado no nome da interface

```typescript
// ❌ ERRADO — nomes ambíguos
export interface PermissionCheckerHelper { /* ... */ }
export interface DataLoadManager { /* ... */ }
export interface PayloadHandler { /* ... */ }
```

```typescript
// ✅ CORRETO — sufixo Service
export interface PermissionCheckerService { /* ... */ }
export interface InitializeDataLoadService { /* ... */ }
export interface BuildSapPayloadService { /* ... */ }
```

### 4. Service para lógica de **um único** use case

```typescript
// ❌ ERRADO — service usado em apenas um use case que poderia ser método privado
export interface NormalizeCheckoutPayloadService {
    execute(params: { items: CartItem[] }): Promise<Either<AbstractError, NormalizedItems>>;
}
// Consumido só por CheckoutUseCaseImpl
```

```typescript
// ✅ CORRETO — método privado do próprio use case
export class CheckoutUseCaseImpl extends BaseUseCaseImpl implements CheckoutUseCase {
    public async execute(params: CheckoutUseCase.Params): Promise<CheckoutUseCase.Result> {
        const normalized = this.normalizePayload(params.items);
        // ...
    }

    private normalizePayload(items: CartItem[]): NormalizedItems { /* ... */ }
}
```

> Promover a service só quando há reuso real (2+ use cases) ou ganho semântico evidente (etapa nomeada do pipeline).

### 5. Service sem `execute` (variantes de método)

```typescript
// ❌ ERRADO — métodos com nome arbitrário
export interface PermissionCheckerService {
    check(userGroups: string[], operation: string): Promise<boolean>;
    checkPermission(userGroups: string[], operation: string): Promise<boolean>;
}
```

```typescript
// ✅ CORRETO — método único execute, mesma convenção de use case
export interface PermissionCheckerService {
    execute(params: PermissionCheckerService.Params): Promise<PermissionCheckerService.Result>;
}
```

### 6. Service retornando boolean/valor primitivo direto (sem `Either`)

```typescript
// ❌ ERRADO — não compõe com pipeline Either dos use cases
export interface PermissionCheckerService {
    execute(params: PermissionCheckerService.Params): Promise<boolean>;
}
```

```typescript
// ✅ CORRETO — Either<AbstractError, boolean> permite propagar erros de I/O da impl
export namespace PermissionCheckerService {
    export type Result = Either<AbstractError, boolean>;
}

export interface PermissionCheckerService {
    execute(params: PermissionCheckerService.Params): Promise<PermissionCheckerService.Result>;
}
```

### 7. Sufixo `-service` no nome do arquivo

```
❌  src/domain/services/permission-checker-service.ts
✅  src/domain/services/permission-checker.ts
```

A pasta `services/` já é o contexto.

### 8. Constante/enum/tipo solto em `services/<nome>.ts`

```typescript
// ❌ ERRADO — service file com export que não é interface/namespace do contrato
export const PERMISSION_OPERATIONS = ['Read', 'Write', 'Delete'] as const;
export interface PermissionCheckerService { /* ... */ }
```

```typescript
// ✅ CORRETO — enums/constantes ficam no namespace do contrato ou no model relacionado
export namespace PermissionCheckerService {
    export type Operation = 'Read' | 'Write' | 'Delete';
}

export interface PermissionCheckerService { /* ... */ }
```
