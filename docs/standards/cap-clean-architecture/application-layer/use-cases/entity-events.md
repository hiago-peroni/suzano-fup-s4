# use-cases/entity-events/

Use cases de entity-events são invocados automaticamente pelos hooks do CAP — eventos disparados antes ou depois de operações CRUD em entidades do serviço. Cada entidade tem sua própria pasta dentro de `entity-events/`; cada hook tem seu próprio arquivo.

---

## Hooks disponíveis

| Arquivo | Quando usar |
|---|---|
| `before-read.ts` | Enriquecer ou filtrar o `SELECT` antes de ir ao banco — ex.: injetar filtros de tenant, aplicar restrições de acesso, hidratar o request com dados do usuário |
| `before-create.ts` | Validar o payload, verificar duplicidades, calcular campos derivados antes da inserção |
| `before-update.ts` | Validar o payload de atualização, verificar regras de negócio antes de persistir |
| `before-delete.ts` | Verificar pré-condições para exclusão — ex.: entidade não pode ser deletada se tiver filhos |
| `after-read.ts` | Enriquecer os resultados retornados com dados externos — ex.: buscar dados de API S/4, calcular campos não persistidos |
| `after-create.ts` | Disparar side-effects pós-criação — ex.: atualizar status de entidades relacionadas, enviar notificações |
| `after-update.ts` | Disparar side-effects pós-atualização — ex.: propagar mudanças para entidades dependentes |
| `after-delete.ts` | Disparar side-effects pós-exclusão — ex.: limpar registros relacionados em cascata |

Crie apenas os arquivos dos hooks que a entidade realmente usa. Não crie arquivos de hook vazios.

---

## Estrutura de pastas

```
use-cases/entity-events/
├── price-lists/
│   ├── before-create.ts
│   ├── before-update.ts
│   ├── before-delete.ts
│   └── index.ts
├── user-preferences/
│   ├── before-read.ts
│   └── index.ts
└── flights/
    ├── before-create.ts
    ├── after-create.ts
    ├── before-update.ts
    └── index.ts
```

---

## Shape canônico — `before-*`

Hooks `before-*` recebem o payload ainda não persistido. Podem retornar dados enriquecidos (que a rota aplica via `Object.assign`) ou simplesmente `right(undefined)` para sinalizar aprovação.

```typescript
// src/application/use-cases/entity-events/user-preferences/before-read.ts
import { left, right } from '@sweet-monads/either';

import { BaseUseCaseImpl } from '@/application/use-cases/base/base.js';
import { BadRequestError } from '@/domain/errors/index.js';
import type { BeforeReadUserPreferencesHydrator } from '@/domain/hydrators/user-preferences/before-read.js';
import type { BeforeReadUserPreferencesUseCase } from '@/domain/use-cases/entity-events/user-preferences/before-read.js';

export class BeforeReadUserPreferencesUseCaseImpl extends BaseUseCaseImpl implements BeforeReadUserPreferencesUseCase {
    constructor(private readonly hydrator: BeforeReadUserPreferencesHydrator) {
        super();
    }

    public async execute(params: BeforeReadUserPreferencesUseCase.Params): Promise<BeforeReadUserPreferencesUseCase.Result> {
        try {
            if (!params.userId) {
                throw new BadRequestError('auth.userNotAuthenticated');
            }
            this.hydrator.hydrate({ request: params.request, userId: params.userId });
            return right(undefined);
        } catch (error) {
            return left(this.handleError(error));
        }
    }
}
```

### `before-create` com validação de modelo e verificação de duplicidade

```typescript
// src/application/use-cases/entity-events/operation-flight-types/before-create.ts
import { left, right } from '@sweet-monads/either';

import { BaseUseCaseImpl } from '@/application/use-cases/base/base.js';
import { BadRequestError } from '@/domain/errors/index.js';
import { OperationFlightTypeModel } from '@/domain/models/db/operation-flight-type.js';
import type { OperationFlightTypeRepository } from '@/domain/repositories/operation-flight-type.js';
import { BeforeCreateOperationFlightTypeUseCase } from '@/domain/use-cases/entity-events/operation-flight-types/index.js';
import type { Translator } from '@/domain/utils/translator.js';

export class BeforeCreateOperationFlightTypeUseCaseImpl extends BaseUseCaseImpl implements BeforeCreateOperationFlightTypeUseCase {
    constructor(
        private readonly translator: Translator,
        private readonly operationFlightTypeRepository: OperationFlightTypeRepository
    ) {
        super();
    }

    public async execute(params: BeforeCreateOperationFlightTypeUseCase.Params): Promise<BeforeCreateOperationFlightTypeUseCase.Result> {
        try {
            const operationFlightType = OperationFlightTypeModel.forCreate(params);
            const validation = operationFlightType.validate();
            if (validation.hasError) {
                const messages = validation.errorMessages
                    .map((msg) => this.translator.translate(msg))
                    .join('\n');
                return left(new BadRequestError(messages));
            }
            const existing = await this.operationFlightTypeRepository.findById(params.id);
            if (existing) {
                return left(new BadRequestError(this.translator.translate('operationFlightTypeIdAlreadyExists')));
            }
            return right(operationFlightType.toCreationObject());
        } catch (error) {
            return left(this.handleError(error));
        }
    }
}
```

