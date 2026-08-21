# Adapters

`domain/adapters/` declara os contratos de integração com tudo que está **fora da fronteira da aplicação**: APIs externas (S/4, REST, AWS), parsers de arquivo (Excel, CSV, JSON) e o `UnitOfWork` que coordena transações em batch. Espelha exatamente a estrutura de `infrastructure/adapters/` — para cada `XxxApiImpl`/`XxxParser`/`CdsUnitOfWork` na infra existe um contrato (interface + namespace) aqui.

> **Diferença essencial entre adapter e repository:** o **repository** acessa o CDS local da própria aplicação (`cds.ql.*` sobre `db.models.*` e `db.replication.*`); o **adapter** atravessa a fronteira da aplicação em direção a sistemas terceiros, arquivos no filesystem ou batches de transação. Quando precisar persistir em tabela própria do projeto via CDS → repository. Para qualquer outra coisa I/O → adapter.

## Estrutura canônica

```
src/domain/adapters/
├── external-api/
│   ├── http/
│   │   └── sap-http-client.ts             → SapHttpClient (interface genérica)
│   └── <sistema>/
│       ├── <recurso>.ts                   → XxxApi (interface + namespace)
│       └── <recurso>.ts
├── parsers/
│   └── <formato>-parser.ts                → XxxParser (interface + namespace)
└── unit-of-work.ts                        → UnitOfWork (interface + namespace)
```

| Subpasta | Conteúdo | Naming canônico |
|---|---|---|
| `adapters/external-api/<sistema>/` | Contratos de domínios de API externa (S/4, AWS, carrier, email) | `XxxApi` (interface) |
| `adapters/external-api/http/` | Contrato HTTP genérico compartilhado | `SapHttpClient` (interface) |
| `adapters/parsers/` | Contratos de parsers de arquivo | `<Formato>Parser` (interface) |
| `adapters/unit-of-work.ts` | Contrato de coordenação transacional batch | `UnitOfWork` (interface) |

> Não existe pasta `adapters/<dominio>/` no domain — apenas `adapters/external-api/<sistema>/`. A subdivisão por contexto pertence à infrastructure (`infrastructure/adapters/external-api/<dominio>/`); o domain mantém a mesma divisão.

## Shape canônico — `XxxApi`

```typescript
// src/domain/adapters/external-api/company/company.ts
import type { CompanyModel } from '@/domain/models/sap/company.js';

export namespace CompanyApi {
    export type FindByCostCentersParams = {
        costCenters: string[];
    };
    export type FindByCostCentersResult = CompanyModel[];

    export type FindByCompanyCodeParams = {
        companyCode: string;
    };
    export type FindByCompanyCodeResult = CompanyModel | null;
}

export interface CompanyApi {
    findByCostCenters(params: CompanyApi.FindByCostCentersParams): Promise<CompanyApi.FindByCostCentersResult>;
    findByCompanyCode(params: CompanyApi.FindByCompanyCodeParams): Promise<CompanyApi.FindByCompanyCodeResult>;
}
```

Pontos canônicos:

