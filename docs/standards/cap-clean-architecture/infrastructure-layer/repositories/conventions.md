# Conventions — Repositories

Referência detalhada de padrões de implementação para repositórios na infrastructure layer. Leia este documento junto com o [README](./README.md) antes de implementar qualquer `*RepositoryImpl`.

## Shape canônico — stateless, sem DI

O caso padrão: repositório sem dependências injetadas, usando `cds` como singleton global. Todos os tipos (`PriceListRepository`, `PriceListRepository.DbRow`) vêm do `domain/` — zero declaração local na infrastructure.

```typescript
// src/domain/repositories/price-lists.ts
import type { PriceListModel } from '@/domain/models/price-list.js';

export namespace PriceListRepository {
    export type DbRow = {
        tenant_id: string;
        id: string;
        name: string;
        // ...demais colunas snake_case da tabela CDS
    };
    export type Patch = Partial<DbRow>;
}

export interface PriceListRepository {
    findById(tenantId: string, id: string): Promise<PriceListModel | null>;
    findAllByTenant(tenantId: string): Promise<PriceListModel[]>;
    save(model: PriceListModel): Promise<void>;
    update(tenantId: string, id: string, patch: PriceListRepository.Patch): Promise<void>;
    delete(tenantId: string, id: string): Promise<void>;
}
```

```typescript
// src/infrastructure/repositories/models/price-lists.ts
import cds from '@sap/cds';

import { PriceListModel } from '@/domain/models/price-list.js';
import type { PriceListRepository } from '@/domain/repositories/price-lists.js';

export class PriceListRepositoryImpl implements PriceListRepository {
    private readonly ENTITY = 'db.models.PriceLists';

    public async findById(tenantId: string, id: string): Promise<PriceListModel | null> {
        const rows: PriceListRepository.DbRow[] = await cds.run(
            cds.ql.SELECT.from(this.ENTITY).where({ tenant_id: tenantId, id }).limit(1)
        );
        if (rows.length === 0) {
            return null;
        }
        return PriceListModel.with(rows[0]);
    }

    public async findAllByTenant(tenantId: string): Promise<PriceListModel[]> {
        const rows: PriceListRepository.DbRow[] = await cds.run(
            cds.ql.SELECT.from(this.ENTITY).where({ tenant_id: tenantId }).orderBy('createdAt desc')
        );
        return rows.map((r) => PriceListModel.with(r));
    }

    public async save(model: PriceListModel): Promise<void> {
        const data = model.toRow();
        await cds.run(cds.ql.INSERT.into(this.ENTITY).entries(data));
    }

    public async update(tenantId: string, id: string, patch: PriceListRepository.Patch): Promise<void> {
        await cds.run(cds.ql.UPDATE.entity(this.ENTITY).set(patch).where({ tenant_id: tenantId, id }));
    }

    public async delete(tenantId: string, id: string): Promise<void> {
        await cds.run(cds.ql.DELETE.from(this.ENTITY).where({ tenant_id: tenantId, id }));
    }
}
```

## Domain model como fronteira de retorno

Todo método de repositório que retorna **dados de uma entidade** serializa o raw via `XxxModel.with(raw)` antes de devolver para a application layer. Isso garante que:

1. **A serialização é obrigatória e centralizada** no domain model — campos snake_case viram camelCase, tipos primitivos viram value objects, defaults são aplicados no único lugar canônico.
2. **A camada externa nunca vê `DbRow`.** O `XxxRepository.DbRow` é detalhe da infrastructure; quem cruza a fronteira é o `XxxModel`.
3. **O código fica homogêneo.** Toda chamada `await repo.findXxx(...)` retorna model — sem precisar lembrar se "esse aqui" devolve raw ou serializado.

| Tipo de método | Retorna model? | Exemplo |
|---|---|---|
| `findById`, `findOne`, `findFirst` | ✅ Sim — `Promise<XxxModel \| null>` | `findById(id): Promise<PriceListModel \| null>` |
| `findAll`, `findByFilters`, `findByTenant` | ✅ Sim — `Promise<XxxModel[]>` | `findAllByTenant(t): Promise<PriceListModel[]>` |
| `save`, `update`, `delete` | ❌ Não — `Promise<void>` | `save(model): Promise<void>` |
| `count`, `exists` | ❌ Não — `Promise<number>` / `Promise<boolean>` | `count(filter): Promise<number>` |
| `bulkInsert`, `bulkUpdate` (via UoW) | ❌ Não — `void` (registra intent) | `bulkInsert(rows): void` |

