# Presentation layer — Layouts

Layouts são **wrappers de estrutura de página** que definem a moldura visual da aplicação: sidebar, header, área de conteúdo. O layout gerado pelo MCP é `AppShell`, que usa `<Outlet />` do react-router-dom para renderizar as páginas dentro da estrutura.

## Estrutura

```
src/presentation/layouts/
└── app-shell/
    └── app-shell.tsx    → AppShell (gerado pelo MCP)
```

## O layout gerado pelo MCP — `AppShell`

O `AppShell` implementa a estrutura padrão com:
- **AppBar** (header fixo com título e toggle dark/light)
- **Drawer** responsivo (sidebar permanente em desktop, temporário em mobile)
- **Área de conteúdo** com `<Outlet />` para renderização das páginas

```typescript
// src/presentation/layouts/app-shell/app-shell.tsx (estrutura simplificada)

import AppBar from '@mui/material/AppBar';
import Box from '@mui/material/Box';
import Drawer from '@mui/material/Drawer';
import { useState } from 'react';
import { Outlet, useNavigate } from 'react-router-dom';

import { useTranslate } from '@/presentation/contexts/translate-context.js';
import { useThemeContext } from '@/presentation/contexts/theme-context.js';
import { ROUTES } from '@/shared/constants/routes.js';

const DRAWER_WIDTH = 240;

export function AppShell() {
    const translate = useTranslate();
    const [mobileOpen, setMobileOpen] = useState(false);
    const navigate = useNavigate();
    const { theme, setDarkMode } = useThemeContext();
    const isDark = theme.darkMode === 'dark';

    return (
        <Box sx={{ display: 'flex', minHeight: '100vh' }}>
            {/* AppBar fixo no topo */}
            <AppBar position="fixed" sx={{ zIndex: (th) => th.zIndex.drawer + 1 }}>
                {/* Título + toggle dark/light */}
            </AppBar>

            {/* Drawer de navegação */}
            <Box component="nav" sx={{ width: { sm: DRAWER_WIDTH } }}>
                <Drawer variant="permanent" open>
                    {/* Links de navegação com ROUTES */}
                </Drawer>
            </Box>

            {/* Área de conteúdo */}
            <Box component="main" sx={{ flexGrow: 1 }}>
                <Outlet /> {/* ← páginas renderizam aqui */}
            </Box>
        </Box>
    );
}
```

## Como o AppShell é usado nas rotas

O `AppShell` é o **elemento pai** de todas as rotas da aplicação. As páginas são definidas como filhas:

```typescript
// src/main/app.tsx

import { BrowserRouter, Route, Routes } from 'react-router-dom';
import { AppShell } from '@/presentation/layouts/app-shell/app-shell.js';
import { SalesOrdersPageFactory } from '@/main/factories/pages/sales-orders-page-factory.js';
import { ROUTES } from '@/shared/constants/routes.js';

function App() {
    return (
        <BrowserRouter>
            <Routes>
                <Route element={<AppShell />}>  {/* ← layout wrapa todas as páginas */}
                    <Route path={ROUTES.SALES_ORDERS} element={<SalesOrdersPageFactory />} />
                    <Route path={ROUTES.SALES_ORDER_DETAIL} element={<SalesOrderDetailPageFactory />} />
                    <Route path={ROUTES.SETTINGS_THEME} element={<ThemeSettingsPage />} />
                </Route>
            </Routes>
        </BrowserRouter>
    );
}
```

## Adicionando itens ao menu da sidebar

O menu de navegação do `AppShell` usa os `ROUTES` de `shared/constants/routes.ts`. Para adicionar um item:

1. Adicione a rota em `shared/constants/routes.ts`
2. Adicione o item na lista do drawer em `app-shell.tsx`

```typescript
// Dentro do drawerContent em app-shell.tsx
<ListItemButton
    selected={isActive(ROUTES.SALES_ORDERS)}
    onClick={() => navigate(ROUTES.SALES_ORDERS)}
>
    <ListItemIcon><ShoppingCartIcon /></ListItemIcon>
    <ListItemText primary={translate('menu.salesOrders')} />
</ListItemButton>
```

## Quando criar um novo layout

O `AppShell` cobre a grande maioria dos casos. Crie um novo layout apenas quando:

- Uma área da aplicação tem estrutura **completamente diferente** (ex.: tela de login sem sidebar)
- Uma sub-área precisa de **toolbar próprio** dentro do conteúdo

```typescript
// src/presentation/layouts/auth-layout/auth-layout.tsx

import Box from '@mui/material/Box';
import { Outlet } from 'react-router-dom';

export function AuthLayout() {
    return (
        <Box sx={{ display: 'flex', justifyContent: 'center', alignItems: 'center', minHeight: '100vh' }}>
            <Outlet />
        </Box>
    );
}
```

```typescript
// Uso no app.tsx
<Routes>
    <Route element={<AuthLayout />}>
        <Route path={ROUTES.LOGIN} element={<LoginPage />} />
    </Route>
    <Route element={<AppShell />}>
        <Route path={ROUTES.HOME} element={<HomePage />} />
    </Route>
</Routes>
```

## Anti-padrões

❌ **Lógica de negócio dentro do layout:**
```typescript
export function AppShell() {
    const { user } = useAuthStore();
    if (!user.hasPermission('admin')) {
        // ← verificação de permissão não vai no layout
    }
}
```

❌ **Layout recebendo dados de feature via props:**
```typescript
export function AppShell({ salesOrders, onLoadOrders }: Props) {
    // ← layout só gerencia estrutura visual
}
```

## Regras de ouro

1. O `AppShell` gerado pelo MCP é o layout padrão — use-o sem modificações estruturais para novas features.
2. Adicionar itens ao menu = adicionar `ListItemButton` no `drawerContent` + a rota correspondente em `ROUTES`.
3. Layouts **não gerenciam dados de feature** — só navegação e estrutura visual.
4. `<Outlet />` é o único ponto de injeção de conteúdo no layout.
5. Novos layouts são criados em `presentation/layouts/<nome>/` seguindo o mesmo padrão de pasta.
