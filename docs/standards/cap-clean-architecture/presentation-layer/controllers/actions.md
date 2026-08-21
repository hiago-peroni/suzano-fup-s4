# controllers/actions/

Controllers para operações `action` do CDS — operações de negócio que modificam estado ou executam lógica complexa. Mapeam 1:1 com cada `action` declarada no `index.cds`.

## Regras

- Um arquivo por action; mesmo nome do arquivo correspondente em `main/factories/controllers/actions/` e `main/factories/use-cases/actions/`.
- A rota em `main/routes/index.ts` monta `UseCase.Params` a partir de `request.data` **antes** de chamar o controller — o controller não acessa `request` diretamente.
- `handle` recebe `UseCase.Params` pronto; nunca recebe `request: any` nas actions.

---

## Shape canônico

```typescript
// src/presentation/controllers/actions/approve-price-list.ts
import type { ApprovePriceListUseCase } from '@/domain/use-cases/actions/approve-price-list.js';
import { BaseController, type BaseControllerResponse } from '@/presentation/controllers/base/controller.js';

export class ApprovePriceListController extends BaseController {
    constructor(private readonly useCase: ApprovePriceListUseCase) {
        super();
    }

    public async handle(params: ApprovePriceListUseCase.Params): Promise<BaseControllerResponse> {
        const result = await this.useCase.execute(params);
        if (result.isLeft()) {
            return this.error(result.value.code, result.value.toErrorDetails());
        }
        return this.success(result.value);
    }
}
```

Como a rota monta e entrega os params:

```typescript
// src/main/routes/index.ts (trecho do handler da action)
async function handleApprovePriceList(request: any) {
    const params: ApprovePriceListUseCase.Params = {
        priceListId: request.data.priceListId,
        userId: request.user?.id,
    };
    const result = await approvePriceListController.handle(params);
    if (result.status >= 400) {
        return request.reject(result.errorData);
    }
    return result.data;
}
```

---

## Variação: action sem retorno de dados (fire-and-forget)

Quando a action não retorna payload ao cliente, o use case retorna `Either<AbstractError, void>`:

```typescript
// src/presentation/controllers/actions/release-loss-provisions.ts
export class ReleaseLossProvisionsController extends BaseController {
    constructor(private readonly useCase: ReleaseLossProvisionsUseCase) {
        super();
    }

    public async handle(params: ReleaseLossProvisionsUseCase.Params): Promise<BaseControllerResponse> {
        const result = await this.useCase.execute(params);
        if (result.isLeft()) {
            return this.error(result.value.code, result.value.toErrorDetails());
        }
        return this.success(undefined);
    }
}
```

---

## Variação: sub-ação (before-execute)

Quando uma action possui uma etapa de validação explícita (`before-execute`) separada da execução principal, cada etapa tem seu próprio controller e use case. A pasta da action agrupa os dois:

```
controllers/actions/save-loss-provisions/
├── before-execute.ts    → BeforeExecuteSaveLossProvisionsController
├── save-loss-provisions.ts → SaveLossProvisionsController
└── index.ts             → reexport dos dois
```

```typescript
// src/presentation/controllers/actions/save-loss-provisions/before-execute.ts
export class BeforeExecuteSaveLossProvisionsController extends BaseController {
    constructor(private readonly useCase: BeforeExecuteSaveLossProvisionsUseCase) {
        super();
    }

    public async handle(params: BeforeExecuteSaveLossProvisionsUseCase.Params): Promise<BaseControllerResponse> {
        const result = await this.useCase.execute(params);
        if (result.isLeft()) {
            return this.error(result.value.code, result.value.toErrorDetails());
        }
        return this.success(undefined);
    }
}
```

Use sub-ação somente quando o ciclo de vida da action no CDS exige dois handlers distintos registrados em `service.before` e `service.on`. Não crie `before-execute` por convenção — só quando necessário.

---

## Estrutura de pastas (referência)

```
controllers/actions/
├── approve-price-list.ts
├── checkout.ts
├── mass-reversal-loss-provisions.ts
├── save-loss-provisions/
│   ├── before-execute.ts
│   ├── save-loss-provisions.ts
│   └── index.ts
└── upload-data-load.ts
```

---

## Naming

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo | `kebab-case.ts` | `approve-price-list.ts` |
| Classe | `PascalCase` + `Controller` | `ApprovePriceListController` |
| Parâmetro de `handle` | `params: XxxUseCase.Params` | `params: ApprovePriceListUseCase.Params` |