> O `right(operationFlightType.toCreationObject())` retorna o payload enriquecido/transformado. A rota (`routes/index.ts`) recebe esse valor e o aplica via `Object.assign(request.data, result.data)` antes do CAP persistir.

---

## Shape canônico — `after-*`

Hooks `after-*` recebem os dados já persistidos. São usados para side-effects pós-persistência ou enriquecimento de resultados.

```typescript
// src/application/use-cases/entity-events/flights/after-create.ts
import { left, right } from '@sweet-monads/either';

import { BaseUseCaseImpl } from '@/application/use-cases/base/base.js';
import type { UpdateLogbookStatusService } from '@/domain/services/update-logbook-status.js';
import type { UpdateAircraftStatusService } from '@/domain/services/update-aircraft-status.js';
import { AfterCreateFlightUseCase } from '@/domain/use-cases/entity-events/flights/after-create.js';

export class AfterCreateFlightUseCaseImpl extends BaseUseCaseImpl implements AfterCreateFlightUseCase {
    constructor(
        private readonly updateLogbookStatusService: UpdateLogbookStatusService,
        private readonly updateAircraftStatusService: UpdateAircraftStatusService
    ) {
        super();
    }

    public async execute(params: AfterCreateFlightUseCase.Params): Promise<AfterCreateFlightUseCase.Result> {
        try {
            const logbookResult = await this.updateLogbookStatusService.execute({
                logbookId: params.logbookId,
                flightId: params.id
            });
            if (logbookResult.isLeft()) {
                return left(logbookResult.value);
            }
            const aircraftResult = await this.updateAircraftStatusService.execute({
                aircraftId: params.aircraftId
            });
            if (aircraftResult.isLeft()) {
                return left(aircraftResult.value);
            }
            return right(logbookResult.value);
        } catch (error) {
            return left(this.handleError(error));
        }
    }
}
```

### Propagação de `Either` de services internos

Quando um use case delega para services que também retornam `Either`, propague o erro sem re-empacotar:

```typescript
// ✅ Propagação correta
const result = await this.someService.execute(params);
if (result.isLeft()) {
    return left(result.value); // ← propaga o AbstractError original intacto
}

// ❌ Errado — re-empacota e perde o tipo de erro
if (result.isLeft()) {
    return left(new ServerError('service failed')); // ← destrói a semântica
}
```

---

## `index.ts` barrel (obrigatório por entidade)

Cada pasta de entidade expõe um `index.ts` que reexporta todos os seus use cases. Isso simplifica os imports nas factories de DI em `main/factories/use-cases/entity-events/`.

```typescript
// src/application/use-cases/entity-events/price-lists/index.ts
export { BeforeCreatePriceListUseCaseImpl } from './before-create.js';
export { BeforeUpdatePriceListUseCaseImpl } from './before-update.js';
export { BeforeDeletePriceListUseCaseImpl } from './before-delete.js';
```

---

## Anti-padrão: lógica de negócio misturada com hook de infraestrutura

❌ **Nunca acesse `cds` ou banco diretamente no use case:**

```typescript
// ERRADO — use case acessa infraestrutura diretamente
import cds from '@sap/cds';

export class BeforeReadOrderUseCaseImpl extends BaseUseCaseImpl implements BeforeReadOrderUseCase {
    public async execute(params: BeforeReadOrderUseCase.Params): Promise<BeforeReadOrderUseCase.Result> {
        try {
            const orders = await cds.run(cds.ql.SELECT.from('Orders').where({ tenantId: params.tenantId }));
            // ...
        } catch (error) {
            return left(this.handleError(error));
        }
    }
}
```

✅ **Injete o repositório e use a interface:**

```typescript
// CERTO — acesso via repositório injetado
export class BeforeReadOrderUseCaseImpl extends BaseUseCaseImpl implements BeforeReadOrderUseCase {
    constructor(private readonly orderRepository: OrderRepository) {
        super();
    }

    public async execute(params: BeforeReadOrderUseCase.Params): Promise<BeforeReadOrderUseCase.Result> {
        try {
            const orders = await this.orderRepository.findByTenant(params.tenantId);
            // ...
            return right(undefined);
        } catch (error) {
            return left(this.handleError(error));
        }
    }
}
```

---

## Naming

| Elemento | Convenção | Exemplo |
|---|---|---|
| Pasta da entidade | `kebab-case-plural` | `price-lists/`, `user-preferences/`, `operation-flight-types/` |
| Arquivo do hook | `<evento>-<ação>.ts` | `before-create.ts`, `after-read.ts` |
| Classe | `<Evento><Ação><Entidade>UseCaseImpl` | `BeforeCreateOperationFlightTypeUseCaseImpl` |
| Contrato no domínio | `<Evento><Ação><Entidade>UseCase` (interface) | `BeforeCreateOperationFlightTypeUseCase` |
