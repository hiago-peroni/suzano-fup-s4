# Models

Models são o **coração da domain layer**: classes ricas que encapsulam um conceito de negócio com seu próprio shape (`XxxProps`), factory canônica (`static with(props)`), getters tipados e — o ponto mais importante — **regras de domínio implementadas como métodos**. Todo invariante, validação, formatação ou transição de estado vive no model. O use case orquestra; o model decide.

> **Princípio:** se a regra é sobre **o que** é válido para uma entidade (forma, conteúdo, transição de estado, derivação de campo), pertence ao model. Se é sobre **quando** chamar a regra (sequência, autorização, persistência), pertence ao use case.

## Estrutura de pasta

A subdivisão por contexto é livre — agrupe pelo "namespace lógico" da origem do dado: `db/` (tabelas próprias), `sap/` ou `s4/` (raw SAP), `cpi/`, `uc4/`, `actions/` (payloads de action). Models de raiz (entidades transversais sem contexto específico) ficam direto em `models/`.

```
src/domain/models/
├── db/                                 → models de entidades próprias
│   ├── price-list.ts                   → PriceListProps + class PriceListModel
│   └── tenant.ts
├── sap/                                → models de raw SAP S/4 (mantém nomenclatura PascalCase do payload)
│   └── company.ts                      → CompanyProps (PascalCase) + class CompanyModel
├── cpi/                                → opcional — payloads CPI
└── action-result.ts                    → entidade transversal de raiz
```

## Shape canônico — model rico com `with(props)`

A combinação obrigatória é:

