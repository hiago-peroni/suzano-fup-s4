# application/use-cases/base/ e application/services/base/

Define as classes base abstratas da camada de aplicação. Todo use case herda `BaseUseCaseImpl`; todo service herda `BaseServiceImpl`. As duas classes são estruturalmente idênticas — o nome diferente serve para tornar explícito, nas assinaturas de herança, se um artefato é chamado por controllers (use case) ou por outros use cases (service).

---

## `BaseUseCaseImpl`

### Arquivo: `use-cases/base/base.ts`

```typescript
// src/application/use-cases/base/base.ts
import { AbstractError } from '@/domain/errors/abstract-error.js';
import { ServerError } from '@/domain/errors/server-error.js';

export abstract class BaseUseCaseImpl {
    protected handleError(error: unknown): AbstractError {
        if (error instanceof AbstractError) {
            return error;
        }
        const err = error as Error;
        return new ServerError(err.stack, err.message);
    }
}
```

### Por que `unknown` na tipagem do parâmetro?

TypeScript, no modo estrito, não garante o tipo de um valor capturado num `catch`. Tipar o parâmetro como `unknown` força a verificação de instância antes de usar a variável — tornando o código seguro por construção.

Alternativas observadas em projetos legados:

| Abordagem | Problema |
|---|---|
| `error as Error` (cast direto) | Não é seguro — o throw pode ser de qualquer tipo |
| `error as AbstractError` (cast para erro de domínio) | Perde a mensagem se o erro for um `Error` genérico de runtime |
| `error: Error & GatewayError` (intersecção) | Específico para gateways S/4 — não se aplica a erros internos |

### Por que verificar `instanceof AbstractError`?

Sem essa verificação, um `throw new BadRequestError(...)` lançado dentro de um método privado e capturado pelo `catch` do `execute` seria convertido em `ServerError`, perdendo o código HTTP e a mensagem de negócio originais.

```typescript
// ✅ Comportamento correto com instanceof check
private validateSomething(value: string): void {
    if (!value) {
        throw new BadRequestError('validation.required'); // ← erro de domínio
    }
}

public async execute(params: XxxUseCase.Params): Promise<XxxUseCase.Result> {
    try {
        this.validateSomething(params.value); // ← pode throw BadRequestError
        // ...
        return right(result);
    } catch (error) {
        return left(this.handleError(error));
        // handleError preserva o BadRequestError intacto (instanceof AbstractError → true)
    }
}
```

```typescript
// ❌ Comportamento errado sem instanceof check
protected handleError(error: unknown): AbstractError {
    const err = error as Error;
    return new ServerError(err.stack, err.message); // ← destrói o BadRequestError
}
```

---

## Few-shot: use case herdando `BaseUseCaseImpl`

```typescript
// src/application/use-cases/actions/save-theme.ts
import { left, right } from '@sweet-monads/either';

import { BaseUseCaseImpl } from '@/application/use-cases/base/base.js';
import { BadRequestError, NotFoundError } from '@/domain/errors/index.js';
import { TenantThemeModel } from '@/domain/models/tenant-theme.js';
import { TenantModel } from '@/domain/models/tenants.js';
import type { TenantThemeRepository } from '@/domain/repositories/tenant-themes.js';
import type { TenantRepository } from '@/domain/repositories/tenants.js';
import { SaveThemeUseCase } from '@/domain/use-cases/actions/save-theme.js';

export class SaveThemeUseCaseImpl extends BaseUseCaseImpl implements SaveThemeUseCase {
    constructor(
        private readonly themeRepo: TenantThemeRepository,
        private readonly tenantRepo: TenantRepository
    ) {
        super();
    }

    public async execute(params: SaveThemeUseCase.Params): Promise<SaveThemeUseCase.Result> {
        try {
            const { tenantId, ...themeFields } = params;
            const tenantValidation = TenantModel.validateId(tenantId);
            if (tenantValidation.hasError) {
                return left(new BadRequestError(tenantValidation.errorMessages!.join('; ')));
            }
            await this.validateTenantExists(tenantId);
            const model = this.buildAndValidateModel(themeFields);
            await this.themeRepo.save(tenantId, model);
            return right(undefined);
        } catch (error) {
            return left(this.handleError(error));
        }
    }

    private async validateTenantExists(tenantId: string): Promise<void> {
        const tenant = await this.tenantRepo.findById(tenantId);
        if (!tenant) {
            throw new NotFoundError('tenant.notFound');
        }
    }

    private buildAndValidateModel(themeFields: Omit<SaveThemeUseCase.Params, 'tenantId'>): TenantThemeModel {
        const model = TenantThemeModel.from(themeFields);
        const validation = model.validate();
        if (validation.hasError) {
            throw new BadRequestError(validation.errorMessages!.join('; '));
        }
        return model;
    }
}
```

