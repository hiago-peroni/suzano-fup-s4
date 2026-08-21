# Replication — Subpasta `replication/`

Tabelas em `db.replication.*` são **espelhos locais** de master data de sistemas externos (Plants, Materials, Suppliers, Cost Centers, etc.) — populadas por job de replicação CDS nativo e lidas como cache local. São conceitualmente diferentes das tabelas próprias do serviço (`db.models.*`) e ficam em uma subpasta dedicada para tornar essa distinção explícita no código.

> A subpasta canônica é `replication/` — **não** `sap/`. Replicação não é exclusiva do SAP: pode ser espelho de SuccessFactors, Salesforce, MDM próprio, qualquer sistema externo cujos dados são replicados para o banco local via job CDS. O critério é o **namespace CDS** (`db.replication.*`), não a origem do dado.

## Por que subpasta separada?

| Aspecto | Tabelas próprias (`db.models.*`) | Tabelas replicadas (`db.replication.*`) |
|---|---|---|
| Proprietário dos dados | O próprio serviço | Sistema externo (S4, SuccessFactors, MDM, etc.) |
| Operações permitidas | Leitura e escrita | Somente leitura |
| Escrita feita por | Use cases da aplicação | Job de replicação CDS nativo |
| Localização do repositório | `repositories/models/<entidade>.ts` | `repositories/replication/<entidade>.ts` |
| Namespace CDS | `db.models.*` | `db.replication.*` |

## Estrutura

```
infrastructure/repositories/
├── models/                          → tabelas próprias (db.models.*)
│   └── price-lists.ts
└── replication/                     → tabelas replicadas (db.replication.*)
    ├── plants.ts                    → PlantRepositoryImpl
    ├── materials.ts                 → MaterialRepositoryImpl
    └── suppliers.ts                 → SupplierRepositoryImpl
```

## Few-shot canônico

```typescript
// src/domain/repositories/replication/plants.ts
import type { PlantModel } from '@/domain/models/replication/plant.js';

export namespace PlantRepository {
    export type DbRow = {
        plantId: string;
        name: string;
        // ...demais colunas espelhadas da entidade db.replication.Plants
    };
}

export interface PlantRepository {
    findById(plantId: string): Promise<PlantModel | null>;
    findAll(): Promise<PlantModel[]>;
}
```

```typescript
// src/infrastructure/repositories/replication/plants.ts
import cds from '@sap/cds';

import { PlantModel } from '@/domain/models/replication/plant.js';
import type { PlantRepository } from '@/domain/repositories/replication/plants.js';

export class PlantRepositoryImpl implements PlantRepository {
    private readonly ENTITY = 'db.replication.Plants';

    public async findById(plantId: string): Promise<PlantModel | null> {
        const [row]: PlantRepository.DbRow[] = await cds.run(
            cds.ql.SELECT.from(this.ENTITY).where({ plantId }).limit(1)
        );
        return row ? PlantModel.with(row) : null;
    }

    public async findAll(): Promise<PlantModel[]> {
        const rows: PlantRepository.DbRow[] = await cds.run(cds.ql.SELECT.from(this.ENTITY));
        return rows.map(PlantModel.with);
    }
}
```

Pontos de atenção no exemplo acima:

- `ENTITY` aponta para `db.replication.Plants` (não `db.models.*`) — sempre como `private readonly` da classe.
- Apenas `findById` e `findAll` — nenhum método de escrita.
- Modelo em `domain/models/replication/plant.ts` (subpasta `replication/`, separada dos modelos próprios em `domain/models/`).
- Interface em `domain/repositories/replication/plants.ts`, com `PlantRepository.DbRow` no namespace.
- A infra importa `PlantRepository` (a interface) e consome `PlantRepository.DbRow` — sem declarar tipos locais.
- O retorno é via `PlantModel.with(row)` — regra de [domain model como fronteira](./conventions.md#domain-model-como-fronteira-de-retorno) vale igual para replicação.

## Regras específicas

1. **Subpasta `replication/` somente se houver tabelas `db.replication.*`.** Projetos sem replicação não criam a subpasta — não antecipar estrutura desnecessária.

2. **Modelos em `domain/models/replication/<entidade>.ts`.** Não misturar com modelos próprios em `domain/models/<entidade>.ts`. A subpasta `replication/` deixa claro que o modelo representa um espelho de master data externo.

3. **Interfaces em `domain/repositories/replication/<entidade>.ts`.** Mesma separação no lado do domínio.

4. **Repositório de replicação é somente leitura.** Sem métodos `save`, `update` ou `delete` — a escrita é responsabilidade exclusiva do job de replicação CDS nativo. Declarar apenas `findById`, `findAll` e variações de leitura.

5. **Uma entidade por repositório de replicação.** Mesma regra de [uma entidade por repository](./conventions.md#uma-entidade-por-repository) — joins para enriquecer dados da entidade pai são permitidos via `cds.ql.SELECT.from(this.ENTITY, c => ...)`, com a entidade auxiliar declarada como `private readonly` ao lado da `ENTITY`.

6. **Fallback cache miss → API externa pertence ao use case.** Quando a estratégia é "busca no cache local; se não encontrar, consulta a API do sistema externo", a lógica de fallback fica no use case:

    ```typescript
    // ✅ CORRETO — use case decide a estratégia
    const plant = await this.plantRepository.findById(plantId);          // replication/
    if (!plant) {
        return await this.companyApi.findPlantById(plantId);             // adapter externo
    }
    return plant;
    ```

    Não fundir as duas responsabilidades em uma única classe de repositório.

## Anti-padrão: misturar replicação com tabelas próprias na raiz

```
❌ ERRADO — plants.ts na raiz de repositories/ quando a tabela é db.replication.Plants
infrastructure/repositories/
├── price-lists.ts          → db.models.PriceLists       ❌ raiz não é mais válida
├── plants.ts               → db.replication.Plants      ❌ confunde o leitor
└── ...

✅ CORRETO — separação explícita por namespace CDS
infrastructure/repositories/
├── models/
│   └── price-lists.ts      → db.models.PriceLists
└── replication/
    └── plants.ts           → db.replication.Plants
```

O caminho do arquivo (`repositories/replication/plants.ts`) e o valor de `ENTITY` (`db.replication.Plants`) devem ser consistentes. Um arquivo em `replication/` com `ENTITY = 'db.models.*'` é igualmente errado na direção oposta — assim como um arquivo em `models/` apontando para `db.replication.*`.

## Anti-padrão: subpasta `sap/`

```
❌ ERRADO — sap/ pressupõe origem específica e não cobre outros sistemas externos
infrastructure/repositories/
└── sap/
    └── plants.ts

✅ CORRETO — replication/ é agnóstico ao sistema de origem
infrastructure/repositories/
└── replication/
    └── plants.ts
```

A subpasta `sap/` (padrão legado de Suzano FUP, MRO) presume que toda replicação vem do SAP. Como replicações podem vir de SuccessFactors, Salesforce, MDM próprio, qualquer sistema externo, o nome correto é `replication/` — pareando exatamente com o namespace CDS `db.replication.*`.
