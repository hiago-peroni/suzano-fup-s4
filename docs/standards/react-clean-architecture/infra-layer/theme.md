# Infra layer — Theme

O sistema de tema encapsula toda a configuração visual da aplicação: cores, tipografia, bordas, logos e suporte a dark mode. É composto por quatro arquivos gerados pelo MCP, mais o store Zustand que gerencia o estado do tema.

## Estrutura

```
src/infra/theme/
├── theme-config.ts      → interfaces de configuração (ThemeConfig, ThemeColors, etc.)
├── theme-defaults.ts    → valores padrão (DEFAULT_THEME)
├── theme-provider.tsx   → ThemeAppProvider (React + MUI + CSS custom properties)
└── theme-utils.ts       → buildMuiTheme(), themeToCustomProperties()
```

## `theme-config.ts` — as interfaces

Define a forma do objeto de configuração de tema. Nenhuma lógica — só tipagem:

```typescript
// src/infra/theme/theme-config.ts

export interface ThemeColors {
    primary: string;        primaryLight: string;    primaryDark: string;
    secondary: string;      secondaryLight: string;  secondaryDark: string;
    background: string;     surface: string;         border: string;
    error: string;          warning: string;         success: string;
    info: string;           textPrimary: string;     textSecondary: string;
}

export interface ThemeLogos {
    header: string;   secondary: string;
    login: string;    favicon: string;
}

export interface ThemeTypography {
    fontFamily: string;   fontUrl: string;   baseFontSize: string;
}

export interface ThemeLayout {
    borderRadius: string;   headerHeight: string;   sidebarWidth: string;
}

export interface ThemeConfig {
    colors: ThemeColors;
    logos: ThemeLogos;
    typography: ThemeTypography;
    layout: ThemeLayout;
    darkMode: 'light' | 'dark' | 'system';
    customCss: string;
}

export type ThemeColorKey = keyof ThemeColors;
export type ThemeLogoKey = keyof ThemeLogos;
```

## `theme-defaults.ts` — os valores padrão

```typescript
// src/infra/theme/theme-defaults.ts

import type { ThemeConfig } from '@/infra/theme/theme-config.js';

export const DEFAULT_THEME: ThemeConfig = {
    colors: {
        primary: '#2563eb',      primaryLight: '#60a5fa',   primaryDark: '#1e40af',
        secondary: '#7c3aed',    secondaryLight: '#a78bfa', secondaryDark: '#6d28d9',
        background: '#f7f8fa',   surface: '#ffffff',        border: '#e4e7ec',
        error: '#e5484d',        warning: '#f5a623',        success: '#30a46c',
        info: '#0091ff',         textPrimary: '#11181c',    textSecondary: '#687076'
    },
    logos: { header: '', secondary: '', login: '', favicon: '' },
    typography: {
        fontFamily: 'DM Sans, sans-serif',
        fontUrl: 'https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;600;700&display=swap',
        baseFontSize: '14px'
    },
    layout: { borderRadius: '8px', headerHeight: '52px', sidebarWidth: '220px' },
    darkMode: 'light',
    customCss: ''
};
```

## `theme-utils.ts` — construção do tema MUI e CSS custom properties

```typescript
// src/infra/theme/theme-utils.ts

import { createTheme } from '@mui/material/styles';
import type { ThemeConfig } from '@/infra/theme/theme-config.js';

export function buildMuiTheme(config: ThemeConfig) {
    const isDark = config.darkMode === 'dark';
    return createTheme({
        palette: {
            mode: isDark ? 'dark' : 'light',
            primary: { main: config.colors.primary, light: config.colors.primaryLight, dark: config.colors.primaryDark },
            secondary: { main: config.colors.secondary },
            error: { main: config.colors.error },
            // ...
        },
        typography: { fontFamily: config.typography.fontFamily },
        shape: { borderRadius: parseInt(config.layout.borderRadius) || 8 }
    });
}

export function themeToCustomProperties(config: ThemeConfig): string {
    const lines = [
        ':root {',
        ...Object.entries(config.colors).map(
            ([key, value]) => `  --color-${key.replace(/([A-Z])/g, '-$1').toLowerCase()}: ${value};`
        ),
        `  --font-family: ${config.typography.fontFamily};`,
        `  --border-radius: ${config.layout.borderRadius};`,
        '}'
    ];
    return lines.join('\n');
}
```