✅ **Certo — retorno sempre via model:**

```typescript
public async findById(id: string): Promise<PriceListModel | null> {
    const rows: PriceListRepository.DbRow[] = await cds.run(
        cds.ql.SELECT.from(this.ENTITY).where({ id }).limit(1)
    );
    return rows[0] ? PriceListModel.with(rows[0]) : null;
}
```

❌ **Errado — retorna raw direto:**

```typescript
public async findById(id: string): Promise<PriceListRepository.DbRow | null> {
    const [row] = await cds.run(cds.ql.SELECT.from(this.ENTITY).where({ id }).limit(1));
    return row ?? null;
}
```

❌ **Errado — retorna objeto montado manualmente fora do model:**

```typescript
public async findById(id: string): Promise<{ id: string; name: string }> {
    const [row] = await cds.run(cds.ql.SELECT.from(this.ENTITY).where({ id }));
    return { id: row.id, name: row.name };
}
```

> Se o método **só retorna um primitivo** (contagem, flag de existência) ou **não tem retorno de entidade** (`void`), a regra não se aplica — não há o que serializar.

## Uma entidade por repository

Cada repositório opera sobre **uma única entidade CDS principal**. Não herdar entidades terciárias dentro da mesma classe — se você precisar de dados de duas entidades sem relação direta na mesma operação, o orquestrador é o use case (que chama dois repositórios).

**Joins são permitidos** quando a query precisa trazer colunas correlacionadas de uma entidade auxiliar (lookup, master data, expand de associação CDS). Use a forma do query builder com função de colunas:

```typescript
cds.ql.SELECT.from(this.ENTITY, (c) => {
    c('*');
    c.supplier((s) => {
        s('id');
        s('name');
    });
}).where({ tenant_id: tenantId });
```

Quando o join trouxer entidade auxiliar, declarar essa entidade como `private readonly` ao lado da `ENTITY` pai — o leitor entende rapidamente que o repositório lê **uma** entidade principal e enriquece com **N** auxiliares conhecidas.

✅ **Certo — entidade auxiliar declarada como `readonly` ao lado da pai:**

```typescript
export class PurchaseOrderRepositoryImpl implements PurchaseOrderRepository {
    private readonly ENTITY = 'db.models.PurchaseOrders';
    private readonly SUPPLIER_ENTITY = 'db.replication.Suppliers';  // ← auxiliar de join

    public async findByIdWithSupplier(id: string): Promise<PurchaseOrderModel | null> {
        const rows: PurchaseOrderRepository.DbRow[] = await cds.run(
            cds.ql.SELECT.from(this.ENTITY, (c) => {
                c('*');
                c.supplier((s) => {
                    s('id');
                    s('name');
                });
            }).where({ id }).limit(1)
        );
        return rows[0] ? PurchaseOrderModel.with(rows[0]) : null;
    }
}
```

❌ **Errado — repositório com duas entidades principais (sem relação de join):**

```typescript
export class PurchaseOrderRepositoryImpl implements PurchaseOrderRepository {
    private readonly ENTITY = 'db.models.PurchaseOrders';
    private readonly PAYMENT_TERMS_ENTITY = 'db.models.PaymentTerms';  // ← entidade autônoma

    public async findById(id: string): Promise<PurchaseOrderModel | null> { /* ... */ }
    public async findPaymentTermById(id: string): Promise<PaymentTermModel | null> {
        const [row] = await cds.run(cds.ql.SELECT.from(this.PAYMENT_TERMS_ENTITY).where({ id }));
        return row ? PaymentTermModel.with(row) : null;
    }
}
```

`PaymentTerms` é uma entidade autônoma com seu próprio ciclo de vida — pertence a `PaymentTermRepositoryImpl`. O use case chama os dois repositórios e coordena.

