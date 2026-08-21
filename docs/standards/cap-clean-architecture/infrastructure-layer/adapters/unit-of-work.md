# Unit of Work

O padrão Unit of Work acumula operações CDS em memória (`intents`) e executa tudo em uma única transação `cds.tx()` no `commit()`. Garante ACID para cargas em lote sem que cada repositório precise gerenciar transação.

## Quando usar

**Apenas em processing-services com batch transacional**: data load, mass update, integração em lote. Use cases de application-service regulares não precisam — o CDS gerencia transação automaticamente no handler do evento.

## Estrutura

```
src/infrastructure/adapters/unit-of-work/
└── cds-unit-of-work.ts    → CdsUnitOfWork
```

## Interface no domínio

Todos os tipos do contrato — incluindo `Where` e o discriminated union `Intent` — ficam no **namespace** da interface, no `domain/`. A implementação em `infrastructure/` consome esses tipos sem declarar nenhum tipo local.

```typescript
// src/domain/adapters/unit-of-work.ts
export namespace UnitOfWork {
    export type Where = Record<string, unknown>;
    export type Row = Record<string, unknown>;

    export type Intent =
        | { kind: 'insert'; entity: string; rows: Row[] }
        | { kind: 'upsert'; entity: string; rows: Row[] }
        | { kind: 'update'; entity: string; where: Where; patch: Row }
        | { kind: 'delete'; entity: string; where: Where };
}

export interface UnitOfWork {
    registerInsert<T extends UnitOfWork.Row>(entityName: string, rows: T | T[]): void;
    registerUpsert<T extends UnitOfWork.Row>(entityName: string, rows: T | T[]): void;
    registerUpdate(entityName: string, where: UnitOfWork.Where, patch: UnitOfWork.Row): void;
    registerDelete(entityName: string, where: UnitOfWork.Where): void;
    commit(): Promise<void>;
    clear(): void;
    readonly pendingCount: number;
}
```

## Few-shot — `CdsUnitOfWork`

```typescript
// src/infrastructure/adapters/unit-of-work/cds-unit-of-work.ts
import cds from '@sap/cds';

import type { Telemetry } from '@/domain/adapters/telemetry.js';
import type { UnitOfWork } from '@/domain/adapters/unit-of-work.js';

export class CdsUnitOfWork implements UnitOfWork {
    private readonly TELEMETRY_EVENT = 'unit-of-work.commit';
    private readonly DEV_ENV = 'development';
    private intents: UnitOfWork.Intent[] = [];

    constructor(private readonly telemetry: Telemetry) {}

    public registerInsert<T extends UnitOfWork.Row>(entityName: string, rows: T | T[]): void {
        const list = Array.isArray(rows) ? rows : [rows];
        if (list.length === 0) {
            return;
        }
        this.intents.push({ kind: 'insert', entity: entityName, rows: list });
    }

    public registerUpsert<T extends UnitOfWork.Row>(entityName: string, rows: T | T[]): void {
        const list = Array.isArray(rows) ? rows : [rows];
        if (list.length === 0) {
            return;
        }
        this.intents.push({ kind: 'upsert', entity: entityName, rows: list });
    }

    public registerUpdate(entityName: string, where: UnitOfWork.Where, patch: UnitOfWork.Row): void {
        this.intents.push({ kind: 'update', entity: entityName, where, patch });
    }

    public registerDelete(entityName: string, where: UnitOfWork.Where): void {
        this.intents.push({ kind: 'delete', entity: entityName, where });
    }

    public async commit(): Promise<void> {
        if (this.intents.length === 0) {
            return;
        }
        const buffered = this.intents;
        this.intents = [];

        // Guard SQLite vs HANA: better-sqlite3 usa I/O síncrono e trava o processo
        // se cds.tx() for chamado enquanto outro handler CAP mantém o lock do arquivo.
        // Em dev, cada intent roda fora de cds.tx aproveitando a transação implícita
        // do SQLite. Em prod (HANA), cds.tx() garante ACID.
        if (process.env.NODE_ENV === this.DEV_ENV) {
            for (const intent of buffered) {
                await this.executeIntent(intent);
            }
        } else {
            await cds.tx(async () => {
                for (const intent of buffered) {
                    await this.executeIntent(intent);
                }
            });
        }

        this.telemetry.event(this.TELEMETRY_EVENT, { intents: buffered.length });
    }

    public clear(): void {
        this.intents = [];
    }

    public get pendingCount(): number {
        return this.intents.length;
    }

    private async executeIntent(intent: UnitOfWork.Intent): Promise<void> {
        if (intent.kind === 'insert') {
            await cds.run(cds.ql.INSERT.into(intent.entity).entries(intent.rows));
            return;
        }
        if (intent.kind === 'upsert') {
            await cds.run(cds.ql.UPSERT.into(intent.entity).entries(intent.rows));
            return;
        }
        if (intent.kind === 'update') {
            await cds.run(cds.ql.UPDATE.entity(intent.entity).where(intent.where).with(intent.patch));
            return;
        }
        if (intent.kind === 'delete') {
            await cds.run(cds.ql.DELETE.from(intent.entity).where(intent.where));
        }
    }
}
```

