# application/services/

Services na camada de aplicação são **operações auxiliares reutilizáveis** chamadas por outros use cases — nunca diretamente por controllers. Representam passos atômicos de pipelines de processamento, side-effects pós-persistência, integrações com sistemas externos, ou cálculos reutilizáveis entre múltiplos use cases.

---

## Distinção entre use case e service

| Critério | Use case | Service |
|---|---|---|
| Quem chama | Controller (via presentation layer) | Outro use case ou outro service |
| Contrato em | `domain/use-cases/<tipo>/<nome>.ts` | `domain/services/<contexto>/<nome>.ts` |
| Implementação em | `application/use-cases/<tipo>/<nome>.ts` | `application/services/<contexto>/<nome>.ts` |
| Classe base | `BaseUseCaseImpl` | `BaseServiceImpl` |
| Sufixo de classe | `XxxUseCaseImpl` | `XxxServiceImpl` |

Se um artefato pode ser chamado diretamente por um controller → é um use case. Se só é chamado por outros use cases → é um service.

---

## Estrutura de pasta

```
application/services/
├── base/
│   └── base.ts                   → BaseServiceImpl abstrata
└── <contexto>/
    └── <kebab-name>.ts            → um service por responsabilidade
```

Exemplos de contextos: `data-load/`, `integration/`, `notification/`, `calculation/`.

---

## Few-shot: service de pipeline atômico

Services de pipeline são passos independentes de um processamento em batch/queue. Cada passo tem sua própria classe e pode ser reordenado, substituído ou testado isoladamente.

```typescript
// src/application/services/data-load/finalize-data-load.ts
import { left, right } from '@sweet-monads/either';

import { BaseServiceImpl } from '@/application/services/base/base.js';
import type { DataLoadRepository } from '@/domain/repositories/data-loads.js';
import type { DataLoadQueueRepository } from '@/domain/repositories/data-load-queue.js';
import type { PriceListRepository } from '@/domain/repositories/price-lists.js';
import { FinalizeDataLoadService } from '@/domain/services/data-load/finalize-data-load.js';

export class FinalizeDataLoadServiceImpl extends BaseServiceImpl implements FinalizeDataLoadService {
    constructor(
        private readonly dataLoadRepo: DataLoadRepository,
        private readonly queueRepo: DataLoadQueueRepository,
        private readonly priceListRepo: PriceListRepository
    ) {
        super();
    }

    public async execute(params: FinalizeDataLoadService.Params): Promise<FinalizeDataLoadService.Result> {
        try {
            const { dataLoad, counters } = params;
            const hasErrors = counters.errorRecords > 0;
            const dataLoadStatus = hasErrors ? 'COMPLETED_WITH_ERRORS' : 'COMPLETED';
            await this.dataLoadRepo.updateProgress(dataLoad.id, {
                status: dataLoadStatus,
                processedRecords: counters.processedRecords,
                errorRecords: counters.errorRecords
            });
            await this.queueRepo.markCompleted(dataLoad.id);
            await this.priceListRepo.updateLoadStatus(
                dataLoad.priceList_tenant_id,
                dataLoad.priceList_id,
                hasErrors ? 'ERROR' : 'SUCCESS'
            );
            return right(undefined);
        } catch (error) {
            return left(this.handleError(error));
        }
    }
}
```

### Pipeline completo de data load (referência de sequência)

```
application/services/data-load/
├── initialize-data-load.ts     → registra o job
├── import-data-load.ts         → importa os registros brutos
├── validate-keys.ts            → valida chaves de negócio
├── validate-service-number.ts  → valida número de serviço
├── promote-data-load.ts        → promove para produção
└── finalize-data-load.ts       → finaliza e notifica
```

O use case de ação (`ProcessDataLoadUseCaseImpl`) injeta cada service como dependência e os executa em sequência, propagando erros via `Either`.

---

## Few-shot: service auxiliar de side-effect

Services de side-effect encapsulam atualizações em entidades relacionadas disparadas por eventos de outra entidade.

```typescript
// src/application/services/update-logbook-status.ts
import { left, right } from '@sweet-monads/either';

import { BaseServiceImpl } from '@/application/services/base/base.js';
import { NotFoundError } from '@/domain/errors/index.js';
import type { LogbookRepository } from '@/domain/repositories/logbook.js';
import { UpdateLogbookStatusService } from '@/domain/services/update-logbook-status.js';

export class UpdateLogbookStatusServiceImpl extends BaseServiceImpl implements UpdateLogbookStatusService {
    constructor(private readonly logbookRepository: LogbookRepository) {
        super();
    }

    public async execute(params: UpdateLogbookStatusService.Params): Promise<UpdateLogbookStatusService.Result> {
        try {
            const logbook = await this.logbookRepository.findById(params.logbookId);
            if (!logbook) {
                throw new NotFoundError('logbook.notFound');
            }
            const updatedStatus = logbook.computeStatusAfterFlight(params.flightId);
            await this.logbookRepository.updateStatus(params.logbookId, updatedStatus);
            return right({ logbookId: params.logbookId, status: updatedStatus });
        } catch (error) {
            return left(this.handleError(error));
        }
    }
}
```

### Como o use case orquestra services com `Either`

```typescript
// src/application/use-cases/entity-events/flights/after-create.ts (trecho)
const logbookResult = await this.updateLogbookStatusService.execute({
    logbookId: params.logbookId,
    flightId: params.id
});
if (logbookResult.isLeft()) {
    return left(logbookResult.value); // ← propaga sem re-empacotar
}

const aircraftResult = await this.updateAircraftStatusService.execute({
    aircraftId: params.aircraftId
});
if (aircraftResult.isLeft()) {
    return left(aircraftResult.value);
}

return right(logbookResult.value);
```

---

## Regras

1. **Contrato em `domain/services/`**, implementação em `application/services/`.
2. **Services nunca são chamados por controllers.** Se um controller precisa chamar um service, extraia um use case que orquestre esse service.
3. **`execute` é o único método público** — mesma convenção dos use cases.
4. **`Result` é `Either<AbstractError, T>`** — sem exceção.
5. **Services podem chamar outros services**, mas evite cadeias longas. Se a cadeia crescer, considere extrair um use case de orquestração.
6. **Agrupe por contexto de negócio** em subpastas (`data-load/`, `integration/`, etc.) quando há mais de dois services no mesmo domínio.

---

## Naming

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo | `kebab-case.ts` | `finalize-data-load.ts` |
| Classe | `PascalCase` + `ServiceImpl` | `FinalizeDataLoadServiceImpl` |
| Contrato no domínio | `PascalCase` + `Service` (interface) | `FinalizeDataLoadService` |
| Pasta de contexto | `kebab-case/` | `data-load/`, `integration/` |

| Artefato | Localização |
|---|---|
| Interface/contrato | `domain/services/<contexto>/<nome>.ts` |
| Implementação | `application/services/<contexto>/<nome>.ts` |
| Classe base | `application/services/base/base.ts` |