## `theme-provider.tsx` — o provider React

O `ThemeAppProvider` é o entry point do sistema de tema. Deve envolver toda a aplicação no `main/index.tsx`:

```typescript
// src/infra/theme/theme-provider.tsx

import CssBaseline from '@mui/material/CssBaseline';
import { ThemeProvider } from '@mui/material/styles';
import { useEffect, useMemo, useState } from 'react';
import type { ReactNode } from 'react';

import { useThemeStore } from '@/infra/stores/theme-store.js';
import { buildMuiTheme, themeToCustomProperties } from '@/infra/theme/theme-utils.js';

export function ThemeAppProvider({ children }: { children: ReactNode }) {
    const { theme, loadTheme } = useThemeStore();
    const [resolvedMode, setResolvedMode] = useState<'light' | 'dark'>('light');

    useEffect(() => { loadTheme(); }, [loadTheme]);

    // Detecta preferência do sistema operacional para dark mode
    useEffect(() => {
        if (theme.darkMode !== 'system') {
            return;
        }
        const mql = window.matchMedia('(prefers-color-scheme: dark)');
        const handler = () => setResolvedMode(mql.matches ? 'dark' : 'light');
        handler();
        mql.addEventListener('change', handler);
        return () => mql.removeEventListener('change', handler);
    }, [theme.darkMode]);

    const muiTheme = useMemo(() => {
        const effectiveConfig = theme.darkMode === 'system'
            ? { ...theme, darkMode: resolvedMode as 'light' | 'dark' }
            : theme;
        return buildMuiTheme(effectiveConfig);
    }, [theme, resolvedMode]);

    // Injeta CSS custom properties no <head> do documento
    useEffect(() => {
        const css = themeToCustomProperties(theme);
        let style = document.getElementById('theme-custom-properties');
        if (!style) {
            style = document.createElement('style');
            style.id = 'theme-custom-properties';
            document.head.appendChild(style);
        }
        style.textContent = css;
    }, [theme]);

    return (
        <ThemeProvider theme={muiTheme}>
            <CssBaseline />
            {children}
        </ThemeProvider>
    );
}
```

## Como usar no `main/index.tsx`

```typescript
// src/main/index.tsx

import React from 'react';
import ReactDOM from 'react-dom/client';
import '@/infra/i18n/index.js';
import { ThemeAppProvider } from '@/infra/theme/theme-provider.js';
import { App } from '@/main/app.js';

ReactDOM.createRoot(document.getElementById('root')!).render(
    <React.StrictMode>
        <ThemeAppProvider>  {/* ← wrapa toda a aplicação */}
            <App />
        </ThemeAppProvider>
    </React.StrictMode>
);
```

## Regras de ouro

1. `ThemeAppProvider` **deve ser o provider mais externo** — aplicado antes do `BrowserRouter` no `index.tsx`.
2. `DEFAULT_THEME` é a referência canônica de valores padrão — sobrescreva via `useThemeStore`, não editando o arquivo.
3. `buildMuiTheme` é chamado apenas pelo `ThemeAppProvider` — outros componentes não precisam chamar diretamente.
4. CSS custom properties são geradas automaticamente a partir do `ThemeConfig` — use `var(--color-primary)` no CSS quando precisar das cores fora do MUI.
5. `theme-config.ts` contém **apenas interfaces** — sem valores concretos (esses ficam em `theme-defaults.ts`).
