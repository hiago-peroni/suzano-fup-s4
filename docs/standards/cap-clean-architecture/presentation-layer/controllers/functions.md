# controllers/functions/

Controllers para operações `function` do CDS — consultas com retorno tipado, sem efeitos colaterais. Mapeam 1:1 com cada `function` declarada no `index.cds`.

O shape é idêntico ao das actions. A diferença está no tipo de operação CDS (somente leitura) e, consequentemente, no uso de `service.on` com a mesma assinatura de actions — não há `service.before`/`after` para functions.

## Regras

- Um arquivo por function; mesmo nome nos arquivos de `main/factories/controllers/functions/` e `main/factories/use-cases/functions/`.
- A rota monta `UseCase.Params` antes de chamar o controller.
- `handle` recebe `UseCase.Params`; não acessa `request: any` diretamente.

---

## Shape canônico

```typescript
// src/presentation/controllers/functions/catalog-home.ts
import type { CatalogHomeUseCase } from '@/domain/use-cases/functions/catalog-home.js';
import { BaseController, type BaseControllerResponse } from '@/presentation/controllers/base/controller.js';

export class CatalogHomeController extends BaseController {
    constructor(private readonly useCase: CatalogHomeUseCase) {
        super();
    }

    public async handle(params: CatalogHomeUseCase.Params): Promise<BaseControllerResponse> {
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
// src/main/routes/index.ts (trecho do handler da function)
async function handleCatalogHome(request: any) {
    const params: CatalogHomeUseCase.Params = {
        tenantId: request.user?.attr?.tenantId,
        userId: request.user?.id,
        language: request._language,
    };
    const result = await catalogHomeController.handle(params);
    if (result.status >= 400) {
        return request.reject(result.errorData);
    }
    return result.data;
}
```

---

## Variação: function sem parâmetros de entrada

Quando a function não recebe parâmetros do cliente (ex.: consulta de contexto do usuário logado), `UseCase.Params` pode ser vazio ou omitido:

```typescript
// src/presentation/controllers/functions/get-logged-user.ts
export class GetLoggedUserController extends BaseController {
    constructor(private readonly useCase: GetLoggedUserUseCase) {
        super();
    }

    public async handle(params: GetLoggedUserUseCase.Params): Promise<BaseControllerResponse> {
        const result = await this.useCase.execute(params);
        if (result.isLeft()) {
            return this.error(result.value.code, result.value.toErrorDetails());
        }
        return this.success(result.value);
    }
}
```

---

## Estrutura de pastas (referência)

```
controllers/functions/
├── catalog-home.ts
├── catalog-search.ts
├── get-logged-user.ts
├── get-order-stats.ts
└── get-part-numbers.ts
```

---

## Naming

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo | `kebab-case.ts` | `catalog-home.ts` |
| Classe | `PascalCase` + `Controller` | `CatalogHomeController` |
| Parâmetro de `handle` | `params: XxxUseCase.Params` | `params: CatalogHomeUseCase.Params` |
