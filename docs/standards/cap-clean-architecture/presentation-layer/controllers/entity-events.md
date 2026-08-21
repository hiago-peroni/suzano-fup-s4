# controllers/entity-events/

Controllers para hooks de entidade do CAP — eventos disparados automaticamente antes ou depois de operações CRUD em entidades do serviço. Cada entidade tem sua própria pasta; cada hook, seu próprio arquivo.

## Estrutura de pastas

```
controllers/entity-events/
└── <entidade-kebab-plural>/
    ├── before-create.ts
    ├── before-read.ts
    ├── before-update.ts
    ├── before-delete.ts
    ├── after-read.ts
    ├── after-create.ts
    ├── after-delete.ts
    └── index.ts             → reexport de todos os controllers da entidade
```

Crie apenas os arquivos dos hooks que a entidade realmente usa. Não crie arquivos de hook vazio.

---

## Naming

| Elemento | Convenção | Exemplo |
|---|---|---|
| Pasta da entidade | `kebab-case-plural` | `price-lists/`, `tenant-erp-connections/` |
| Arquivo do hook | `<evento>-<ação>.ts` | `before-update.ts`, `after-read.ts` |
| Classe | `<Evento><Ação><Entidade>Controller` | `BeforeUpdatePriceListController` |

---

## Hooks suportados e correspondência CDS

| Arquivo | Registro em `routes/index.ts` |
|---|---|
| `before-create.ts` | `service.before('CREATE', 'Entity', ...)` |
| `before-read.ts` | `service.before('READ', 'Entity', ...)` |
| `before-update.ts` | `service.before('UPDATE', 'Entity', ...)` |
| `before-delete.ts` | `service.before('DELETE', 'Entity', ...)` |
| `after-read.ts` | `service.after('READ', 'Entity', ...)` |
| `after-create.ts` | `service.after('CREATE', 'Entity', ...)` |
| `after-delete.ts` | `service.after('DELETE', 'Entity', ...)` |

---

## Shape canônico — before-*

Hooks `before-*` recebem o `request` bruto do CAP (o payload ainda pode ser modificado neste ponto).

```typescript
// src/presentation/controllers/entity-events/price-lists/before-update.ts
import type { BeforeUpdatePriceListUseCase } from '@/domain/use-cases/entity-events/price-lists/before-update.js';
import { BaseController, type BaseControllerResponse } from '@/presentation/controllers/base/controller.js';

export class BeforeUpdatePriceListController extends BaseController {
    constructor(private readonly useCase: BeforeUpdatePriceListUseCase) {
        super();
    }

    public async handle(params: BeforeUpdatePriceListUseCase.Params): Promise<BaseControllerResponse> {
        const result = await this.useCase.execute(params);
        if (result.isLeft()) {
            return this.error(result.value.code, result.value.toErrorDetails());
        }
        return this.success(undefined);
    }
}
```

A rota monta os `params` a partir de `request.data` antes de chamar o controller:

```typescript
// src/main/routes/index.ts (trecho de registro do hook)
async function handleBeforeUpdatePriceList(request: any) {
    const params: BeforeUpdatePriceListUseCase.Params = {
        priceList: request.data,
        userId: request.user?.id,
    };
    const result = await beforeUpdatePriceListController.handle(params);
    if (result.status >= 400) {
        return request.reject(result.errorData);
    }
}
```

Observe que em `before-*` bem-sucedido a rota **não retorna** `result.data` — o CAP prossegue com o `request` original. Retornar dados de um `before-*` substitui o payload, o que raramente é o comportamento desejado.

---

## Shape canônico — after-*

Hooks `after-*` recebem os dados **após** a operação do banco. A rota passa o array de resultados para o controller.

```typescript
// src/presentation/controllers/entity-events/orders/after-read.ts
import type { AfterReadOrderUseCase } from '@/domain/use-cases/entity-events/orders/after-read.js';
import { BaseController, type BaseControllerResponse } from '@/presentation/controllers/base/controller.js';

export class AfterReadOrderController extends BaseController {
    constructor(private readonly useCase: AfterReadOrderUseCase) {
        super();
    }

    public async handle(params: AfterReadOrderUseCase.Params): Promise<BaseControllerResponse> {
        const result = await this.useCase.execute(params);
        if (result.isLeft()) {
            return this.error(result.value.code, result.value.toErrorDetails());
        }
        return this.success(result.value);
    }
}
```

```typescript
// src/main/routes/index.ts (trecho de registro do after-read)
async function handleAfterReadOrder(data: any[], request: any) {
    const params: AfterReadOrderUseCase.Params = { data, userId: request.user?.id };
    const result = await afterReadOrderController.handle(params);
    if (result.status >= 400) {
        return request.reject(result.errorData);
    }
}
```

---

## `index.ts` de reexport

Cada pasta de entidade expõe um `index.ts` que reexporta todos os seus controllers. Isso simplifica os imports em `main/factories/controllers/entity-events/`:

```typescript
// src/presentation/controllers/entity-events/price-lists/index.ts
export { BeforeUpdatePriceListController } from './before-update.js';
export { BeforeCreatePriceListController } from './before-create.js';
```

---

## Anti-padrão — mutação de `request.data` no controller

❌ **Nunca faça isso no controller:**

```typescript
// ERRADO — mutação de request dentro do controller
public async handle(request: any): Promise<BaseControllerResponse> {
    const result = await this.useCase.execute(request.data);
    if (result.isLeft()) {
        return this.error(result.value.code, result.value.toErrorDetails());
    }
    Object.assign(request.data, result.value); // ← efeito colateral no controller
    return this.success(undefined);
}
```

✅ **Faça assim — o use case retorna o payload enriquecido e a rota aplica a mutação:**

```typescript
// CERTO — controller retorna os dados; a rota decide o que fazer com eles
public async handle(params: XxxUseCase.Params): Promise<BaseControllerResponse> {
    const result = await this.useCase.execute(params);
    if (result.isLeft()) {
        return this.error(result.value.code, result.value.toErrorDetails());
    }
    return this.success(result.value); // ← dados enriquecidos como resposta
}

// Em routes/index.ts:
async function handleBeforeCreateX(request: any) {
    const params = { data: request.data };
    const result = await beforeCreateXController.handle(params);
    if (result.status >= 400) {
        return request.reject(result.errorData);
    }
    Object.assign(request.data, result.data); // ← mutação explícita na rota
}
```

---

## Estrutura de pastas (referência)

```
controllers/entity-events/
├── orders/
│   ├── before-create.ts
│   ├── before-update.ts
│   ├── after-read.ts
│   └── index.ts
├── price-lists/
│   ├── before-update.ts
│   └── index.ts
└── tenant-erp-connections/
    ├── before-create.ts
    ├── before-update.ts
    ├── before-delete.ts
    └── index.ts
```
