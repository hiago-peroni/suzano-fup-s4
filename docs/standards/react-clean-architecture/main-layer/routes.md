# Main layer — Routes

O roteamento usa **react-router-dom v7** com `BrowserRouter`. As rotas são definidas em `app.tsx`, usando as constantes de `shared/constants/routes.ts` e as page factories como elementos. O `index.tsx` é o entry point que inicializa o React e aplica os providers globais.

## Estrutura

```
src/main/
├── index.tsx    → entry point (ReactDOM.createRoot + providers)
├── app.tsx      → BrowserRouter + Routes + Route com factories
└── routes.tsx   → re-export de App (convenção de barrel)
```

## `index.tsx` — entry point

```typescript
// src/main/index.tsx

import React from 'react';
import ReactDOM from 'react-dom/client';

import '@/infra/i18n/index.js';                                              // ← i18n antes de tudo
import { ThemeAppProvider } from '@/infra/theme/theme-provider.js';
import { App } from '@/main/app.js';

ReactDOM.createRoot(document.getElementById('root')!).render(
    <React.StrictMode>
        <ThemeAppProvider>                                    {/* ← provider mais externo */}
            <App />
        </ThemeAppProvider>
    </React.StrictMode>
);
```

**Ordem obrigatória:**
1. `import '@/infra/i18n/index.js'` — inicializa i18next antes de qualquer componente
2. `ThemeAppProvider` — wrapa tudo para garantir tema disponível em toda a árvore
3. `App` — contém o `BrowserRouter` e as rotas

## `app.tsx` — rotas da aplicação

```typescript
// src/main/app.tsx

import { BrowserRouter, Route, Routes, Navigate } from 'react-router-dom';

import { AppShell } from '@/presentation/layouts/app-shell/app-shell.js';
import { ThemeSettingsPage } from '@/presentation/pages/settings/theme-settings.js';
import { SalesOrdersPageFactory } from '@/main/factories/pages/sales-orders-page-factory.js';
import { SalesOrderDetailPageFactory } from '@/main/factories/pages/sales-order-detail-page-factory.js';
import { ROUTES } from '@/shared/constants/routes.js';

export function App() {
    return (
        <BrowserRouter>
            <Routes>
                <Route element={<AppShell />}>
                    <Route path={ROUTES.HOME} element={<Navigate to={ROUTES.SALES_ORDERS} replace />} />
                    <Route path={ROUTES.SALES_ORDERS} element={<SalesOrdersPageFactory />} />
                    <Route path={ROUTES.SALES_ORDER_DETAIL} element={<SalesOrderDetailPageFactory />} />
                    <Route path={ROUTES.SETTINGS_THEME} element={<ThemeSettingsPage />} />
                </Route>
            </Routes>
        </BrowserRouter>
    );
}
```

## Como adicionar uma nova rota

1. Adicione a constante em `shared/constants/routes.ts`:
```typescript
export const ROUTES = {
    HOME: '/',
    SALES_ORDERS: '/sales-orders',
    SALES_ORDER_DETAIL: '/sales-orders/:id',
    PRODUCTS: '/products',                    // ← nova rota
    SETTINGS_THEME: '/settings/theme'
} as const;
```

2. Crie a page factory em `main/factories/pages/`:
```typescript
// src/main/factories/pages/products-page-factory.tsx
export const ProductsPageFactory: React.FC = () => {
    const controller = useMemo(() => makeProductsController(httpClient), []);
    return <ProductsPage controller={controller} />;
};
```

3. Registre a rota em `app.tsx`:
```typescript
<Route path={ROUTES.PRODUCTS} element={<ProductsPageFactory />} />
```

4. Adicione o item no menu do `AppShell` (`presentation/layouts/app-shell/app-shell.tsx`).

## Rotas sem factory (páginas estáticas)

Páginas que não precisam de controller (settings de tema, páginas de erro) podem ser usadas diretamente:

```typescript
<Route path={ROUTES.SETTINGS_THEME} element={<ThemeSettingsPage />} />
```

## `routes.tsx` — re-export

```typescript
// src/main/routes.tsx
export { App } from '@/main/app.js';
```

Arquivo de convenção — re-exporta `App` para manter compatibilidade com imports que esperam o módulo `routes`.

## Anti-padrões

❌ **Definir rotas fora do `app.tsx`:**
```typescript
// Rotas não devem ser espalhadas por componentes de feature
export function SalesOrdersSection() {
    return <Route path="/sales-orders" element={<SalesOrdersPage />} />;
}
```

❌ **Usar string literal como path em vez de `ROUTES`:**
```typescript
<Route path="/sales-orders" element={...} />  // ← use ROUTES.SALES_ORDERS
```

❌ **Importar ThemeAppProvider dentro de `app.tsx` (duplicação):**
```typescript
// ThemeAppProvider pertence ao index.tsx, não ao App
function App() {
    return (
        <ThemeAppProvider>  {/* ← já está no index.tsx */}
            <BrowserRouter>...</BrowserRouter>
        </ThemeAppProvider>
    );
}
```

## Regras de ouro

1. **`@/infra/i18n/index.js` é sempre o primeiro import** em `index.tsx`.
2. **`ThemeAppProvider` wrapa todo o `App`** — nunca aplicado parcialmente.
3. **Todas as rotas usam constantes de `ROUTES`** — nunca strings literais.
4. **Page factories são os elementos das rotas** — nunca páginas diretamente (exceto páginas sem controller).
5. O `AppShell` é o elemento pai de todas as rotas da aplicação — criando um layout aninhado (`<Route element={<AppShell />}>`).
