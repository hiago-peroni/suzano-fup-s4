# routes/

Ponto de entrada do CAP: `cds-ts serve --from ./src/main/routes`. Contém dois arquivos fixos.

## index.cds — definição do serviço OData

Define o serviço CAP seguindo a separação obrigatória em três blocos:

1. **`service MyService { ... }`** — somente entidades (projections). Nenhuma action ou function aqui.
2. **`extend service MyService with { // actions }`** — um bloco de extend exclusivo para actions.
3. **`extend service MyService with { // functions }`** — um bloco de extend exclusivo para functions.

Essa separação mantém o `index.cds` legível à medida que o serviço cresce e deixa claro o tipo de cada operação sem precisar ler a assinatura.

```cds
// src/main/routes/index.cds
using { db.models } from '../../../../db/models';
using { db.types } from '../../../../db/types';

@path: '/my-service'
@requires: 'authenticated-user'
service MyService {
    // Entidades
    entity Orders as projection on models.Orders;
    entity OrderItems as projection on models.OrderItems;
}

// Actions (modificam estado ou consultas complexas com vários parâmetros de entrada)
extend service MyService with {
    action confirmOrder(orderId: UUID) returns Boolean;
    action bulkLoadOrders(payload: types.BulkLoadOrders.Payload) returns returns types.BulkLoadOrders.Result;
}

// Functions (somente leitura)
extend service MyService with {        
    function getOrderStats(month: Integer) returns types.OrderStats;
}
```

---

## index.ts — wiring dos handlers

### Estrutura obrigatória

```typescript
// src/main/routes/index.ts
import '../config/module-alias';                                    // 1. SEMPRE primeiro

import { Service } from '@sap/cds';

import { confirmOrderController } from '@/main/factories/controllers/actions/confirm-order';
import { getOrderStatsController } from '@/main/factories/controllers/functions/get-order-stats';
import { beforeUpdateOrderController } from '@/main/factories/controllers/entity-events/orders/before-update';
import { translator } from '@/main/factories/utils/translator';

// 2. Funções de registro (opcional, organiza handlers por tipo)
function registerActions(service: Service): void {
    service.on('confirmOrder', async (request: any) => {
        return translator.withLanguage(request._language, () => handleConfirmOrder(request));
    });
}

function registerFunctions(service: Service): void {
    service.on('getOrderStats', async (request: any) => {
        return translator.withLanguage(request._language, () => handleGetOrderStats(request));
    });
}

function registerEntityHandlers(service: Service): void {
    service.before('UPDATE', 'Orders', async (request: any) => {
        return translator.withLanguage(request._language, () => handleBeforeUpdateOrder(request));
    });
}

// 3. Export default — contrato CAP
export default (service: Service) => {
    service.before('*', async (request: any) => {           // middleware i18n
        const raw = request?.headers['accept-language']?.split(',')[0] || 'en';
        request._language = raw.split('-')[0];
    });

    registerActions(service);
    registerFunctions(service);
    registerEntityHandlers(service);
};

// 4. Handle* — um por handler, delegam ao controller
async function handleConfirmOrder(request: any) {
    const result = await confirmOrderController.handle(request);
    if (result.status >= 400) {
        return request.reject(result.errorData);
    }
    return result.data;
}

async function handleGetOrderStats(request: any) {
    const result = await getOrderStatsController.handle(request);
    if (result.status >= 400) {
        return request.reject(result.errorData);
    }
    return result.data;
}

async function handleBeforeUpdateOrder(request: any) {
    const result = await beforeUpdateOrderController.handle(request);
    if (result.status >= 400) {
        return request.reject(result.errorData);
    }
}
```

### Regras do `index.ts`

1. **`module-alias` sempre na primeira linha** — antes de qualquer import de `@/`.
2. **Importar singletons** (ex.: `confirmOrderController`), não funções factory (ex.: `makeConfirmOrderController`).
3. **`export default (service: Service) => { ... }`** é o contrato do CAP — nunca alterar a assinatura.
4. **Middleware de idioma antes de tudo** — `service.before('*', ...)` com `request._language`.
5. **Sem lógica de negócio** — as funções `handle*` só chamam `.handle()`, verificam status e rejeitam ou retornam `result.data`.

---

## Anti-padrão — lógica de negócio no routes

❌ **Nunca faça isso:**

```typescript
// ERRADO — lógica de negócio inline no routes/index.ts
service.before('CREATE', 'CartItems', async (request: any) => {
    const sessionId = request.data?.session_ID ?? '';
    const rows = await cds.run(
        cds.ql.SELECT.from('db.models.CartSessions').where({ id: sessionId })
    );
    if (!rows.length) return;
    const scenario = getScenario({ source: rows[0].source_id });
    request.data.qtyPet = applyStockBehavior(request.data, scenario).qtyPet;
    // ... 40 linhas de lógica ...
});
```

✅ **Faça assim:**

```typescript
// CERTO — delega para controller que delega para use-case
service.before('CREATE', 'CartItems', async (request: any) => {
    return translator.withLanguage(request._language, () => handleBeforeCreateCartItem(request));
});

async function handleBeforeCreateCartItem(request: any) {
    const result = await beforeCreateCartItemController.handle(request);
    if (result.status >= 400) {
        return request.reject(result.errorData);
    }
}
```
