# Shared layer — Utils

Funções utilitárias puras de uso geral — formatação de moeda, data, string e outras transformações sem side effects. São as únicas funções do projeto que não pertencem a nenhuma camada específica.

## Estrutura

```
src/shared/utils/
├── format-currency.ts    → formatCurrency()
├── format-date.ts        → formatDate()
└── format-status-label.ts → formatStatusLabel()
```

## Definição de "função pura"

Uma função pura:
- Dado o mesmo input, **sempre** retorna o mesmo output
- **Não modifica** variáveis externas
- **Não acessa** rede, storage, contexto React ou estado global
- **Não lança** exceções para inputs válidos

Se a função precisa de acesso a store, context ou I/O, ela não é um utilitário — vai para um hook ou para a infra.

## Exemplos gerados pela referência do projeto

### `format-currency.ts`

```typescript
// src/shared/utils/format-currency.ts

// Construir um Intl.NumberFormat é ordens de magnitude mais caro que chamar .format().
// Em células de tabela (centenas de chamadas por render) isso pesa — então cacheamos por locale+currency.
const formatters = new Map<string, Intl.NumberFormat>();

export function formatCurrency(
    value: number,
    locale: string = 'pt-BR',
    currency: string = 'BRL'
): string {
    const key = `${locale}:${currency}`;
    let formatter = formatters.get(key);
    if (!formatter) {
        formatter = new Intl.NumberFormat(locale, { style: 'currency', currency });
        formatters.set(key, formatter);
    }
    return formatter.format(value);
}

// Uso:
// formatCurrency(1500)           → "R$ 1.500,00"
// formatCurrency(1500, 'en-US', 'USD') → "$1,500.00"
```

### `format-date.ts`

```typescript
// src/shared/utils/format-date.ts

import dayjs from 'dayjs';

export function formatDate(value: string | Date, format: string = 'DD/MM/YYYY'): string {
    return dayjs(value).format(format);
}

// Uso:
// formatDate('2024-01-15')              → "15/01/2024"
// formatDate('2024-01-15', 'MM/YYYY')  → "01/2024"
```

### `format-status-label.ts`

```typescript
// src/shared/utils/format-status-label.ts

type Status = 'COMPLETED' | 'PENDING' | 'REJECTED';

const STATUS_LABELS: Record<Status, string> = {
    COMPLETED: 'Concluído',
    PENDING: 'Pendente',
    REJECTED: 'Rejeitado'
};

export function formatStatusLabel(status: Status): string {
    return STATUS_LABELS[status] ?? status;
}
```

> **Nota:** labels de status que precisam de internacionalização devem usar `translate()` via `infra/i18n` — nesse caso, a função não pertence ao `shared/utils/` (pois importaria de `infra/`), mas sim ser definida diretamente no componente ou no hook que a usa.

## Padrão para novas utils

```typescript
// src/shared/utils/format-percentage.ts

export function formatPercentage(value: number, decimals: number = 1): string {
    return `${value.toFixed(decimals)}%`;
}

// Uso: formatPercentage(0.153) → "0.2%"
// Uso: formatPercentage(15.3, 2) → "15.30%"
```

## Convenções

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo | `format-kebab-case.ts` ou `<verbo>-kebab-case.ts` | `format-currency.ts`, `parse-date.ts` |
| Função | `camelCase` com prefixo do verbo | `formatCurrency`, `parseDate`, `truncateText` |
| Prefixos comuns | `format*`, `parse*`, `truncate*`, `to*`, `is*` | `formatDate`, `isValidEmail` |

## O que vai em utils vs. onde não vai

| Caso | Vai em `shared/utils/`? | Alternativa |
|---|---|---|
| `formatCurrency(value)` | ✅ Sim — pura, sem dependência de camada | — |
| `formatDate(value)` | ✅ Sim — pura, usa `dayjs` (pacote neutro) | — |
| `translateStatus(status)` com `translate()` | ❌ Não — importa de `infra/i18n` | Inline no componente |
| `getUserName()` com `useAuthStore()` | ❌ Não — acessa estado global | Hook |
| `validateCPF(cpf)` — regra de negócio | ❌ Não — pertence ao model | Método no model |
| `buildQueryString(params)` — helper de repositório | ❌ Não — pertence à infra | `private readonly` no repositório |

## Anti-padrões

❌ **Função com side effect:**
```typescript
export function formatAndLog(value: number): string {
    console.log(`Formatting: ${value}`); // ← side effect
    return formatCurrency(value);
}
```

❌ **Utilitário com acesso a store:**
```typescript
export function getFormattedUserName(): string {
    const { user } = useAuthStore(); // ← hooks só em componentes/hooks React
    return user?.name ?? '';
}
```

❌ **Lógica de negócio disfarçada de utilitário:**
```typescript
export function calculateOrderTotal(items: OrderItem[]): number {
    // ← regra de negócio — pertence ao model ou ao use case
    return items.reduce((sum, i) => sum + i.price * i.quantity, 0);
}
```

## Regras de ouro

1. Toda função em `utils/` é **pura** — sem side effects, sem acesso a contexto ou store.
2. Se a função precisar de `translate()`, ela **não** vai em `shared/utils/` — use inline no componente ou crie um hook.
3. Uma função por arquivo — nome do arquivo reflete o nome da função exportada.
4. Funções puras são trivialmente testáveis — não precisam de mocks.
