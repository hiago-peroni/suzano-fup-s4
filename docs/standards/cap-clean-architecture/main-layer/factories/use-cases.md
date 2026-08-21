# factories/use-cases/

Instanciam use-cases da camada `application/use-cases/*`, injetando repositórios, adapters e serviços.

## Regras

- A subpasta (`actions/`, `functions/`, `entity-events/`) e o nome do arquivo **espelham exatamente** os de `factories/controllers/`.
- Use-cases nunca exportam singleton — apenas a função `make*` — pois quem exporta o singleton é a factory de controller.
- Nunca instanciar `*RepositoryImpl` ou `*AdapterImpl` diretamente: chame as factories correspondentes.

---

## Shape simples (1-2 dependências)

```typescript
// src/main/factories/use-cases/functions/catalog-home.ts
import { CatalogHomeUseCaseImpl } from '@/application/use-cases/functions/catalog-home-use-case-impl';
import { makeCatalogSearchRepository } from '@/main/factories/repositories/catalog-search';

export const makeCatalogHomeUseCase = () => {
    return new CatalogHomeUseCaseImpl(makeCatalogSearchRepository());
};
```

---

## Shape complexo (múltiplas dependências)

```typescript
// src/main/factories/use-cases/actions/approve-price-list.ts
import { ApprovePriceListUseCaseImpl } from '@/application/use-cases/actions/approve-price-list';
import { makeDataLoadProductStagesRepository } from '@/main/factories/repositories/data-load-product-stages';
import { makeDataLoadRepository } from '@/main/factories/repositories/data-loads';
import { makePartNumberDescriptionRepository } from '@/main/factories/repositories/part-number-descriptions';
import { makePartNumberPendingChangesRepository } from '@/main/factories/repositories/part-number-pending-changes';
import { makePartNumberPriceHistoriesRepository } from '@/main/factories/repositories/part-number-price-histories';
import { makePartNumberRepository } from '@/main/factories/repositories/part-numbers';
import { makePriceListRepository } from '@/main/factories/repositories/price-lists';

export const makeApprovePriceListUseCase = () => {
    return new ApprovePriceListUseCaseImpl(
        makePriceListRepository(),
        makeDataLoadProductStagesRepository(),
        makePartNumberPendingChangesRepository(),
        makePartNumberRepository(),
        makePartNumberPriceHistoriesRepository(),
        makePartNumberDescriptionRepository(),
        makeDataLoadRepository(),
    );
};
```

---

## Shape com composição de use-cases

Use quando um use-case orquestra outros use-cases (ex.: pipeline de processamento em etapas).

```typescript
// src/main/factories/use-cases/actions/process-data-load-queue.ts
import { ProcessDataLoadQueueUseCaseImpl } from '@/application/use-cases/actions/process-data-load-queue';
import { makeDataLoadQueueRepository } from '@/main/factories/repositories/data-load-queue';
import { makeDataLoadRepository } from '@/main/factories/repositories/data-loads';
import { makeFeatureFlagRepository } from '@/main/factories/repositories/feature-flag-repository';
import { makePriceListRepository } from '@/main/factories/repositories/price-lists';
import { makeTenantFeatureFlagRepository } from '@/main/factories/repositories/tenant-feature-flag-repository';
import { makeFinalizeDataLoadUseCase } from '@/main/factories/use-cases/actions/finalize-data-load';
import { makeImportDataLoadUseCase } from '@/main/factories/use-cases/actions/import-data-load';
import { makeInitializeDataLoadUseCase } from '@/main/factories/use-cases/actions/initialize-data-load';
import { makePromoteDataLoadUseCase } from '@/main/factories/use-cases/actions/promote-data-load';

export const makeProcessDataLoadQueueUseCase = () => {
    return new ProcessDataLoadQueueUseCaseImpl(
        makeDataLoadQueueRepository(),
        makePriceListRepository(),
        makeFeatureFlagRepository(),
        makeTenantFeatureFlagRepository(),
        makeInitializeDataLoadUseCase(),
        makeImportDataLoadUseCase(),
        makePromoteDataLoadUseCase(),
        makeFinalizeDataLoadUseCase(),
        makeDataLoadRepository(),
    );
};
```

---

## Shape para entity-events

```typescript
// src/main/factories/use-cases/entity-events/price-lists/before-update.ts
import { BeforeUpdatePriceListUseCaseImpl } from '@/application/use-cases/entity-events/price-lists/before-update';
import { makePriceListRepository } from '@/main/factories/repositories/price-lists';

export const makeBeforeUpdatePriceListUseCase = () => {
    return new BeforeUpdatePriceListUseCaseImpl(makePriceListRepository());
};
```
