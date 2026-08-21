# Repositories

`domain/repositories/` declara os contratos de persistência: para cada entidade do serviço existe **uma** `interface XxxRepository` que descreve as operações suportadas (find, save, update, delete, count, bulk*) e um `namespace XxxRepository` que reúne os tipos auxiliares (`DbRow`, `Filters`, `Patch`).

Esses contratos são consumidos pela application layer (use cases) via DI e implementados pela infrastructure layer (`infrastructure/repositories/<namespace>/<entidade>.ts`). O domain **não conhece** `cds.ql`, `cds.run`, ou qualquer detalhe de I/O — só descreve o que precisa.

## Estrutura

`domain/repositories/` é **plano** — um arquivo por entidade, sem subpastas por contexto. A subdivisão por `models/` vs `replication/` existe **apenas na infrastructure layer** (`infrastructure/repositories/models/` e `infrastructure/repositories/replication/`); no domain, o contrato é único, independente do namespace CDS.

```
src/domain/repositories/
├── price-lists.ts                 → PriceListRepository (db.models.PriceLists)
├── tenants.ts                     → TenantRepository (db.models.Tenants)
├── part-numbers.ts                → PartNumberRepository (db.models.PartNumbers + UoW)
├── plants.ts                      → PlantRepository (db.replication.Plants — só leitura)
└── data-loads.ts                  → DataLoadRepository (db.models.DataLoads)
```

> Mesmo entidades replicadas (somente leitura) ficam em `repositories/` — não há subpasta `domain/repositories/replication/`. A natureza "leitura-only" se manifesta na **interface** (sem `save`/`update`/`delete`), não na localização.

## Shape canônico — interface + namespace

```typescript
// src/domain/repositories/price-lists.ts
import type { PriceListModel } from '@/domain/models/db/price-list.js';

export namespace PriceListRepository {
    export type DbRow = {
        tenant_id: string;
        id: string;
        name: string;
        description: string | null;
        status: 'DRAFT' | 'ACTIVE' | 'ARCHIVED';
        valid_item_count: number;
        created_at: string;
    };
    export type Patch = Partial<Omit<DbRow, 'tenant_id' | 'id'>>;
    export type Filters = {
        status?: DbRow['status'];
        createdAfter?: string;
    };
}

export interface PriceListRepository {
    findById(tenantId: string, id: string): Promise<PriceListModel | null>;
    findAllByTenant(tenantId: string): Promise<PriceListModel[]>;
    findByFilters(tenantId: string, filters: PriceListRepository.Filters): Promise<PriceListModel[]>;
    save(model: PriceListModel): Promise<void>;
    update(tenantId: string, id: string, patch: PriceListRepository.Patch): Promise<void>;
    delete(tenantId: string, id: string): Promise<void>;
    count(tenantId: string): Promise<number>;
    exists(tenantId: string, id: string): Promise<boolean>;
}
```

Pontos canônicos:

- **`namespace XxxRepository`** agrupa todos os tipos auxiliares no mesmo arquivo do contrato — sem `xxx.types.ts`, sem pasta `types/`.
- **`DbRow`** descreve a linha de banco em snake_case (espelha exatamente as colunas CDS) — usado pela infrastructure para tipar `cds.run(SELECT...)`.
- **`Patch`** é `Partial<Omit<DbRow, 'tenant_id' | 'id'>>` — chaves não podem ser atualizadas; demais campos são opcionais.
- **`Filters`** descreve o shape de filtros dinâmicos (`findByFilters`).
- **Métodos retornam `XxxModel` ou `XxxModel[]`** — nunca `DbRow` direto. A infrastructure serializa via `XxxModel.with(row)` antes de devolver. Ver [infrastructure-layer/repositories/conventions.md → "Domain model como fronteira de retorno"](../../infrastructure-layer/repositories/conventions.md#domain-model-como-fronteira-de-retorno).
- **Métodos sem retorno de entidade** (`save`, `update`, `delete`) retornam `Promise<void>`.
- **Métodos contadores** (`count`, `exists`) retornam primitivos (`Promise<number>`, `Promise<boolean>`).

## Métodos típicos por categoria

| Categoria | Método | Retorno | Quando |
|---|---|---|---|
| **Leitura simples** | `findById(...)` | `Promise<XxxModel \| null>` | Buscar por chave; `null` quando ausente |
| | `findOne(filter)` | `Promise<XxxModel \| null>` | Buscar primeiro que casa filtro |
| | `findFirst(filter, sort)` | `Promise<XxxModel \| null>` | Buscar primeiro com ordenação explícita |
| **Leitura múltipla** | `findAll()` | `Promise<XxxModel[]>` | Buscar tudo (cuidado com volume) |
| | `findAllByTenant(t)` | `Promise<XxxModel[]>` | Buscar tudo de um tenant |
| | `findByFilters(t, f)` | `Promise<XxxModel[]>` | Filtros dinâmicos |
| | `findByKeys(keys)` | `Promise<XxxModel[]>` | Bulk lookup por chaves compostas |
| **Escrita** | `save(model)` | `Promise<void>` | Inserir novo |
| | `update(t, id, patch)` | `Promise<void>` | Atualizar campos parciais |
| | `delete(t, id)` | `Promise<void>` | Remover por chave |
| | `bulkInsert(rows)` | `void` (com UoW) ou `Promise<void>` | Inserir N linhas |
| | `bulkUpdate(updates)` | `void` (com UoW) ou `Promise<void>` | Atualizar N linhas |
| **Agregações** | `count(filter)` | `Promise<number>` | Contar registros |
| | `exists(t, id)` | `Promise<boolean>` | Checar existência sem trazer dados |

> O agrupamento dos métodos é por **negócio**, não por CRUD. Se o repositório só faz `findByTenant` e `findById`, ele tem só esses dois métodos. Não inflar a interface com métodos não usados.

## Repositório com `UnitOfWork` (batch)

Quando o repositório registra operações em lote para serem executadas em uma única transação, o contrato declara métodos **sem `Promise`** (apenas registram a intenção; o `await` acontece em `uow.commit()`). A implementação na infrastructure recebe o `UnitOfWork` via DI.

```typescript
// src/domain/repositories/part-numbers.ts
import type { PartNumberModel } from '@/domain/models/db/part-number.js';

export namespace PartNumberRepository {
    export type DbRow = {
        id: string;
        tenant_id: string;
        price_list_id: string;
        supplier_part_number: string;
        description: string | null;
    };
    export type InsertRow = Omit<DbRow, 'id'> & { id?: string };
    export type UpdatePatch = Partial<Pick<DbRow, 'supplier_part_number' | 'description'>>;
    export type Filters = {
        priceListId?: string;
        supplierPartNumber?: string;
    };
}

export interface PartNumberRepository {
    findById(tenantId: string, priceListId: string, id: string): Promise<PartNumberModel | null>;
    findByFilters(tenantId: string, filters: PartNumberRepository.Filters): Promise<PartNumberModel[]>;

    bulkInsert(rows: PartNumberRepository.InsertRow[]): void;
    bulkUpdate(updates: { id: string; patch: PartNumberRepository.UpdatePatch }[]): void;
}
```

> Os métodos `bulk*` retornam `void` (síncrono) — apenas registram intent no UoW. O `commit` final é responsabilidade do use case que chama `await uow.commit()`. Métodos de leitura permanecem `Promise<...>`.

## Repositório de tabela replicada (somente leitura)

Quando a entidade é replicada de um sistema externo (`db.replication.<Entidade>`), o contrato **não tem** `save`/`update`/`delete` — apenas leitura. Toda escrita ocorre via job de replicação CDS nativo.

```typescript
// src/domain/repositories/plants.ts
import type { PlantModel } from '@/domain/models/db/plant.js';

export namespace PlantRepository {
    export type DbRow = {
        plant_id: string;
        plant_name: string;
        country_code: string;
    };
}

export interface PlantRepository {
    findById(plantId: string): Promise<PlantModel | null>;
    findByCountry(countryCode: string): Promise<PlantModel[]>;
}
```

> Sem `save`, `update`, `delete`, ou `bulk*` — a interface comunica claramente que a entidade é read-only no contexto da aplicação.

## Filtros dinâmicos — sempre via `Filters` no namespace

Filtros opcionais combinados (`status`, `createdAfter`, `priceListId`, etc.) ficam tipados em `XxxRepository.Filters`. A interface aceita o objeto completo; a implementação na infrastructure encadeia `query.and(...)` por campo presente.

```typescript
export namespace DataLoadRepository {
    export type Filters = {
        status?: 'PENDING' | 'PROCESSING' | 'COMPLETED' | 'COMPLETED_WITH_ERRORS';
        createdAfter?: string;
        priceListId?: string;
    };
}

export interface DataLoadRepository {
    findByFilters(tenantId: string, filters: DataLoadRepository.Filters): Promise<DataLoadModel[]>;
}
```

> A semântica é **AND** entre os campos presentes. Se algum filtro precisar de OR, declarar uma estrutura específica (`type FiltersWithOr = { ... }`) ou criar um método dedicado (`findByStatusOrCreatedAfter`).

## Filtros com paginação

Quando o método retorna lista paginada, a paginação fica **separada** do filtro:

```typescript
export namespace PartNumberRepository {
    export type Filters = { priceListId?: string; supplierPartNumber?: string };
    export type Pagination = { top: number; skip: number };
    export type FindAllResult = { items: PartNumberModel[]; total: number };
}

export interface PartNumberRepository {
    findAll(
        filters: PartNumberRepository.Filters,
        pagination: PartNumberRepository.Pagination,
    ): Promise<PartNumberRepository.FindAllResult>;
}
```

## Imports permitidos

| O que pode importar | Exemplo |
|---|---|
| Models do domain | `import type { PriceListModel } from '@/domain/models/db/price-list.js';` |
| Tipo `Either` (raro — só se método retornar Either) | `import type { Either } from '@sweet-monads/either';` |

| O que **não** pode importar | Motivo |
|---|---|
| `@sap/cds`, `@models/*`, `@cds-models/*` | Acoplamento com framework |
| `application/`, `infrastructure/`, `presentation/`, `main/` | Quebra de isolamento |
| Outros repositories | Repositórios são contratos planos — composição é via use case |

## Regras

1. **Interface pura.** Nenhuma implementação dentro do arquivo — só `interface` + `namespace` com tipos.
2. **Tipos auxiliares no namespace** (`DbRow`, `Filters`, `Patch`, `Pagination`, `InsertRow`) — alinhado com [infrastructure-layer/repositories/conventions.md](../../infrastructure-layer/repositories/conventions.md). Sem arquivos `xxx.types.ts` separados.
3. **Métodos com retorno de entidade devolvem model**, nunca `DbRow` direto — infrastructure serializa via `XxxModel.with(row)`.
4. **`findById` retorna `null` se ausente; `findAll` retorna `[]`** — não lançar `NotFoundError` do contrato; quem decide se ausência é erro é o use case.
5. **`save`/`update`/`delete` retornam `Promise<void>`** — sem retorno de "registro atualizado".
6. **`bulk*` síncrono** quando há `UoW` (registra intent); assíncrono caso contrário.
7. **Sem barrel `repositories/index.ts`** — factories e use cases importam por caminho direto (`@/domain/repositories/price-lists.js`).
8. **Naming plural** para o arquivo (`price-lists.ts`, não `price-list.ts`) — espelha o nome da entidade CDS (`db.models.PriceLists`).
9. **Sem sufixo `-repository` no arquivo** (`price-lists.ts`, não `price-lists-repository.ts`) — a pasta já é o contexto.
10. **Sem `@sap/cds`** — quando precisar do tipo de uma `Transaction` ou `EventContext`, declarar wrapper local no namespace (`XxxRepository.Transaction`).

## Anti-padrões

### 1. Tipo top-level fora do namespace

```typescript
// ❌ ERRADO — tipos espalhados no module-level
export type PartNumberDbRow = { /* ... */ };
export type PartNumberFilters = { /* ... */ };
export type PartNumberPagination = { /* ... */ };

export interface PartNumberRepository { /* ... */ }
```

```typescript
// ✅ CORRETO — agrupados no namespace
export namespace PartNumberRepository {
    export type DbRow = { /* ... */ };
    export type Filters = { /* ... */ };
    export type Pagination = { /* ... */ };
}

export interface PartNumberRepository { /* ... */ }
```

> Tipos top-level prefixados (padrão MRO) funcionam mas multiplicam imports (`PartNumberDbRow`, `PartNumberFilters`, `PartNumberPagination` em vez de `PartNumberRepository.{DbRow, Filters, Pagination}`). O namespace é mais coeso e alinhado com o standard de [infrastructure-layer/repositories/conventions.md](../../infrastructure-layer/repositories/conventions.md).

### 2. Implementação dentro do contrato

```typescript
// ❌ ERRADO — método com lógica no domain/
export interface PriceListRepository {
    findById(id: string): Promise<PriceListModel | null>;
}

export class PriceListRepositoryHelper {
    static normalizeId(id: string): string { return id.trim().toLowerCase(); }
}
```

```typescript
// ✅ CORRETO — domain só declara contrato; helpers vivem no model ou na implementação
export interface PriceListRepository {
    findById(id: string): Promise<PriceListModel | null>;
}
// Normalização vai para PriceListModel.with(...) ou para PriceListRepositoryImpl
```

### 3. Acoplamento com `@sap/cds`

```typescript
// ❌ ERRADO — Transaction do CAP no contrato
import type { Transaction } from '@sap/cds';

export interface OperationInputRepository {
    executeInTransaction<T>(callback: (tx: Transaction) => Promise<T>): Promise<T>;
}
```

```typescript
// ✅ CORRETO — wrapper local no namespace
export namespace OperationInputRepository {
    export type Transaction = {
        run: (query: unknown) => Promise<unknown[]>;
        commit: () => Promise<void>;
        rollback: () => Promise<void>;
    };
}

export interface OperationInputRepository {
    executeInTransaction<T>(
        callback: (tx: OperationInputRepository.Transaction) => Promise<T>
    ): Promise<T>;
}
```

> Mesmo que pareça verboso, o wrapper isola o domain do framework. Se o CAP mudar a assinatura de `Transaction`, o impacto fica restrito à infrastructure que adapta.

### 4. Acoplamento com `@models/*` (CDS typer)

```typescript
// ❌ ERRADO — DbRow gerado pelo CDS typer
import type { PriceLists } from '@cds-models/db/models.js';

export interface PriceListRepository {
    findById(id: string): Promise<PriceLists | null>;
}
```

```typescript
// ✅ CORRETO — DbRow declarado independente do schema CDS
export namespace PriceListRepository {
    export type DbRow = {
        tenant_id: string;
        id: string;
        name: string;
    };
}

export interface PriceListRepository {
    findById(id: string): Promise<PriceListModel | null>;
}
```

### 5. Retorno de `DbRow` direto

```typescript
// ❌ ERRADO — vaza shape de banco para o consumidor
export interface PriceListRepository {
    findById(id: string): Promise<PriceListRepository.DbRow | null>;
}
```

```typescript
// ✅ CORRETO — retorna model
export interface PriceListRepository {
    findById(id: string): Promise<PriceListModel | null>;
}
```

### 6. Sufixo `-repository` no arquivo

```
❌  src/domain/repositories/price-list-repository.ts
✅  src/domain/repositories/price-lists.ts
```

A pasta `repositories/` já comunica o contexto — sufixo é redundante e alonga imports.

### 7. Singular em vez de plural

```
❌  src/domain/repositories/price-list.ts        (singular)
✅  src/domain/repositories/price-lists.ts       (plural — alinhado com db.models.PriceLists)
```

### 8. Métodos sem semântica de negócio (CRUD genérico)

```typescript
// ❌ ERRADO — método "create" sem semântica do domínio
export interface PriceListRepository {
    create(data: any): Promise<PriceListModel>;
    read(id: string): Promise<PriceListModel | null>;
    update(id: string, data: any): Promise<PriceListModel>;
    delete(id: string): Promise<boolean>;
}
```

```typescript
// ✅ CORRETO — métodos com semântica, tipos do domain
export interface PriceListRepository {
    save(model: PriceListModel): Promise<void>;
    findById(tenantId: string, id: string): Promise<PriceListModel | null>;
    update(tenantId: string, id: string, patch: PriceListRepository.Patch): Promise<void>;
    delete(tenantId: string, id: string): Promise<void>;
}
```

### 9. Repositório com `Either` em vez de exceção

```typescript
// ❌ ERRADO — repositório não retorna Either
export interface PriceListRepository {
    findById(id: string): Promise<Either<AbstractError, PriceListModel | null>>;
}
```

```typescript
// ✅ CORRETO — repositório propaga exceção; quem captura é o use case
export interface PriceListRepository {
    findById(id: string): Promise<PriceListModel | null>;
}
```

> O contrato com a application layer é: **caminho feliz** retorna o valor; **erro de I/O** propaga exceção que `BaseUseCaseImpl.handleError` converte em `ServerError`. `Either` é responsabilidade do use case, não do repository.

### 10. Barrel `repositories/index.ts`

```typescript
// ❌ ERRADO — barrel parcial ou completo no domain/repositories/
// src/domain/repositories/index.ts
export * from './price-lists.js';
export * from './tenants.js';
// ...
```

Sem barrel. Imports diretos:

```typescript
// ✅ CORRETO
import type { PriceListRepository } from '@/domain/repositories/price-lists.js';
import type { TenantRepository } from '@/domain/repositories/tenants.js';
```

> Barrels parciais (alguns repositories dentro, outros fora) são particularmente perniciosos — o leitor nunca sabe se deve importar pelo barrel ou direto. Eliminar a tentação removendo o barrel.

### 11. Múltiplas entidades no mesmo arquivo

```typescript
// ❌ ERRADO — duas entidades autônomas no mesmo contrato
export interface PurchaseOrderRepository {
    findOrderById(id: string): Promise<PurchaseOrderModel | null>;
    findPaymentTermById(id: string): Promise<PaymentTermModel | null>;
}
```

Quebrar em dois arquivos: `purchase-orders.ts` e `payment-terms.ts`. O use case coordena múltiplos repositories. Mesma regra da [infrastructure-layer/repositories/conventions.md → "Uma entidade por repository"](../../infrastructure-layer/repositories/conventions.md#uma-entidade-por-repository).