❌ **Errado — repositório herdando outro repositório para reaproveitar queries:**

```typescript
export class PurchaseOrderWithSuppliersRepositoryImpl extends SupplierRepositoryImpl { /* ... */ }
```

Herança entre repositórios é proibida. Composição via use case é o caminho.

## SQL pleno como exceção

A regra geral é **sempre** usar o query builder do CAP (`cds.ql.SELECT/INSERT/UPDATE/DELETE`). SQL bruto via `cds.run({ sql, values })` é exceção, justificada apenas quando o CAP não oferece o recurso. Casos aceitos:

| Caso | Justificativa |
|---|---|
| TVF HANA (`SELECT * FROM TF_FOO(?)`) | `cds.ql` não suporta Table-Valued Functions nativas |
| Stored procedure (`CALL SCHEMA.SP_FOO(?, ?)`) | Idem — execução de procedure não tem builder |
| Hint específico do HANA (`/*+ NO_INLINE */`) | Otimizações de plano de execução só via SQL bruto |
| `forUpdate()` para locking pessimista | Não tem equivalente em `cds.ql` — usar global `SELECT` |

Em todos os casos, a string SQL fica como `private readonly XXX_SQL` da classe (nunca module-level) e um comentário explica por que o builder não atende:

```typescript
// src/infrastructure/repositories/models/fup-history.ts
export class FupHistoryRepositoryImpl implements FupHistoryRepository {
    private readonly ENTITY = 'db.models.FupHistory';
    // TVF HANA — cds.ql não suporta Table-Valued Functions; SQL bruto é único caminho.
    private readonly TVF_SQL = 'SELECT * FROM TF_GET_FUP_HISTORY(?)';
    private readonly SQLITE_KIND = 'sqlite';

    public async getByPurchaseOrder(purchaseOrderId: string): Promise<FupHistoryModel[]> {
        if (cds.db?.kind === this.SQLITE_KIND) {
            const rows: FupHistoryModel.Row[] = await cds.run(
                cds.ql.SELECT.from(this.ENTITY).where({ purchaseOrderId })
            );
            return rows.map(FupHistoryModel.with);
        }
        const rows: FupHistoryModel.Row[] = await cds.run({ sql: this.TVF_SQL, values: [purchaseOrderId] });
        return rows.map(FupHistoryModel.with);
    }
}
```

❌ **Errado — SQL bruto sem justificativa quando `cds.ql` atende:**

```typescript
// ❌ ERRADO — query builder atende perfeitamente
const sql = `SELECT * FROM "db.models.PriceLists" WHERE tenant_id = ?`;
const rows = await cds.run({ sql, values: [tenantId] });

// ✅ CORRETO — query builder
const rows = await cds.run(cds.ql.SELECT.from(this.ENTITY).where({ tenant_id: tenantId }));
```

## Operações CDS canônicas

| Operação | API canônica | Quando |
|---|---|---|
| Insert | `cds.ql.INSERT.into(ENTITY).entries(data)` | Inserir 1 ou N linhas |
| Upsert | `cds.ql.UPSERT.into(ENTITY).entries(data)` | Insert-or-update por chave |
| Update | `cds.ql.UPDATE.entity(ENTITY).set(patch).where(filter)` | Atualizar campos parciais |
| Delete | `cds.ql.DELETE.from(ENTITY).where(filter)` | Remover por filtro |
| Select 1 | `cds.ql.SELECT.from(ENTITY, columns?).where(filter).limit(1)` | Retornar primeira linha |
| Select N | `cds.ql.SELECT.from(ENTITY, columns?).where(filter).orderBy(...).limit(top, skip)` | Paginação |
| Select com join | `cds.ql.SELECT.from(ENTITY, c => { c('*'); c.assoc(a => a('id')); })` | Expand de associação CDS |
| Locking HANA | `SELECT.from(ENTITY).where({ id }).forUpdate()` (global, sem `cds.ql`) | Lock pessimista por linha |
| SQL bruto | `cds.run({ sql: this.XXX_SQL, values: [...] })` | TVF, stored procedure, hint HANA |

