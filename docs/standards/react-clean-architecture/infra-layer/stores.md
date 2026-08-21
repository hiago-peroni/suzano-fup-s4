# Infra layer — Stores (Zustand)

Stores Zustand gerenciam **estado global persistente ou transversal** — aquele que precisa ser acessado por múltiplas partes da aplicação sem passar por props. O store gerado pelo MCP é `useThemeStore`; novos stores seguem o mesmo padrão.

## Estrutura

```
src/infra/stores/
└── theme-store.ts      → useThemeStore (gerado pelo MCP)
└── <nome>-store.ts     → outros stores adicionados pelo time
```

## Quando usar store vs. estado local

| Tipo de estado | Onde colocar |
|---|---|
| Dados de uma feature específica (lista de pedidos, loading de página) | Estado local no **hook** da feature |
| Tema visual, modo dark/light | **Store** (`useThemeStore`) |
| Dados do usuário autenticado | **Store** (`useAuthStore`) |
| Preferências globais do usuário | **Store** |
| Cache de dados com escopo de tela | Estado local no hook |

> **Regra:** se o estado é usado por componentes em rotas diferentes ou por providers globais, é um store. Se é usado apenas na árvore de um página/feature, é estado local no hook.

## O store gerado pelo MCP — `useThemeStore`

```typescript
// src/infra/stores/theme-store.ts

import { create } from 'zustand';
import type { ThemeColorKey, ThemeConfig, ThemeLogoKey } from '@/infra/theme/theme-config.js';
import { DEFAULT_THEME } from '@/infra/theme/theme-defaults.js';

const STORAGE_KEY = 'app-theme';

interface ThemeStore {
    theme: ThemeConfig;
    isLoading: boolean;
    isDirty: boolean;
    loadTheme: () => Promise<void>;
    saveTheme: () => Promise<void>;
    resetTheme: () => void;
    updateColor: (key: ThemeColorKey, value: string) => void;
    updateLogo: (key: ThemeLogoKey, value: string) => void;
    updateTypography: (key: 'fontFamily' | 'fontUrl' | 'baseFontSize', value: string) => void;
    updateLayout: (key: 'borderRadius' | 'headerHeight' | 'sidebarWidth', value: string) => void;
    setDarkMode: (mode: 'light' | 'dark' | 'system') => void;
    setCustomCss: (css: string) => void;
    applyPreset: (colors: ThemeConfig['colors']) => void;
}

export const useThemeStore = create<ThemeStore>((set, get) => ({
    theme: { ...DEFAULT_THEME },
    isLoading: false,
    isDirty: false,

    loadTheme: async () => {
        set({ isLoading: true });
        try {
            const stored = localStorage.getItem(STORAGE_KEY);
            if (stored) {
                set({ theme: { ...DEFAULT_THEME, ...JSON.parse(stored) }, isDirty: false });
            }
        } finally {
            set({ isLoading: false });
        }
    },

    saveTheme: async () => {
        localStorage.setItem(STORAGE_KEY, JSON.stringify(get().theme));
        set({ isDirty: false });
    },

    resetTheme: () => set({ theme: { ...DEFAULT_THEME }, isDirty: true }),

    updateColor: (key, value) =>
        set((s) => ({ theme: { ...s.theme, colors: { ...s.theme.colors, [key]: value } }, isDirty: true })),
    updateLogo: (key, value) =>
        set((s) => ({ theme: { ...s.theme, logos: { ...s.theme.logos, [key]: value } }, isDirty: true })),
    updateTypography: (key, value) =>
        set((s) => ({ theme: { ...s.theme, typography: { ...s.theme.typography, [key]: value } }, isDirty: true })),
    updateLayout: (key, value) =>
        set((s) => ({ theme: { ...s.theme, layout: { ...s.theme.layout, [key]: value } }, isDirty: true })),
    setDarkMode: (mode) => set((s) => ({ theme: { ...s.theme, darkMode: mode }, isDirty: true })),
    setCustomCss: (css) => set((s) => ({ theme: { ...s.theme, customCss: css }, isDirty: true })),
    applyPreset: (colors) => set((s) => ({ theme: { ...s.theme, colors: { ...colors } }, isDirty: true }))
}));
```

## Padrão para criar um novo store

```typescript
// src/infra/stores/auth-store.ts

import { create } from 'zustand';

interface User {
    id: string;
    name: string;
    email: string;
    roles: string[];
}

interface AuthStore {
    user: User | null;
    isAuthenticated: boolean;
    setUser: (user: User) => void;
    clearUser: () => void;
}

export const useAuthStore = create<AuthStore>((set) => ({
    user: null,
    isAuthenticated: false,

    setUser: (user) => set({ user, isAuthenticated: true }),
    clearUser: () => set({ user: null, isAuthenticated: false })
}));
```

## Como usar um store em um componente

```typescript
// src/presentation/layouts/app-shell/app-shell.tsx

import { useThemeStore } from '@/infra/stores/theme-store.js';

export function AppShell() {
    // Selecione apenas o que o componente usa — assinar o store inteiro (`useThemeStore()`)
    // re-renderiza o AppShell a cada `set()`, mesmo em mudanças irrelevantes.
    const darkMode = useThemeStore((s) => s.theme.darkMode);
    const setDarkMode = useThemeStore((s) => s.setDarkMode);
    const isDark = darkMode === 'dark';

    return (
        <IconButton onClick={() => setDarkMode(isDark ? 'light' : 'dark')}>
            {isDark ? <LightModeIcon /> : <DarkModeIcon />}
        </IconButton>
    );
}
```

## Convenções

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo | `kebab-case` + `-store.ts` | `theme-store.ts`, `auth-store.ts` |
| Interface do store | `PascalCase` + `Store` | `ThemeStore`, `AuthStore` |
| Hook exportado | `use` + `PascalCase` + `Store` | `useThemeStore`, `useAuthStore` |
| Chave no localStorage | `kebab-case` string | `'app-theme'`, `'user-prefs'` |

## Anti-padrões

❌ **Colocar estado de feature em store:**
```typescript
// Proibido — lista de pedidos é estado de feature, não global
export const useSalesOrdersStore = create(() => ({
    orders: [],
    loadOrders: async () => { ... }
}));
// ← isso vai no hook useSalesOrders, não num store global
```

❌ **Lógica de negócio dentro do store:**
```typescript
updateColor: (key, value) => {
    if (!isValidHex(value)) {
        return; // ← validação de negócio não pertence ao store
    }
    set((s) => ({ ... }));
}
```

❌ **Store importando de `application/` ou `presentation/`:**
```typescript
import { LoadThemeUseCase } from '@/application/use-cases/load-theme-use-case.js'; // ← proibido
```

## Regras de ouro

1. Stores gerenciam estado **global e transversal** — não estado de feature ou de página.
2. A interface do store deve ser declarada **explicitamente** antes do `create()` — nunca inferida.
3. Mutações sempre usam `set()` com imutabilidade — nunca mutar o estado diretamente.
4. Para persistência no `localStorage`, sempre usar uma `STORAGE_KEY` como constante de módulo (`const STORAGE_KEY`, fora do `create()`).
5. Componentes consomem o store via **selector** (`useThemeStore((s) => s.x)`) — nunca desestruturando o store inteiro, que re-renderiza a cada mudança.
6. Stores podem importar de `domain/` (tipos) e de `infra/theme/` — nunca de `application/`, `presentation/` ou `main/`.
