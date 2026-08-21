# Presentation layer — Components

Componentes reutilizáveis são **puramente visuais**: recebem dados via props, renderizam UI e disparam callbacks — sem estado próprio de loading/erro e sem chamadas a use cases. São os building blocks compartilhados entre múltiplas páginas.

## Estrutura

```
src/presentation/components/
├── <categoria>/
│   └── <nome>.tsx
└── index.ts    → barrel export de todos os componentes
```

Categorias comuns baseadas no projeto de referência:

```
components/
├── error-message/
│   └── error-message.tsx
├── loader/
│   └── loader.tsx
├── status-badge/
│   └── status-badge.tsx
├── tables/
│   └── base-table.tsx
└── index.ts
```

## Barrel export — `index.ts`

Centraliza os imports de todos os componentes reutilizáveis:

```typescript
// src/presentation/components/index.ts

export { BaseTable } from './tables/base-table';
export type { BaseTableColumn } from './tables/base-table';
export { ErrorMessage } from './error-message/error-message';
export { Loader } from './loader/loader';
export { StatusBadge } from './status-badge/status-badge';
```

Com o barrel, as páginas importam de forma limpa:

```typescript
import { Loader, ErrorMessage, BaseTable } from '@/presentation/components/index.js';
```

## Exemplos de componentes

### Exemplo 1 — Componente de loading

```typescript
// src/presentation/components/loader/loader.tsx

import Box from '@mui/material/Box';
import CircularProgress from '@mui/material/CircularProgress';

export function Loader() {
    return (
        <Box sx={{ display: 'flex', justifyContent: 'center', alignItems: 'center', p: 4 }}>
            <CircularProgress size={32} />
        </Box>
    );
}
```

### Exemplo 2 — Componente de erro com ação de retry

```typescript
// src/presentation/components/error-message/error-message.tsx

import Box from '@mui/material/Box';
import Button from '@mui/material/Button';
import Typography from '@mui/material/Typography';
import { useTranslate } from '@/presentation/contexts/translate-context.js';

interface Props {
    message?: string;
    onRetry?: () => void;
}

export function ErrorMessage({ message, onRetry }: Props) {
    const translate = useTranslate();
    return (
        <Box sx={{ display: 'flex', flexDirection: 'column', alignItems: 'center', gap: 2, p: 4 }}>
            <Typography color="error" variant="body1">
                {message ?? translate('errors.generic')}
            </Typography>
            {onRetry && (
                <Button variant="outlined" size="small" onClick={onRetry}>
                    {translate('common.retry')}
                </Button>
            )}
        </Box>
    );
}
```

### Exemplo 3 — Badge de status

```typescript
// src/presentation/components/status-badge/status-badge.tsx

import Chip from '@mui/material/Chip';

type Status = 'COMPLETED' | 'PENDING' | 'REJECTED';

interface Props {
    status: Status;
    label: string;
}

const colorMap: Record<Status, 'success' | 'warning' | 'error'> = {
    COMPLETED: 'success',
    PENDING: 'warning',
    REJECTED: 'error'
};

export function StatusBadge({ status, label }: Props) {
    return <Chip label={label} color={colorMap[status]} size="small" />;
}
```

### Exemplo 4 — Tabela genérica com tipagem

```typescript
// src/presentation/components/tables/base-table.tsx

import Table from '@mui/material/Table';
import TableBody from '@mui/material/TableBody';
import TableCell from '@mui/material/TableCell';
import TableHead from '@mui/material/TableHead';
import TableRow from '@mui/material/TableRow';

export interface BaseTableColumn<T> {
    key: keyof T | string;
    header: string;
    render?: (row: T) => React.ReactNode;
}

interface Props<T> {
    columns: BaseTableColumn<T>[];
    rows: T[];
    getRowKey: (row: T) => string;
}

export function BaseTable<T>({ columns, rows, getRowKey }: Props<T>) {
    return (
        <Table size="small">
            <TableHead>
                <TableRow>
                    {columns.map((col) => (
                        <TableCell key={String(col.key)}>{col.header}</TableCell>
                    ))}
                </TableRow>
            </TableHead>
            <TableBody>
                {rows.map((row) => (
                    <TableRow key={getRowKey(row)}>
                        {columns.map((col) => (
                            <TableCell key={String(col.key)}>
                                {col.render ? col.render(row) : String((row as Record<string, unknown>)[String(col.key)] ?? '')}
                            </TableCell>
                        ))}
                    </TableRow>
                ))}
            </TableBody>
        </Table>
    );
}
```

## Diferença: componente global vs. componente local de página

| Tipo | Onde fica | Quando usar |
|---|---|---|
| Global (`components/`) | `presentation/components/` | Usado em **2+ páginas diferentes** |
| Local (`pages/<feature>/components/`) | Dentro da pasta da página | Usado apenas **naquela página** |

```
presentation/pages/sales-orders/
├── components/                     ← componentes locais da página
│   ├── dialogs/
│   │   └── sales-order-confirmation-dialog.tsx
│   └── tables/
│       ├── sales-order-detail-table.tsx
│       └── sales-order-table.tsx
└── sales-orders-page.tsx
```

## Anti-padrões

❌ **Estado de loading/erro dentro do componente reutilizável:**
```typescript
export function SalesOrderTable() {
    const [orders, setOrders] = useState([]);
    useEffect(() => { /* fetch aqui */ }, []); // ← estado de feature vai no hook
}
```

❌ **Chamada a use case ou controller dentro do componente:**
```typescript
export function CustomerSelect({ onSelect }: Props) {
    const controller = new CustomerController(...); // ← não instanciar aqui
}
```

❌ **Componente reutilizável importando diretamente de `application/`:**
```typescript
import { LoadCustomersUseCase } from '@/application/use-cases/customers/load-customers-use-case.js'; // ← proibido
```

## Regras de ouro

1. Componentes em `components/` são **puramente visuais** — sem lógica de negócio, sem chamadas a use cases.
2. **Barrel export** em `index.ts` é obrigatório — centraliza e simplifica os imports.
3. Um componente vai para `components/` quando é **usado em 2 ou mais páginas** — caso contrário, fica em `pages/<feature>/components/`.
4. Props devem ser **tipadas explicitamente** com `interface Props { ... }`.
5. Componentes podem importar de `domain/` (tipos) e de `presentation/contexts` (hooks como `useTranslate`, `useThemeContext`) — nunca de `infra/`, `application/` ou `main/`.