Pontos de atenção no exemplo acima:

- `UnitOfWork.Intent`, `UnitOfWork.Where` e `UnitOfWork.Row` vêm do `domain/` — zero declaração local de `type`/`interface`.
- Constantes (`TELEMETRY_EVENT`, `DEV_ENV`) ficam como `private readonly` da classe.
- `intents: UnitOfWork.Intent[]` é estado mutável da instância — fica como propriedade da classe (sem `readonly`), não em module-level.

### Guard SQLite vs HANA

O `if (process.env.NODE_ENV === 'development')` dentro de `commit()` é a única exceção aceita ao anti-padrão de selector `NODE_ENV` em infrastructure. A justificativa é técnica: `better-sqlite3` — driver padrão do CAP em dev — implementa I/O síncrono. Quando `cds.tx()` é invocado enquanto outro handler CAP mantém o lock do arquivo SQLite, o processo Node.js trava em deadlock. Em produção (HANA), `cds.tx()` é necessário para garantir atomicidade. O comportamento diferencia **mecanismo de transação**, não lógica de negócio.

## Few-shot — Repositório batch consumindo `UnitOfWork`

```typescript
// src/infrastructure/repositories/part-numbers.ts
import type { PartNumberRepository } from '@/domain/repositories/part-numbers.js';
import type { UnitOfWork } from '@/domain/adapters/unit-of-work.js';

export class PartNumberRepositoryImpl implements PartNumberRepository {
    private readonly ENTITY = 'db.models.PartNumbers';

    constructor(private readonly uow: UnitOfWork) {}

    public bulkInsert(partNumbers: PartNumberRepository.InsertRow[]): void {
        this.uow.registerInsert(this.ENTITY, partNumbers);
    }

    public bulkUpdate(updates: { id: string; patch: PartNumberRepository.UpdatePatch }[]): void {
        for (const u of updates) {
            this.uow.registerUpdate(this.ENTITY, { id: u.id }, u.patch);
        }
    }
}
```

O repositório **não chama `cds.run` diretamente** — toda operação vai via `registerXxx`. Quem chama `commit()` é o use case (ou o pipeline de processing-service), após acumular todas as operações do batch.

## Tipagem e constantes

Aplicar as 3 regras globais da [seção "Tipagem e constantes" do README da infrastructure-layer](../README.md#tipagem-e-constantes). Especificamente para o UoW:

- `UnitOfWork.Intent` é a única definição do discriminated union — declarada no namespace da interface em `domain/adapters/unit-of-work.ts`. A implementação `CdsUnitOfWork` consome o tipo via import; não redeclara `type Intent` localmente.
- `UnitOfWork.Where` e `UnitOfWork.Row` substituem `Record<string, unknown>` solto, oferecendo nomes semânticos no contrato.
- Strings que parametrizam a execução (nome do evento de telemetria, label do `NODE_ENV`) ficam como `private readonly` da classe — não como `const` em module-level.

## Regras

1. **Stateful por execução de pipeline.** Uma instância de `CdsUnitOfWork` por execução de use case. A factory cria nova instância a cada invocação — nunca singleton.

2. **Repositórios com UoW recebem-no via DI no constructor.** Exceção à regra geral de "repositórios sem DI" — justificada pela natureza transacional do batch.

3. **`commit()` é idempotente para `intents` vazios.** Retorna imediatamente se `this.intents.length === 0`. Seguro chamar mesmo que nenhuma operação tenha sido registrada.

4. **Nunca chamar `cds.run` direto em repositório que usa UoW.** Toda operação CDS passa por `registerInsert`, `registerUpsert`, `registerUpdate` ou `registerDelete`. Chamadas diretas a `cds.run` quebram a garantia de atomicidade do `commit()`.

5. **Tipos do discriminated union (`Intent`) vivem no domain.** Nada de `type Intent = ...` ou `interface XxxIntent` dentro de `infrastructure/adapters/unit-of-work/`.