## Repositório com DI de `UnitOfWork` (batch)

Quando o repositório registra operações em lote para serem executadas em uma única transação, ele recebe o `UnitOfWork` via constructor. Os métodos **não são `async`** — apenas registram a intenção; o `await` acontece no `uow.commit()` que o use case dispara.

```typescript
// src/infrastructure/repositories/models/part-numbers.ts
import type { UnitOfWork } from '@/domain/adapters/unit-of-work.js';
import type { PartNumberRepository } from '@/domain/repositories/part-numbers.js';

export class PartNumberRepositoryImpl implements PartNumberRepository {
    private readonly ENTITY = 'db.models.PartNumbers';

    constructor(private readonly uow: UnitOfWork) {}

    public bulkInsert(rows: PartNumberRepository.InsertRow[]): void {
        this.uow.registerInsert(this.ENTITY, rows);
    }

    public bulkUpdate(updates: { id: string; patch: PartNumberRepository.UpdatePatch }[]): void {
        for (const u of updates) {
            this.uow.registerUpdate(this.ENTITY, { id: u.id }, u.patch);
        }
    }
}
```

## Locking pessimista HANA (`forUpdate`)

`forUpdate()` não existe na API `cds.ql` — usa-se o global `SELECT` diretamente. O lock é liberado ao final da transação corrente (`cds.tx`). Em SQLite local, o `forUpdate()` é silenciosamente ignorado (no-op).

```typescript
// src/infrastructure/repositories/models/resource-locks.ts
import type { ResourceLockRepository, ResourceLockRow } from '@/domain/repositories/resource-locks.js';

export class ResourceLockRepositoryImpl implements ResourceLockRepository {
    private readonly ENTITY = 'db.models.ResourceLocks';

    public async findByIdForUpdate(id: string): Promise<ResourceLockRow | null> {
        // SELECT global + forUpdate() — bloqueia a linha até o final da transação atual
        // Requer cds.tx ativo; em SQLite local, forUpdate() é no-op silencioso
        const [row] = await SELECT.from(this.ENTITY).where({ id }).forUpdate();
        return row ?? null;
    }
}
```

## Colunas customizadas, filtros dinâmicos e mapeamento local

Quando a query seleciona colunas explícitas ou o `WHERE` é construído dinamicamente, o tipo da linha (snake_case espelhando o resultado do CDS) vive no namespace da interface no `domain/` — nunca como `type` local dentro da infrastructure.

```typescript
// src/domain/repositories/data-loads.ts
export namespace DataLoadRepository {
    export type DbRow = {
        id: string;
        tenant_id: string;
        status: string;
        created_at: string;
        processed_at: string | null;
    };
    export type Filters = {
        status?: string;
        createdAfter?: string;
    };
}

export interface DataLoadRepository {
    findByFilters(tenantId: string, filters: DataLoadRepository.Filters): Promise<DataLoadModel[]>;
}
```

```typescript
// src/infrastructure/repositories/models/data-loads.ts
import cds from '@sap/cds';

import { DataLoadModel } from '@/domain/models/data-load.js';
import type { DataLoadRepository } from '@/domain/repositories/data-loads.js';

export class DataLoadRepositoryImpl implements DataLoadRepository {
    private readonly ENTITY = 'db.models.DataLoads';
    private readonly LOG_COLUMNS = ['id', 'tenant_id', 'status', 'created_at', 'processed_at'];

    public async findByFilters(
        tenantId: string,
        filters: DataLoadRepository.Filters,
    ): Promise<DataLoadModel[]> {
        let query = cds.ql.SELECT.from(this.ENTITY, this.LOG_COLUMNS).where({ tenant_id: tenantId });
        query = this.applyFilters(query, filters);
        const rows: DataLoadRepository.DbRow[] = await cds.run(query);
        return rows.map((r) => this.mapRow(r));
    }

    private applyFilters(
        query: SELECT<DataLoadRepository.DbRow>,
        filters: DataLoadRepository.Filters,
    ): SELECT<DataLoadRepository.DbRow> {
        if (filters.status) {
            query = query.and({ status: filters.status });
        }
        if (filters.createdAfter) {
            query = query.and(`created_at >= '${filters.createdAfter}'`);
        }
        return query;
    }

    private mapRow(row: DataLoadRepository.DbRow): DataLoadModel {
        return DataLoadModel.with({
            id: row.id,
            tenantId: row.tenant_id,
            status: row.status,
            createdAt: new Date(row.created_at),
            processedAt: row.processed_at ? new Date(row.processed_at) : null,
        });
    }
}
```

