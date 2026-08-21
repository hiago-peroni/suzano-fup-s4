# Shared layer

A shared layer contém **utilitários transversais** usados por múltiplas camadas da aplicação. É a única camada que pode ser importada por qualquer outra — mas ela mesma não importa de nenhuma camada da arquitetura.

> **Regra de isolamento:** `shared/` **não importa** de `domain/`, `application/`, `infra/`, `presentation/` ou `main/`. Importa apenas de pacotes externos genéricos (`dayjs`, `zod`, etc.). Tudo que precisa de uma camada específica não pertence ao `shared/`.

## Estrutura

```
src/shared/
├── constants/
│   └── routes.ts       → ROUTES + funções de navegação
├── types/              → tipos TypeScript transversais (sem lógica)
└── utils/
    └── <nome>.ts       → funções puras de formatação/transformação
```

## Responsabilidades

| Subpasta | Responsabilidade |
|---|---|
| `constants/` | Constantes globais: rotas, códigos de status, enum de configuração |
| `types/` | Tipos TypeScript auxiliares usados em múltiplas camadas |
| `utils/` | Funções puras sem side effects: formatação de moeda, data, string |

## O que NÃO pertence ao shared

| Tipo | Onde pertence |
|---|---|
| Tipos de entidades de negócio | `domain/models/` |
| Constantes de uma feature específica | `private readonly` na classe, ou no model |
| Funções com acesso a estado (store, context) | Hook ou componente |
| Funções que fazem I/O (fetch, localStorage) | `infra/` |
| Helpers de um use case específico | `application/use-cases/<feature>/` |

## Naming

| Artefato | Padrão | Exemplo |
|---|---|---|
| Arquivo de util | `kebab-case` | `format-currency.ts` |
| Função utilitária | `camelCase` | `formatCurrency()` |
| Arquivo de constantes | `kebab-case` | `routes.ts` |
| Constante de rota | `SCREAMING_SNAKE_CASE` ou objeto `camelCase` | `ROUTES.home` ou `routes.home` |
| Pasta | `kebab-case` | `shared/utils/`, `shared/constants/` |

## Regras de ouro

1. **Shared não tem dependências de camada** — se precisar importar de `domain/` ou `infra/`, o módulo não pertence ao shared.
2. **Funções em `utils/` são puras** — dado o mesmo input, sempre o mesmo output. Sem side effects.
3. **`constants/routes.ts` é a única fonte de verdade** para caminhos de rota — nunca strings literais nas rotas.
4. A shared é **pequena e focada** — não é um "misc" onde tudo que não tem lugar vai parar.

## Documentos desta seção

- [constants.md](./constants.md) — `ROUTES`, funções de navegação (`to*`), outras constantes globais
- [utils.md](./utils.md) — funções puras de formatação, convenções e testabilidade