- **Cada operação tem `XxxParams` + `XxxResult` no namespace** — espelha o padrão de use cases mas sem `Either`.
- **Adapter retorna model** quando há entidade serializável — `CompanyApi.FindByCostCentersResult = CompanyModel[]`. Mesma regra do repository (a infrastructure serializa via `XxxModel.with(raw)` antes de devolver).
- **Adapter propaga exceção** — sem `Either`. Quem captura é o use case via `BaseUseCaseImpl.handleError`. Mesma regra dos repositórios e da [infrastructure-layer/adapters](../../infrastructure-layer/adapters/README.md#tratamento-de-erro).
- **Sem `@sap/cds`** — wrappers locais quando precisar do tipo de `Service`/`destination`.

## Shape canônico — adapter HTTP genérico

`SapHttpClient` é o adapter HTTP compartilhado entre múltiplos `XxxApi` que precisam fazer POST/PATCH/DELETE em destinations SAP (Cloud SDK). Vive em `external-api/http/` porque é compartilhado, não em um sistema específico.

```typescript
// src/domain/adapters/external-api/http/sap-http-client.ts
export namespace SapHttpClient {
    export type PostParams = {
        destinationName: string;
        url: string;
        path: string;
        data: unknown;
        headers?: Record<string, string>;
    };

    export type FetchCsrfParams = {
        destinationName: string;
        url: string;
        path: string;
    };

    export type FetchCsrfResult = {
        token: string;
        cookies: string;
    };
}

export interface SapHttpClient {
    post<T>(params: SapHttpClient.PostParams): Promise<T>;
    fetchCsrfToken(params: SapHttpClient.FetchCsrfParams): Promise<SapHttpClient.FetchCsrfResult>;
}
```

> O `SapHttpClient` não retorna model — é "transporte" puro. A serialização raw → model fica no `XxxApiImpl` específico que o consome. Ver few-shot completo em [infrastructure-layer/adapters/external-apis.md → Few-shot 3](../../infrastructure-layer/adapters/external-apis.md#few-shot-3--sapcloudsdkhttpclient-adapter-http-genérico).

## Shape canônico — parser

```typescript
// src/domain/adapters/parsers/excel-parser.ts
export namespace ExcelParser {
    export type ParseParams = {
        buffer: Buffer;
        sheetName?: string;
    };

    export type ParseResult<T> = {
        rows: T[];
        errors: ExcelParser.ParseError[];
    };

    export type ParseError = {
        row: number;
        column: string;
        message: string;
    };
}

export interface ExcelParser {
    parse<T>(params: ExcelParser.ParseParams, schema: ExcelParser.Schema<T>): ExcelParser.ParseResult<T>;
}

export namespace ExcelParser {
    export type Schema<T> = {
        columns: ReadonlyArray<{
            key: keyof T;
            header: string;
            type: 'string' | 'number' | 'date' | 'boolean';
            required?: boolean;
        }>;
    };
}
```

Pontos canônicos:

- **`Buffer`** é tipo nativo do Node.js — aceitável no domain (não é framework de aplicação).
- **`ParseResult<T>` é genérico** — o consumidor passa o tipo da linha esperada via `Schema<T>`.
- **`ParseError`** é tipo do namespace — não criar `parser-errors.ts` separado.

## Shape canônico — `UnitOfWork`

```typescript
// src/domain/adapters/unit-of-work.ts
export namespace UnitOfWork {
    export type Intent =
        | { kind: 'insert'; entity: string; rows: ReadonlyArray<Record<string, unknown>> }
        | { kind: 'update'; entity: string; filter: Record<string, unknown>; patch: Record<string, unknown> }
        | { kind: 'delete'; entity: string; filter: Record<string, unknown> };

    export type CommitOptions = {
        useTransaction?: boolean;
    };
}

export interface UnitOfWork {
    registerInsert(entity: string, rows: ReadonlyArray<Record<string, unknown>>): void;
    registerUpdate(entity: string, filter: Record<string, unknown>, patch: Record<string, unknown>): void;
    registerDelete(entity: string, filter: Record<string, unknown>): void;
    pendingCount(): number;
    commit(options?: UnitOfWork.CommitOptions): Promise<void>;
    clear(): void;
}
```

Pontos canônicos:

- **`Intent` é um discriminated union** — o consumidor sabe qual operação cada intent representa via `kind`.
- **`registerXxx` retorna `void`** — apenas registra; o I/O acontece em `commit()`.
- **`commit` é o único `Promise`** — o batch acontece de uma vez.
- **`Record<string, unknown>` para `rows`/`filter`/`patch`** — o UoW é genérico; a tipagem específica fica no repositório que registra (`PartNumberRepository.InsertRow`, etc.).

## Imports permitidos

| O que pode importar | Exemplo |
|---|---|
| Models | `import type { CompanyModel } from '@/domain/models/sap/company.js';` |
| Tipos nativos do Node | `Buffer`, `Readable` (sem `import` — globais) |

| O que **não** pode importar | Motivo |
|---|---|
| `@sap/cds`, `@sap-cloud-sdk/*` | Acoplamento com framework — wrapper local |
| `application/`, `infrastructure/`, `presentation/`, `main/` | Quebra de isolamento |
| Outros adapters (raro — se realmente precisar, repensar a separação) | Adapters são contratos planos |

## Sem `Either` em adapters

Diferente dos use cases, **adapters não retornam `Either`**. O contrato com a application layer é o mesmo dos repositórios:

- **Caminho feliz** → retorna o valor tipado (model, array, primitivo).
- **Não encontrado** → retorna `null` ou `[]` (sem lançar `NotFoundError`).
- **Erro de I/O** → propaga exceção. Captura via `BaseUseCaseImpl.handleError`.

```typescript
// ✅ CORRETO — adapter retorna model ou null
export interface CompanyApi {
    findByCompanyCode(params: CompanyApi.FindByCompanyCodeParams): Promise<CompanyModel | null>;
}

// ❌ ERRADO — adapter retorna Either
export interface CompanyApi {
    findByCompanyCode(params: CompanyApi.FindByCompanyCodeParams): Promise<Either<AbstractError, CompanyModel>>;
}
```

> A regra é simétrica entre repositories e adapters: **infrastructure propaga exceção; application captura via `Either`.** Ver [infrastructure-layer/adapters/README.md → Tratamento de erro](../../infrastructure-layer/adapters/README.md#tratamento-de-erro).

## Naming

| Elemento | Convenção | Exemplo |
|---|---|---|
| Pasta de sistema externo | `kebab-case-singular` | `company/`, `carrier/`, `email/` |
| Arquivo de API por recurso | `kebab-case.ts` | `company.ts`, `purchase-order.ts` |
| Interface API | `XxxApi` | `CompanyApi`, `CarrierApi` |
| Interface HTTP genérico | `XxxHttpClient` | `SapHttpClient`, `RestHttpClient` |
| Arquivo parser | `<formato>-parser.ts` | `excel-parser.ts`, `csv-parser.ts`, `json-parser.ts` |
| Interface parser | `<Formato>Parser` | `ExcelParser`, `CsvParser` |
| Arquivo UoW | `unit-of-work.ts` (raiz de adapters) | — |
| Interface UoW | `UnitOfWork` | — |
| Namespace colateral | mesmo nome da interface | `namespace CompanyApi { ... }` |

## Regras

1. **Interface pura** — sem implementação no domain.
2. **`namespace XxxApi`** agrupa `Params`/`Result` por operação.
3. **Adapter retorna model** quando há entidade serializável; `null`/`[]` para ausência; primitivo para `count`/`exists`.
4. **Sem `Either`** — adapter propaga exceção (mesma regra dos repositories).
5. **Sem `@sap/cds`** — wrapper local para tipos de framework.
6. **Estrutura espelha a infrastructure layer:** `external-api/<sistema>/`, `parsers/`, `unit-of-work.ts`.
7. **Sem barrel `adapters/index.ts`** — imports diretos.
8. **`SapHttpClient` em `external-api/http/`** — compartilhado entre múltiplos `XxxApi` que usam destinations SAP.
9. **Cada API por recurso, não monolítica** — `company.ts` e `purchase-order.ts` separados, mesmo se ambos são "S/4". Pasta `<sistema>/` agrupa os recursos.

## Anti-padrões

### 1. Pasta `external-apis/` raiz (padrão LE44)

```
❌  src/domain/external-apis/s4/company.ts
✅  src/domain/adapters/external-api/company/company.ts
```

A pasta `external-apis/` raiz isolada de `adapters/` é divergência histórica do LE44. Espelhar a estrutura da infra-layer mantém coesão entre as camadas.

### 2. Adapter monolítico para o sistema inteiro

```typescript
// ❌ ERRADO — uma interface S/4 com todos os recursos
export interface S4Api {
    findCompany(params): Promise<CompanyModel | null>;
    findPurchaseOrder(params): Promise<PurchaseOrderModel | null>;
    createDelivery(params): Promise<DeliveryModel>;
    // ... 28+ métodos
}
```

```typescript
// ✅ CORRETO — uma interface por recurso
// adapters/external-api/s4/company.ts
export interface CompanyApi { findByCostCenters(params): Promise<CompanyApi.FindByCostCentersResult>; }

// adapters/external-api/s4/purchase-order.ts
export interface PurchaseOrderApi { findById(params): Promise<PurchaseOrderApi.FindByIdResult>; }

// adapters/external-api/s4/delivery.ts
export interface DeliveryApi { create(params): Promise<DeliveryApi.CreateResult>; }
```

> A subdivisão por **recurso** facilita DI no `factories/adapters/`, isola contratos, e permite injetar fakes por API individual nos testes.

### 3. Adapter retornando `Either`

```typescript
// ❌ ERRADO
export interface CompanyApi {
    findByCompanyCode(params): Promise<Either<AbstractError, CompanyModel>>;
}
```

```typescript
// ✅ CORRETO — propaga exceção, retorna null para ausência
export interface CompanyApi {
    findByCompanyCode(params): Promise<CompanyModel | null>;
}
```

### 4. `@sap/cds.Service` no contrato

```typescript
// ❌ ERRADO — Service do CAP no domain
import type { Service } from '@sap/cds';

export interface CompanyApi {
    getInstance(): Promise<Service>;
}
```

```typescript
// ✅ CORRETO — domain conhece apenas Params/Result; Service vive na infra
export interface CompanyApi {
    findByCostCenters(params: CompanyApi.FindByCostCentersParams): Promise<CompanyApi.FindByCostCentersResult>;
}
```

### 5. Pasta `adapters/<dominio>/` sem `external-api/` ou outro contexto

```
❌  src/domain/adapters/jwt-decoder.ts          (LE44 — adapter solto)
❌  src/domain/adapters/hdb-adapter.ts          (MRO — adapter solto)
✅  src/domain/adapters/external-api/auth/jwt-decoder.ts
✅  src/domain/utils/hana/hana-adapter.ts (se for utilitário interno, não API externa)
```

> Adapters precisam de contexto: ou `external-api/<sistema>/` (API externa), ou `parsers/` (parser de arquivo), ou `unit-of-work.ts` (raiz). Soltos em `adapters/` raiz são anti-padrão.

### 6. Implementação dentro do contrato

```typescript
// ❌ ERRADO — domain/adapters/external-api/company/company.ts
export interface CompanyApi { /* ... */ }

export class CompanyApiHelper {
    static normalizeCostCenter(cc: string): string { return cc.padStart(10, '0'); }
}
```

```typescript
// ✅ CORRETO — helpers vivem na infra (CompanyApiImpl) ou no model (CompanyModel)
export interface CompanyApi { /* ... */ }
// Normalização vai para CompanyModel.with(...) ou para CompanyApiImpl
```

### 7. Adapter de telemetria/cache em pasta solta

```
❌  src/domain/adapters/telemetry/telemetry-sink.ts          (MRO — solto)
❌  src/domain/adapters/object-store-adapter.ts              (MRO — solto)
```

```
✅  src/domain/adapters/external-api/telemetry/telemetry.ts  (se for serviço externo de telemetria)
✅  src/domain/utils/observability.ts                        (se for utilitário interno)
```

> O critério de inclusão é "atravessa fronteira externa?" → external-api. Caso contrário, é utilitário/serviço da infra que não precisa de contrato no domain.