Note que o array de colunas (`LOG_COLUMNS`) é `private readonly` da classe — não `const LOG_COLUMNS = [...]` em module-level. Mesmo para arrays imutáveis, a [regra global de constantes](../README.md#tipagem-e-constantes) se aplica.

## TVF HANA + sqlite guard

TVFs são o caso mais comum de [SQL pleno](#sql-pleno-como-exceção) — e exigem fallback para SQLite local (driver de testes não suporta TVF nativa do HANA). O guard `cds.db?.kind === 'sqlite'` resolve. Veja o few-shot canônico de `FupHistoryRepositoryImpl` na seção SQL pleno; ele mostra o padrão completo:

- `private readonly ENTITY` (caminho do builder para o SQLite).
- `private readonly TVF_SQL` (SQL bruto do HANA com comentário justificando).
- `private readonly SQLITE_KIND = 'sqlite'` (constante para o guard).
- Retorno sempre via `FupHistoryModel.with(...)` — a regra de [domain model como fronteira](#domain-model-como-fronteira-de-retorno) vale tanto no caminho HANA quanto no SQLite.

## `cds.ql.*` vs globais `SELECT`/`UPDATE`

Dentro do query builder, o padrão é o namespace `cds.ql.*` (`cds.ql.SELECT` / `cds.ql.INSERT` / `cds.ql.UPDATE` / `cds.ql.DELETE`).

Exceções aceitas para globais (`SELECT`/`UPDATE` sem `cds.ql`):

- **`SELECT.from(...).forUpdate()`** — não há equivalente em `cds.ql`; usar o global `SELECT` apenas para locking pessimista.
- **`INSERT.into(...).entries(...)`** global — aceito em hot paths que evitam o overhead do builder (raro; justificar com comentário).

Nunca usar `cds.update(...).with(...)` (API antiga sem `.entity()`).

Para casos onde **nem** `cds.ql` **nem** globais atendem (TVFs, stored procedures, hints HANA), recorrer a SQL bruto via `cds.run({ sql, values })` — ver [SQL pleno como exceção](#sql-pleno-como-exceção).

## Tipagem e constantes

Aplicar as 3 regras globais da [seção "Tipagem e constantes" do README da infrastructure-layer](../README.md#tipagem-e-constantes). Especificamente para repositórios:

| Conceito | Onde fica | Exemplo |
|---|---|---|
| Shape da linha de banco (snake_case) | Namespace da interface em `domain/repositories/<entidade>.ts` | `PriceListRepository.DbRow` |
| Shape da linha quando há um Model com `Row` correspondente | Namespace do model em `domain/models/<entidade>.ts` | `FupHistoryModel.Row` |
| `Filters`, `Patch`, `InsertRow`, `UpdatePatch` | Namespace da interface em `domain/repositories/<entidade>.ts` | `DataLoadRepository.Filters`, `PartNumberRepository.InsertRow` |
| Nome da entidade CDS (`db.models.Xxx`) | `private readonly ENTITY` da classe | `private readonly ENTITY = 'db.models.PriceLists';` |
| Lista de colunas projetadas (TVF, partial select) | `private readonly LOG_COLUMNS = [...]` da classe | — |
| String SQL bruta (TVF HANA) | `private readonly XXX_SQL = '...'` da classe | `private readonly TVF_SQL = 'SELECT ...';` |

❌ **Errado — `type` local declarado no arquivo do repositório:**

```typescript
// src/infrastructure/repositories/models/data-loads.ts
type DataLoadDbRow = {       // ← declaração local proibida
    id: string;
    tenant_id: string;
};

const LOG_COLUMNS = ['id', 'tenant_id'];   // ← module-level proibido

export class DataLoadRepositoryImpl implements DataLoadRepository {
    // ...
}
```

✅ **Certo — tipo no domain, constante na classe:**

```typescript
// src/domain/repositories/data-loads.ts
export namespace DataLoadRepository {
    export type DbRow = { id: string; tenant_id: string; };
}

// src/infrastructure/repositories/models/data-loads.ts
import type { DataLoadRepository } from '@/domain/repositories/data-loads.js';

export class DataLoadRepositoryImpl implements DataLoadRepository {
    private readonly LOG_COLUMNS = ['id', 'tenant_id'];

    public async findByFilters(/* ... */): Promise<DataLoadModel[]> {
        const rows: DataLoadRepository.DbRow[] = await cds.run(/* ... */);
        return rows.map(DataLoadModel.with);
    }
}
```

## Anti-padrões

### 1. Catch silencioso → null

```typescript
// ❌ ERRADO — oculta falhas de infra (timeout, constraint violation)
try {
    const [row] = await cds.run(cds.ql.SELECT.from(this.ENTITY).where({ id }).limit(1));
    return row ? PriceListModel.with(row) : null;
} catch {
    return null;
}

// ✅ CORRETO — deixar propagar; quem trata é o use case
const [row] = await cds.run(cds.ql.SELECT.from(this.ENTITY).where({ id }).limit(1));
return row ? PriceListModel.with(row) : null;
```

### 2. Sufixo `-repository` no nome do arquivo

```
❌  purchase-order-delivery-details-repository.ts
✅  purchase-order-delivery-details.ts
```

O diretório `repositories/` já é o contexto — o sufixo é redundante e alonga desnecessariamente o caminho de import.

### 3. Singular/plural inconsistente

```
❌  part-number.ts
✅  part-numbers.ts
```

Usar sempre **plural** no nome do arquivo, alinhado com o nome da entidade CDS (`db.models.PartNumbers`).

### 4. Constante `ENTITY` sem `readonly` ou em lowercase

```typescript
// ❌ ERRADO
private entity = 'db.models.PriceLists';

// ✅ CORRETO
private readonly ENTITY = 'db.models.PriceLists';
```

`UPPER_SNAKE` + `readonly` sinaliza que é uma constante de infra, não um campo mutável.

### 5. `cds.update().with(...)` (API antiga)

```typescript
// ❌ ERRADO
await cds.update(this.ENTITY).with(patch).where({ id });

// ✅ CORRETO
await cds.run(cds.ql.UPDATE.entity(this.ENTITY).set(patch).where({ id }));
```

### 6. Importar de `application/` ou `presentation/` no repositório

```typescript
// ❌ ERRADO
import type { UpdatePriceListDto } from '@/application/use-cases/update-price-list.js';

// ✅ CORRETO — repositório conhece apenas @sap/cds e domain/
import type { PriceListRepository } from '@/domain/repositories/price-lists.js';
```

A infrastructure layer importa **apenas** de `domain/` e de outras partes de `infrastructure/`.

### 7. `type XxxDbRow` declarado dentro do arquivo do repositório

```typescript
// ❌ ERRADO — type local na infrastructure
type DataLoadDbRow = { id: string; tenant_id: string; };

export class DataLoadRepositoryImpl implements DataLoadRepository { /* ... */ }
```

Todo tipo de linha vive no namespace da interface no `domain/` (`DataLoadRepository.DbRow`). Mesmo se usado apenas dentro de um único repositório.

### 8. `const FOO = ...` em module-level dentro de `infrastructure/repositories/`

```typescript
// ❌ ERRADO
const LOG_COLUMNS = ['id', 'tenant_id', 'status'];
const TVF_SQL = 'SELECT * FROM TF_GET_FUP_HISTORY(?)';

export class XxxRepositoryImpl implements XxxRepository { /* ... */ }

// ✅ CORRETO — propriedades da classe
export class XxxRepositoryImpl implements XxxRepository {
    private readonly LOG_COLUMNS = ['id', 'tenant_id', 'status'];
    private readonly TVF_SQL = 'SELECT * FROM TF_GET_FUP_HISTORY(?)';
}
```

A [regra global de constantes](../README.md#tipagem-e-constantes) proíbe `const`/`let` em module-level dentro de `infrastructure/`.

### 9. Retornar `DbRow` ou objeto avulso em vez de `XxxModel`

```typescript
// ❌ ERRADO — DbRow vaza para a application layer
public async findById(id: string): Promise<PriceListRepository.DbRow | null> {
    const [row] = await cds.run(cds.ql.SELECT.from(this.ENTITY).where({ id }));
    return row ?? null;
}

// ❌ ERRADO — objeto montado fora do model
public async findById(id: string): Promise<{ id: string; name: string } | null> {
    const [row] = await cds.run(cds.ql.SELECT.from(this.ENTITY).where({ id }));
    return row ? { id: row.id, name: row.name } : null;
}

// ✅ CORRETO — sempre via model
public async findById(id: string): Promise<PriceListModel | null> {
    const [row]: PriceListRepository.DbRow[] = await cds.run(
        cds.ql.SELECT.from(this.ENTITY).where({ id }).limit(1)
    );
    return row ? PriceListModel.with(row) : null;
}
```

`save`/`update`/`delete` (`Promise<void>`) e `count`/`exists` (primitivos) ficam isentos — não há entidade a serializar.

### 10. Duas (ou mais) entidades principais no mesmo repositório

```typescript
// ❌ ERRADO — duas entidades autônomas no mesmo repositório
export class PurchaseOrderRepositoryImpl implements PurchaseOrderRepository {
    private readonly ENTITY = 'db.models.PurchaseOrders';
    private readonly PAYMENT_TERMS_ENTITY = 'db.models.PaymentTerms';

    public async findPurchaseOrderById(id: string): Promise<PurchaseOrderModel | null> { /* ... */ }
    public async findPaymentTermById(id: string): Promise<PaymentTermModel | null> { /* ... */ }
}
```

Quebrar em dois repositórios (`PurchaseOrderRepositoryImpl` + `PaymentTermRepositoryImpl`). O use case coordena.

> **Não é violação:** declarar `private readonly SUPPLIER_ENTITY = 'db.replication.Suppliers'` ao lado da `ENTITY` pai **quando** essa entidade aparece apenas como **alvo de join** dentro de queries da entidade pai. Ela não tem método público próprio; só é referenciada em `cds.ql.SELECT.from(this.ENTITY, c => c.supplier(...))`.

### 11. Herança entre repositórios

```typescript
// ❌ ERRADO
export class PurchaseOrderWithSuppliersRepositoryImpl extends SupplierRepositoryImpl { /* ... */ }
```

Repositórios não estendem outros repositórios. Reuso é via use case (composição), não herança.

### 12. SQL pleno quando o query builder atende

```typescript
// ❌ ERRADO — cds.ql suporta este caso perfeitamente
const sql = `SELECT * FROM "db.models.PriceLists" WHERE tenant_id = ?`;
const rows = await cds.run({ sql, values: [tenantId] });

// ✅ CORRETO
const rows = await cds.run(cds.ql.SELECT.from(this.ENTITY).where({ tenant_id: tenantId }));
```

SQL bruto fica reservado para TVFs, stored procedures e hints HANA. Sempre com `private readonly XXX_SQL` da classe e comentário justificando.

### 13. SQL bruto inline (não como `private readonly`)

```typescript
// ❌ ERRADO — SQL inline no método
public async getByPo(id: string): Promise<FupHistoryModel[]> {
    const rows = await cds.run({ sql: 'SELECT * FROM TF_GET_FUP(?)', values: [id] });
    return rows.map(FupHistoryModel.with);
}

// ✅ CORRETO — extraído para constante da classe
private readonly TVF_SQL = 'SELECT * FROM TF_GET_FUP(?)';

public async getByPo(id: string): Promise<FupHistoryModel[]> {
    const rows = await cds.run({ sql: this.TVF_SQL, values: [id] });
    return rows.map(FupHistoryModel.with);
}
```

Centralizar SQL em `private readonly` facilita auditoria de segurança (revisar todas as strings SQL do projeto = `grep readonly.*_SQL`) e reúso entre métodos.
