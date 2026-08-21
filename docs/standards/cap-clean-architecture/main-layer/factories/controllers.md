# factories/controllers/

Instanciam controllers da camada de apresentação (`presentation/controllers/*`), injetando o use-case correspondente.

## Regras

- Um arquivo de factory por controller.
- O arquivo de factory tem **o mesmo nome** do arquivo de use-case correspondente.
- Sempre exportar um singleton (`export const xController = makeXController()`), pois o `routes/index.ts` importa a instância, não a função.
- A factory de controller **nunca** instancia repositórios ou adapters — apenas recebe o use-case pronto.

---

## Shape canônico (actions e functions)

```typescript
// src/main/factories/controllers/actions/approve-price-list.ts
import { makeApprovePriceListUseCase } from '@/main/factories/use-cases/actions/approve-price-list';
import { ApprovePriceListController } from '@/presentation/controllers/actions/approve-price-list';

const makeApprovePriceListController = () => {
    const useCase = makeApprovePriceListUseCase();
    return new ApprovePriceListController(useCase);
};

export const approvePriceListController = makeApprovePriceListController();
```

---

## Shape com singleton lazy

Use quando o controller não pode ser instanciado na inicialização do módulo (ex.: depende de contexto disponível só em runtime) ou quando deve ser obtido sob demanda.

```typescript
// src/main/factories/controllers/actions/upload-data-load.ts
import { makeUploadDataLoadUseCase } from '@/main/factories/use-cases/actions/upload-data-load';
import { UploadDataLoadController } from '@/presentation/controllers/actions/upload-data-load';

let instance: UploadDataLoadController | null = null;

const makeUploadDataLoadController = () => {
    const useCase = makeUploadDataLoadUseCase();
    return new UploadDataLoadController(useCase);
};

export const getUploadDataLoadController = (): UploadDataLoadController => {
    if (!instance) {
        instance = makeUploadDataLoadController();
    }
    return instance;
};
```

---

## Shape para entity-events

O nome do arquivo é o evento (`before-update.ts`, `before-create.ts`, `after-read.ts`). A pasta pai é o nome da entidade em kebab-case plural.

```typescript
// src/main/factories/controllers/entity-events/price-lists/before-update.ts
import { makeBeforeUpdatePriceListUseCase } from '@/main/factories/use-cases/entity-events/price-lists/before-update';
import { BeforeUpdatePriceListController } from '@/presentation/controllers/entity-events/price-lists/before-update';

const makeBeforeUpdatePriceListController = () => {
    const useCase = makeBeforeUpdatePriceListUseCase();
    return new BeforeUpdatePriceListController(useCase);
};

export const beforeUpdatePriceListController = makeBeforeUpdatePriceListController();
```

---

## Subpastas e correspondência com o CDS

| Subpasta | Tipo de operação CDS | Registro no `routes/index.ts` |
|---|---|---|
| `actions/` | `action MyAction(...)` | `service.on('myAction', ...)` |
| `functions/` | `function MyFn(...) returns ...` | `service.on('myFn', ...)` |
| `entity-events/<entidade>/` | Hooks de entidade | `service.before/after('CREATE'\|'UPDATE'\|'DELETE'\|'READ', 'Entity', ...)` |

## Estrutura de pastas (referência)

```
factories/controllers/
├── actions/
│   ├── approve-price-list.ts
│   ├── checkout.ts
│   └── ...
├── entity-events/
│   ├── price-lists/
│   │   └── before-update.ts
│   ├── tenant-erp-connections/
│   │   ├── before-create.ts
│   │   ├── before-update.ts
│   │   └── before-delete.ts
│   └── tenants/
│       └── before-update.ts
└── functions/
    ├── catalog-home.ts
    ├── get-part-numbers.ts
    └── ...
```
