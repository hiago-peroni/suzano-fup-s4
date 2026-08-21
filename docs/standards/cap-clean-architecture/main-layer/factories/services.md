# factories/services/

Instanciam serviços definidos em `domain/services/*`, cujas implementações vivem em `application/services/*`.

## Distinção fundamental: use-case vs service

| | `factories/use-cases/` | `factories/services/` |
|---|---|---|
| **Chamado por** | `routes/index.ts` (diretamente pela rota CDS) | Outros use-cases (orquestração interna) |
| **Interface em** | `application/use-cases/` | `domain/services/` |
| **Implementação em** | `application/use-cases/` | `application/services/` |
| **Quando criar** | Toda operação exposta em `index.cds` (action, function, entity-event) | Sub-unidade de pipeline chamada internamente por use-cases |

> **Regra**: se o componente é chamado diretamente pelo `routes/index.ts`, é use-case. Se é orquestrado por outro use-case, é service.

## Quando criar `factories/services/`

Crie a pasta somente se existirem interfaces em `domain/services/`.

Cenário típico: um use-case complexo delega etapas para sub-services (ex.: `processDataLoadQueue` orquestra `importService`, `promoteService`, `finalizeService`).

---

## Shape 1 — Sub-service de pipeline (application/services)

Use quando o service é uma etapa interna de um use-case mais complexo. A implementação fica em `application/services/`.

```typescript
// src/main/factories/services/import-data-load.ts
import type { ImportDataLoadService } from '@/domain/services/data-load/import-data-load';
import { ImportDataLoadServiceImpl } from '@/application/services/data-load/import-data-load';
import { makeDataLoadRepository } from '@/main/factories/repositories/data-loads';
import { makeSpreadsheetParser } from '@/main/factories/adapters/spreadsheet-parser';
import { makeUnitOfWork } from '@/main/factories/adapters/unit-of-work';

export const makeImportDataLoadService = (): ImportDataLoadService => {
    return new ImportDataLoadServiceImpl(
        makeDataLoadRepository(),
        makeSpreadsheetParser(),
        makeUnitOfWork(),
    );
};
```

---

## Regras

- O tipo de retorno sempre é a interface do domínio (`): ImportDataLoadService`).
- Services **não** exportam singleton por padrão — são criados sob demanda pela factory do use-case que os consome.
- Se o componente precisa ser exposto como rota no `index.cds`, ele é use-case, não service.