1. **`XxxProps`** — `type` exportado descrevendo as propriedades (uma source of truth para a forma).
2. **`class XxxModel`** com `private constructor(private props: XxxProps)`.
3. **`static with(props): XxxModel`** — factory canônica.
4. **Getters explícitos** para cada propriedade exposta.
5. **Métodos de domínio** — validações, derivações, transições.
6. **Métodos de serialização** — `toObject()`, `toRow()`, `toJSON()`, `toCreationObject()`, `toUpdateObject()`, `forCreate()`, `forUpdate()` — sempre **instance methods**, com nome semântico que descreve a saída. Ver seção [Serialização](#serialização--instance-methods-não-factories).

```typescript
// src/domain/models/db/price-list.ts
import type { ValidationResult } from '@/domain/models/common/validation-result.js';

export type PriceListProps = {
    id: string;
    tenantId: string;
    name: string;
    description?: string;
    status: 'DRAFT' | 'ACTIVE' | 'ARCHIVED';
    validItemCount: number;
    createdAt: Date;
};

export class PriceListModel {
    private constructor(private readonly props: PriceListProps) {}

    public static with(props: PriceListProps): PriceListModel {
        return new PriceListModel(props);
    }

    public get id(): string {
        return this.props.id;
    }

    public get tenantId(): string {
        return this.props.tenantId;
    }

    public get name(): string {
        return this.props.name;
    }

    public get status(): PriceListProps['status'] {
        return this.props.status;
    }

    public get validItemCount(): number {
        return this.props.validItemCount;
    }

    public isActive(): boolean {
        return this.props.status === 'ACTIVE';
    }

    public canBeArchived(): boolean {
        return this.props.status === 'ACTIVE' && this.props.validItemCount > 0;
    }

    public validate(): ValidationResult {
        const errorMessages: string[] = [];
        if (!this.props.name || this.props.name.trim().length === 0) {
            errorMessages.push('priceList.name.required');
        }
        if (this.props.name && this.props.name.length > 100) {
            errorMessages.push('priceList.name.tooLong');
        }
        return { hasError: errorMessages.length > 0, errorMessages };
    }

    public decrementValidItemCount(): PriceListModel {
        if (this.props.validItemCount <= 0) {
            throw new Error('priceList.cannotDecrementBelowZero');
        }
        return PriceListModel.with({
            ...this.props,
            validItemCount: this.props.validItemCount - 1,
        });
    }

    public toObject(): PriceListProps {
        return { ...this.props };
    }
}
```

> O `private constructor` força o uso da factory `with` — clientes nunca fazem `new PriceListModel(...)`. O `private readonly props` garante imutabilidade conceitual; transições retornam **novo** model em vez de mutar (`decrementValidItemCount` acima).

## Tipo `ValidationResult` — local ao model, não compartilhado

`{ hasError: boolean; errorMessages: string[] }` é o shape canônico usado por `validate()`. Declarar o tipo **localmente** no arquivo do model que o usa — ou importar de um único helper compartilhado em `domain/models/common/validation-result.ts` se múltiplos models usarem. **Não criar pasta `domain/validators/`** (proibida — ver [README §"Pastas proibidas"](../README.md#anti-padrão-crítico--pastas-proibidas)).

```typescript
// src/domain/models/common/validation-result.ts (opcional — só se múltiplos models compartilharem)
export type ValidationResult = {
    hasError: boolean;
    errorMessages: string[];
};
```

Quando uma única entidade tem o tipo, declarar inline no próprio arquivo do model (sem o helper):

```typescript
// src/domain/models/db/discrepancy-report.ts
type ValidationResult = {
    hasError: boolean;
    errorMessages: string[];
};

export class DiscrepancyReportModel {
    public validate(): ValidationResult { /* ... */ }
}
```

## Múltiplas origens de dados — overloads de `with`, não factories alternativas

Quando o model precisa ser construído a partir de origens distintas (raw da API externa, row do DB, payload bruto), criar **métodos auxiliares de transformação** que normalizam o input e chamam `with` — nunca multiplicar nomes de factory (`from`, `fromRow`, `basic`, `forCreate`).

```typescript
// src/domain/models/db/data-load.ts
export type DataLoadProps = {
    id: string;
    tenantId: string;
    totalChunks: number;
    status: 'PENDING' | 'PROCESSING' | 'COMPLETED' | 'COMPLETED_WITH_ERRORS';
};

export class DataLoadModel {
    private constructor(private readonly props: DataLoadProps) {}

    public static with(props: DataLoadProps): DataLoadModel {
        return new DataLoadModel(props);
    }

    public static withFromDbRow(row: DataLoadDbRow): DataLoadModel {
        return DataLoadModel.with({
            id: row.id,
            tenantId: row.tenant_id,
            totalChunks: row.total_chunks ?? 0,
            status: row.status,
        });
    }

    public static withFromStaging(staging: DataLoadStagingPayload): DataLoadModel {
        return DataLoadModel.with({
            id: staging.dataLoadId,
            tenantId: staging.tenantId,
            totalChunks: staging.chunkCount,
            status: 'PENDING',
        });
    }
}

type DataLoadDbRow = {
    id: string;
    tenant_id: string;
    total_chunks: number | null;
    status: 'PENDING' | 'PROCESSING' | 'COMPLETED' | 'COMPLETED_WITH_ERRORS';
};

type DataLoadStagingPayload = {
    dataLoadId: string;
    tenantId: string;
    chunkCount: number;
};
```

> O prefixo `withFrom*` mantém a factory canônica `with` como porta única — apenas adiciona **transformações** explícitas para cada origem. Continua sendo "construir o model **com** algo", agora com adjetivo de fonte.

## Composição em vez de "God Model"

Models que crescem além de ~300 linhas concentram responsabilidades demais — a famosa "God Model" do LE44 (`loss-provision-base.ts` com 900+ linhas). Mitigações canônicas, em ordem de preferência:

1. **Helpers privados** — extrair métodos de validação intermediários para `private` no próprio model.
2. **Models auxiliares** — extrair sub-conceitos para models próprios (`LossProvisionItemModel`, `LossProvisionPeriodModel`).
3. **Composição via use case** — o use case orquestra múltiplos models em vez de pedir tudo ao model raiz.

> **Nunca** mover **lógica de regra de negócio** para `domain/services/` ou `domain/validators/` para "esvaziar o model". `validators/` é proibida; `services/` aceita só **contratos de helper de pipeline** (etapa nomeada consumida por 2+ use cases) — não é lugar para regras do model. A lógica fica no domain (em outro model) ou na orquestração (use case).

## Nomenclatura de getters — espelha a fonte

Quando o model representa um payload externo (S/4, CPI), os getters mantêm a nomenclatura da fonte (PascalCase do SAP). Quando representa dado interno (DB próprio), camelCase. A regra é: **getter espelha o nome do campo no `Props`**, e o `Props` espelha a fonte.

```typescript
// src/domain/models/sap/company.ts (raw SAP — PascalCase)
export type CompanyProps = {
    CostCenter: string;
    CompanyCode: string;
    CompanyName: string;
};

export class CompanyModel {
    private constructor(private readonly props: CompanyProps) {}

    public static with(props: CompanyProps): CompanyModel {
        return new CompanyModel(props);
    }

    public get CostCenter(): string {
        return this.props.CostCenter;
    }

    public get CompanyCode(): string {
        return this.props.CompanyCode;
    }
}
```

```typescript
// src/domain/models/db/price-list.ts (DB próprio — camelCase)
export type PriceListProps = {
    id: string;
    tenantId: string;
    validItemCount: number;
};

export class PriceListModel {
    public get id(): string { return this.props.id; }
    public get tenantId(): string { return this.props.tenantId; }
    public get validItemCount(): number { return this.props.validItemCount; }
}
```

## Serialização — instance methods, não factories

Quando o model precisa cruzar fronteira (controller → response, repository → INSERT/UPDATE, payload de draft do CAP), serializar via método de **instância** explícito. **Diferente da factory canônica `with`, métodos de serialização podem (e devem) ter nome semântico que descreve a saída.** A distinção crucial é:

| Direção | Tipo | Nome canônico |
|---|---|---|
| **input → model** (construção) | `static` | **Apenas `with`** ou `withFrom*` |
| **model → output** (serialização) | `public` (instance) | Livre, com semântica explícita |

Nomes de serialização legítimos:

| Método | Quando | Retorno |
|---|---|---|
| `toObject()` | Serialização canônica para controller/response | `XxxProps` (cópia rasa) |
| `toJSON()` | Serialização para `JSON.stringify` ou response HTTP | `XxxProps` ou shape público |
| `toRow()` | Para repository (INSERT/UPDATE genérico) | shape do DB (snake_case) |
| `toDraftObject()` | Para draft CAP (subset de campos modificáveis) | subset documentado |
| `toCreationObject()` / `forCreate()` | Payload específico para action/repo de **criação** | shape específico (sem `id`, sem timestamps gerados pelo DB) |
| `toUpdateObject()` / `forUpdate()` | Payload específico para action/repo de **update** | `Partial<XxxProps>` ou shape de patch |

```typescript
// src/domain/models/db/price-list.ts
public toObject(): PriceListProps {
    return { ...this.props };
}

public toRow(): PriceListRepository.DbRow {
    return {
        id: this.props.id,
        tenant_id: this.props.tenantId,
        name: this.props.name,
        status: this.props.status,
    };
}

public toCreationObject(): Omit<PriceListProps, 'id' | 'createdAt'> {
    return {
        tenantId: this.props.tenantId,
        name: this.props.name,
        description: this.props.description,
        status: 'DRAFT',
        validItemCount: 0,
    };
}

public forUpdate(): Partial<Pick<PriceListProps, 'name' | 'description' | 'status'>> {
    return {
        name: this.props.name,
        description: this.props.description,
        status: this.props.status,
    };
}
```

> O shape de `toRow()` espelha o `XxxRepository.DbRow` declarado em `domain/repositories/<entidade>.ts`. Se houver duplicação, importar o tipo: `public toRow(): PriceListRepository.DbRow { ... }`.

### O que **não** é serialização legítima

Os mesmos nomes (`forCreate`, `forUpdate`, etc.) são **proibidos como `static`** — porque aí viram factory alternativa. Ver anti-padrão #2 abaixo.

```typescript
// ❌ ERRADO — forCreate como factory (anti-padrão RVE)
public static forCreate(props: PriceListForCreateProps): PriceListModel { /* ... */ }

// ✅ CERTO — forCreate como instance method de serialização
public forCreate(): Omit<PriceListProps, 'id' | 'createdAt'> { /* ... */ }
```

## Imports permitidos no model

| O que pode importar | Exemplo |
|---|---|
| Outros models | `import { LossProvisionItemModel } from '@/domain/models/db/loss-provision-item.js';` |
| Erros do domain | `import { BadRequestError } from '@/domain/errors/index.js';` (raro — preferir retornar `ValidationResult` que o use case converte em erro) |
| Tipo do repository (apenas o `DbRow`) | `import type { PriceListRepository } from '@/domain/repositories/price-lists.js';` |

| O que **não** pode importar | Motivo |
|---|---|
| `@sap/cds`, `@models/*`, `@cds-models/*` | Acoplamento com framework |
| `application/`, `infrastructure/`, `presentation/`, `main/` | Quebra de isolamento |
| `@sweet-monads/either` | Models não retornam `Either` — quem retorna é o use case |

## Regras

1. **`private constructor` + `static with(props)`** — clientes nunca usam `new XxxModel()`.
2. **`private readonly props: XxxProps`** — imutabilidade conceitual; transições retornam novo model.
3. **Getter para cada propriedade exposta** — sem expor `props` direto.
4. **Lógica de domínio como método** (`validate()`, `canBe*()`, `is*()`, `*Computed`).
5. **Sem I/O no model** — o model não chama repositório, adapter, ou função pura externa que faça I/O.
6. **Sem `Either` no model** — `validate()` retorna `ValidationResult`; quem traduz para `Either<AbstractError, T>` é o use case.
7. **`with` é o único nome de factory pública** — variantes têm prefixo `withFrom*`.
8. **Models são imutáveis por convenção** — não há setters; transições retornam nova instância.
9. **Sem dependência de framework** — zero `@sap/cds`, zero `@models/*`.

## Anti-padrões

### 1. Model anêmico (type alias sem classe)

```typescript
// ❌ ERRADO — apenas type, sem comportamento
export type PurchaseOrderItem = {
    poNumber: string;
    poItem: number;
    quantity: number;
};
```

```typescript
// ✅ CORRETO — model rico
export type PurchaseOrderItemProps = {
    poNumber: string;
    poItem: number;
    quantity: number;
};

export class PurchaseOrderItemModel {
    private constructor(private readonly props: PurchaseOrderItemProps) {}

    public static with(props: PurchaseOrderItemProps): PurchaseOrderItemModel {
        return new PurchaseOrderItemModel(props);
    }

    public get poNumber(): string { return this.props.poNumber; }
    public get poItem(): number { return this.props.poItem; }
    public get quantity(): number { return this.props.quantity; }

    public hasQuantity(): boolean {
        return this.props.quantity > 0;
    }
}
```

### 2. Múltiplos nomes de factory `static`

A regra vale só para **factories** (input → model). Métodos de instância de serialização (model → output) podem usar livremente os mesmos nomes — `forCreate()`, `forUpdate()`, `toCreationObject()` como `public` é legítimo (ver seção de Serialização).

```typescript
// ❌ ERRADO — multiplicação de nomes de factory (input → model)
public static from(props): PriceListModel { /* ... */ }
public static fromDbRow(row): PriceListModel { /* ... */ }
public static fromStaging(staging): PriceListModel { /* ... */ }
public static basic(props): PriceListModel { /* ... */ }
public static forCreate(props: PriceListForCreateProps): PriceListModel { /* ... */ }   // anti-padrão RVE
public static forUpdate(props: PriceListForUpdateProps): PriceListModel { /* ... */ }
```

```typescript
// ✅ CORRETO — porta única de construção + transformadores explícitos com prefixo `withFrom*`
public static with(props): PriceListModel { /* ... */ }
public static withFromDbRow(row): PriceListModel { /* normaliza row e chama with() */ }
public static withFromStaging(staging): PriceListModel { /* normaliza staging e chama with() */ }

// ✅ CORRETO — serialização com nome semântico fica como instance method
public forCreate(): PriceListCreationPayload { /* ... */ }
public forUpdate(): Partial<PriceListProps> { /* ... */ }
```

### 3. Public constructor

```typescript
// ❌ ERRADO — clientes podem fazer new PriceListModel({...})
export class PriceListModel {
    constructor(public props: PriceListProps) {}
}
```

```typescript
// ✅ CORRETO — força uso da factory
export class PriceListModel {
    private constructor(private readonly props: PriceListProps) {}
    public static with(props: PriceListProps): PriceListModel {
        return new PriceListModel(props);
    }
}
```

### 4. Mutação direta em vez de retornar novo model

```typescript
// ❌ ERRADO — muta props in-place
public decrementValidItemCount(): void {
    this.props.validItemCount -= 1;
}
```

```typescript
// ✅ CORRETO — retorna novo model
public decrementValidItemCount(): PriceListModel {
    return PriceListModel.with({
        ...this.props,
        validItemCount: this.props.validItemCount - 1,
    });
}
```

### 5. Chamada de I/O dentro do model

```typescript
// ❌ ERRADO — model chama repository
public async refreshFromDb(): Promise<PriceListModel> {
    const row = await this.repo.findById(this.props.id);
    return PriceListModel.with(row);
}
```

```typescript
// ✅ CORRETO — model é puro; quem orquestra I/O é o use case
// No use case:
const refreshed = await this.priceListRepo.findById(model.id);
```

### 6. Acoplamento com `@sap/cds` ou `@models/*`

```typescript
// ❌ ERRADO — importa CDS typer
import type { PriceLists } from '@cds-models/db/models.js';

export type PriceListProps = PriceLists;
```

```typescript
// ✅ CORRETO — Props declarado independente do schema CDS
export type PriceListProps = {
    id: string;
    tenantId: string;
    name: string;
    status: 'DRAFT' | 'ACTIVE' | 'ARCHIVED';
};
```

### 7. Validação fora do model (em pasta `validators/`)

```typescript
// ❌ ERRADO — domain/validators/price-list-validator.ts
export class PriceListValidator {
    static validateName(name: string): ValidationResult { /* ... */ }
}
```

```typescript
// ✅ CORRETO — método do próprio model
export class PriceListModel {
    public validate(): ValidationResult { /* ... */ }
}
```

### 8. Getter com lógica complexa em vez de método

```typescript
// ❌ ERRADO — getter que faz cálculo complexo
public get summary(): PriceListSummary {
    const total = this.calculateTotal();
    const items = this.aggregateItems();
    return { total, items, /* ... */ };
}
```

```typescript
// ✅ CORRETO — método explícito sinaliza custo computacional
public computeSummary(): PriceListSummary {
    const total = this.calculateTotal();
    const items = this.aggregateItems();
    return { total, items, /* ... */ };
}
```

> **Regra de bolso para getters vs métodos:** se a operação é uma leitura direta de `props`, é getter. Se faz cálculo, agregação, ou tem custo > O(1), é método (`compute*`, `calculate*`, `derive*`).
