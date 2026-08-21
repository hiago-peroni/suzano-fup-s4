# Infra layer

A infra layer **executa o trabalho técnico**: faz requisições HTTP, persiste dados, gerencia estado global, carrega configurações de tema e inicializa internacionalização. Todas as implementações satisfazem interfaces de domínio. Nenhuma regra de negócio vive aqui.

> **Regra de isolamento:** a infra layer importa de `domain/` (para implementar interfaces) e de bibliotecas externas (`react`, `@mui/material`, `zustand`, `i18next`). **Não importa** de `application/`, `presentation/` ou `main/`.

## Estrutura canônica

```
src/infra/
├── http/
│   └── fetch-http-client.ts         → FetchHttpClient implements HttpClient
├── repositories/
│   └── <entidade>-repository-impl.ts → XxxRepositoryImpl implements XxxRepository
├── stores/
│   └── theme-store.ts               → useThemeStore (Zustand)
├── theme/
│   ├── theme-config.ts              → ThemeConfig, ThemeColors (interfaces)
│   ├── theme-defaults.ts            → DEFAULT_THEME (valores padrão)
│   ├── theme-provider.tsx           → ThemeAppProvider (React context + MUI)
│   └── theme-utils.ts               → buildMuiTheme(), themeToCustomProperties()
└── i18n/
    ├── index.ts                     → inicialização do i18next + export translate()
    ├── pt-br.json                   → traduções em português
    └── en.json                      → traduções em inglês
```

## Responsabilidades

| Subpasta | Responsabilidade |
|---|---|
| `http/` | Implementação do `HttpClient` via Fetch API, mapeamento de status → erros de domínio |
| `repositories/` | Implementações concretas de `XxxRepository`, transformação de resposta API → model |
| `stores/` | Estado global com Zustand (tema, preferências de usuário, dados de sessão) |
| `theme/` | Configuração, defaults, builder de tema MUI e provider React |
| `i18n/` | Inicialização do i18next, detecção de idioma, arquivos JSON de tradução |

## Regras de ouro

1. **Nenhuma regra de negócio.** A infra executa I/O e transforma dados — não decide o que fazer com eles.
2. **Errors são lançados (`throw`), não retornados como `Either`.** A application layer captura e converte para `left()`.
3. **Repositórios recebem `HttpClient` via constructor** — nunca instanciam `FetchHttpClient` diretamente.
4. **Stores Zustand gerenciam estado transversal** (tema, autenticação) — não estado de feature (que fica nos hooks).
5. **Sem imports de `application/`, `presentation/` ou `main/`** dentro da infra.

## Tipagem e constantes

Estas regras valem para **repositórios e HTTP client** — os pontos onde a infra fala com a API. Stores, providers e arquivos de configuração de tema têm exceções explícitas, detalhadas abaixo.

### Regra 1 — Sem DTOs de API na infra

Nenhuma declaração de `type` ou `interface` local para **shapes de request/response de API** dentro de repositórios ou do HTTP client. Esses tipos vêm de `@/domain/` — via namespace do repositório ou use case.

**Exceção:** stores, providers e `theme-config.ts` declaram interfaces locais para descrever a própria forma (`interface ThemeStore`, `interface ThemeAppProviderProps`, `interface ThemeColors`). Não são DTOs de API — são o contrato interno daquele módulo de infra e ficam no próprio arquivo.

| Conceito | Onde fica |
|---|---|
| Shape de resposta de API | Namespace da interface do repositório em `domain/repositories/...` |
| `Params`/`Result` de método público | Namespace do contrato (`XxxRepository.Filters`, `XxxUseCase.Params`) |
| Model de entidade | `domain/models/<entidade>.ts` |

### Regra 2 — Tipos do domain sempre

Métodos de repositório recebem e retornam tipos do domain layer. Nunca criar um tipo intermediário na infra.

❌ **Errado — tipo declarado dentro da infra:**

```typescript
// src/infra/repositories/customer-repository-impl.ts
type CustomerApiResponse = {
    id: string;
    name: string;
};

export class CustomerRepositoryImpl implements CustomerRepository {
    // ...
}
```

✅ **Certo — tipo no domain, importado pela infra:**

```typescript
// src/domain/repositories/customer-repository.ts
export namespace CustomerRepository {
    export type ApiResponse = {
        id: string;
        name: string;
    };
}

// src/infra/repositories/customer-repository-impl.ts
import type { CustomerRepository } from '@/domain/repositories/customer-repository.js';

export class CustomerRepositoryImpl implements CustomerRepository {
    // ...
}
```

### Regra 3 — Constantes de repositório como `private readonly`

Constantes acopladas a uma classe — como o `BASE_URL` de um repositório — devem ser declaradas como `private readonly` dentro da classe, não como `const` de módulo.

**Exceção:** chaves de `localStorage` de um store (`STORAGE_KEY`) são `const` de módulo, porque o store é uma função (`create()`), não uma classe — não existe `private const` em TypeScript.

❌ **Errado — `const` em module-level:**

```typescript
const BASE_URL = 'https://api.example.com';

export class CustomerRepositoryImpl implements CustomerRepository {
    // ...
}
```

✅ **Certo — propriedade `private readonly` da classe:**

```typescript
export class CustomerRepositoryImpl implements CustomerRepository {
    private readonly BASE_URL = 'https://api.example.com';
    // ...
}
```

## Naming

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo | `kebab-case.ts/.tsx` | `fetch-http-client.ts`, `theme-store.ts` |
| Classe de repositório | `PascalCase` + `RepositoryImpl` | `CustomerRepositoryImpl` |
| Classe HTTP client | `PascalCase` + `HttpClient` | `FetchHttpClient` |
| Store Zustand | `use` + `PascalCase` + `Store` | `useThemeStore` |
| Provider React | `PascalCase` + `Provider` | `ThemeAppProvider` |

## Documentos desta seção

- [http.md](./http.md) — `FetchHttpClient`, mapeamento HTTP → erros de domínio
- [repositories.md](./repositories.md) — `XxxRepositoryImpl`, transformação API → model
- [stores.md](./stores.md) — Zustand, padrão de store, quando usar store vs. estado local
- [theme.md](./theme.md) — `ThemeConfig`, `ThemeAppProvider`, `buildMuiTheme`
- [i18n.md](./i18n.md) — inicialização i18next, `translate()`, estrutura dos JSONs