Pontos obrigatórios:
- `extends BaseUseCaseImpl` antes de `implements XxxUseCase`
- `super()` no constructor
- `return left(this.handleError(error))` no catch — nunca `new ServerError(...)` manual

---

## `BaseServiceImpl`

Estruturalmente idêntica à `BaseUseCaseImpl`. O nome sinaliza que o artefato é um service (chamado por outros use cases), não um use case diretamente invocado por controllers.

### Arquivo: `services/base/base.ts`

```typescript
// src/application/services/base/base.ts
import { AbstractError } from '@/domain/errors/abstract-error.js';
import { ServerError } from '@/domain/errors/server-error.js';

export abstract class BaseServiceImpl {
    protected handleError(error: unknown): AbstractError {
        if (error instanceof AbstractError) {
            return error;
        }
        const err = error as Error;
        return new ServerError(err.stack, err.message);
    }
}
```

### Few-shot: service herdando `BaseServiceImpl`

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
            const status = hasErrors ? 'COMPLETED_WITH_ERRORS' : 'COMPLETED';
            await this.dataLoadRepo.updateProgress(dataLoad.id, { status });
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

---

---

## Anti-padrão: tipos e interfaces declarados no arquivo da application layer

Definições de `type`, `interface` ou `enum` **não pertencem** a arquivos da camada de aplicação. Elas devem viver em `domain/use-cases/`, `domain/models/` ou `domain/errors/`.

❌ **Errado — `type` solto no arquivo da application layer:**

```typescript
// src/application/use-cases/actions/checkout.ts  ← ERRADO
import { CheckoutUseCase } from '@/domain/use-cases/actions/checkout.js';

type PipelineContext = {         // ← tipo solto; deve estar em domain/
    session: CartSessionWithItems;
    existingHistory: TransactionHistoryRow[];
    completedSteps: string[];
};

type KitRequestGroupAccumulator = {   // ← idem
    centroOrigem: string;
    items: KitDeliveryRequestService.RequestItem[];
};

export class CheckoutUseCaseImpl extends BaseUseCaseImpl implements CheckoutUseCase {
    // ...
}
```

```typescript
// src/data/use-cases/entity-events/steps/after-read.ts  ← ERRADO
type ProgramQueryResult = {    // ← tipo interno; deve estar no contrato do domain
    id: string;
    isHelicopter: boolean;
};

export class AfterReadStepsUseCaseImpl implements AfterReadStepsUseCase {
    // ...
}
```

✅ **Certo — tipos definidos no namespace do contrato em `domain/`:**

```typescript
// src/domain/use-cases/actions/checkout.ts  ← correto: domain
export namespace CheckoutUseCase {
    export type Params = { /* ... */ };
    export type Result = Either<AbstractError, CheckoutResult>;

    export type PipelineContext = {
        session: CartSessionWithItems;
        existingHistory: TransactionHistoryRow[];
        completedSteps: string[];
    };

    export type KitRequestGroupAccumulator = {
        centroOrigem: string;
        items: KitDeliveryRequestService.RequestItem[];
    };
}

export interface CheckoutUseCase {
    execute(params: CheckoutUseCase.Params): Promise<CheckoutUseCase.Result>;
}
```

```typescript
// src/application/use-cases/actions/checkout.ts  ← limpo: só a implementação
import { CheckoutUseCase } from '@/domain/use-cases/actions/checkout.js';

export class CheckoutUseCaseImpl extends BaseUseCaseImpl implements CheckoutUseCase {
    public async execute(params: CheckoutUseCase.Params): Promise<CheckoutUseCase.Result> {
        // usa CheckoutUseCase.PipelineContext importado do domain
        const context: CheckoutUseCase.PipelineContext = { /* ... */ };
        // ...
    }
}
```

> **Regra de bolso:** se você precisou escrever `type` ou `interface` fora de uma classe em um arquivo de `application/`, esse artefato pertence ao domínio.

---

## Regras

- `BaseUseCaseImpl` e `BaseServiceImpl` são `abstract` — nunca instanciar diretamente.
- O método `handleError` é `protected` — acessível apenas pela subclasse; nunca exposto publicamente.
- Nunca adicionar lógica de negócio à base. Se sentir necessidade, a lógica pertence ao use case ou ao domínio.
- Nunca sobrescrever `handleError` nas subclasses para tratar erros de gateway externo — prefira um método privado específico no use case.
- Constantes de classe (`private readonly NOME = valor`) são permitidas para valores locais fixos. Propriedades mutáveis ou campos não-`readonly` não são permitidos.
