# Shared layer — Constants

O arquivo `routes.ts` é a **única fonte de verdade para os caminhos de rota** da aplicação. Centraliza as strings de URL e fornece funções auxiliares para construir rotas com parâmetros dinâmicos.

## Estrutura

```
src/shared/constants/
└── routes.ts    → ROUTES + funções to*()
```

## O arquivo gerado pelo MCP

```typescript
// src/shared/constants/routes.ts

export const ROUTES = {
    HOME: '/',
    SETTINGS_THEME: '/settings/theme'
} as const;
```

## Padrão expandido com rotas de feature

```typescript
// src/shared/constants/routes.ts

export const ROUTES = {
    HOME: '/',
    SALES_ORDERS: '/sales-orders',
    SALES_ORDER_DETAIL: '/sales-orders/:id',
    PRODUCTS: '/products',
    SETTINGS_THEME: '/settings/theme'
} as const;

// Funções auxiliares para construir rotas com parâmetros
export const toSalesOrderDetail = (id: string) => `/sales-orders/${id}`;
export const toProductDetail = (id: string) => `/products/${id}`;
```

O `as const` garante que os valores sejam tipos literais (`'/sales-orders'`), não `string` genérico.

## Onde cada elemento é usado

| Elemento | Usado em |
|---|---|
| `ROUTES.*` (paths sem parâmetros) | `app.tsx` (definição de rota), `AppShell` (navegação no menu) |
| `ROUTES.*` (paths com `:param`) | `app.tsx` (definição de rota com `useParams`) |
| `toXxx(id)` | Componentes que navegam para detalhe (`useNavigate`) |

## Exemplos de uso

### Em `app.tsx` — definição de rota

```typescript
import { ROUTES } from '@/shared/constants/routes.js';

<Route path={ROUTES.SALES_ORDERS} element={<SalesOrdersPageFactory />} />
<Route path={ROUTES.SALES_ORDER_DETAIL} element={<SalesOrderDetailPageFactory />} />
```

### No `AppShell` — item de menu

```typescript
import { ROUTES } from '@/shared/constants/routes.js';

<ListItemButton
    selected={location.pathname === ROUTES.SALES_ORDERS}
    onClick={() => navigate(ROUTES.SALES_ORDERS)}
>
    <ListItemText primary={translate('menu.salesOrders')} />
</ListItemButton>
```

### Em um componente — link para detalhe

```typescript
import { useNavigate } from 'react-router-dom';
import { toSalesOrderDetail } from '@/shared/constants/routes.js';

const navigate = useNavigate();

<Button onClick={() => navigate(toSalesOrderDetail(order.id))}>
    {translate('salesOrders.viewDetail')}
</Button>
```

### Em um hook — verificar rota atual

```typescript
import { useLocation } from 'react-router-dom';
import { ROUTES } from '@/shared/constants/routes.js';

const location = useLocation();
const isOnSalesOrders = location.pathname === ROUTES.SALES_ORDERS;
```

## Adicionando novas rotas

1. Adicione a constante no objeto `ROUTES`
2. Se a rota tem parâmetro dinâmico, adicione a função auxiliar `toXxx()`
3. Registre a rota em `main/app.tsx`
4. Adicione item no menu do `AppShell` se necessário

## Anti-padrões

❌ **String literal de rota fora do arquivo `routes.ts`:**
```typescript
navigate('/sales-orders'); // ← use navigate(ROUTES.SALES_ORDERS)
<Route path="/sales-orders" /> // ← use ROUTES.SALES_ORDERS
```

❌ **Construir URL com template string fora da função auxiliar:**
```typescript
navigate(`/sales-orders/${id}`); // ← use navigate(toSalesOrderDetail(id))
```

❌ **Usar `ROUTES` como tipo genérico:**
```typescript
type Route = string; // ← use typeof ROUTES[keyof typeof ROUTES] para tipo derivado
```

## Regras de ouro

1. **`ROUTES` é o único lugar** onde strings de URL são definidas — sem duplicatas.
2. **`as const`** é obrigatório — garante tipos literais para type safety.
3. Rotas com parâmetros dinâmicos têm **função auxiliar `toXxx(id)`** — sem construção manual de string.
4. Não criar `ROUTES` separados por feature — um único objeto centralizado.
