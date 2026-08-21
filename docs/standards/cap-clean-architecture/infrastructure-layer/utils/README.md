# Utils

`infrastructure/utils/` agrupa utilitários transversais com I/O leve ou leitura de contexto runtime. É diferente das outras subpastas da infrastructure:

| Subpasta | O que faz |
|---|---|
| `adapters/` | Wrappers de I/O remoto (APIs externas, parsers, UoW) |
| `repositories/` | Acesso ao CDS/HANA via `cds.ql.*` |
| `hydrators/` | Mutadores de `request.query.SELECT.where` ou `request.body` |
| `utils/` | Leitura de contexto runtime e tradução de mensagens — I/O leve, sem orquestração |

## Estrutura

```
infrastructure/utils/
├── translator/
│   ├── translator.ts                  → TranslatorImpl (classe) + DI de ResourceManager + ALS
│   └── i18n/
│       ├── i18n.properties            → bundle base (chaves + texto EN, fallback)
│       ├── i18n_pt.properties         → tradução PT-BR
│       └── i18n_es.properties         → tradução ES
├── get-user.ts                        → function getUser(request) — extrai usuário do request CAP
└── get-environment.ts                 → function getEnvironment() — lê credentials BTP via @sap/xsenv
```

## Naming

| Elemento | Convenção | Exemplo |
|---|---|---|
| Translator (classe) | `TranslatorImpl` | `TranslatorImpl` |
| Função utilitária | `kebab-case.ts` + `export function camelCase` | `get-user.ts` → `export function getUser(request)` |
| Pasta `i18n` | `i18n/` plano dentro de `translator/` | `utils/translator/i18n/` |
| Bundle base | `i18n.properties` | — |
| Bundle por locale | `i18n_<locale>.properties` | `i18n_pt.properties`, `i18n_es.properties` |

## Quando criar uma função em `utils/` vs uma classe

| Caso | Decisão |
|---|---|
| Leitura única de contexto runtime (env, request.user, headers) | `export function getXxx(...)` em arquivo próprio |
| Reuso entre múltiplas operações com I/O ou estado | classe `XxxImpl` com DI (translator é o único exemplo hoje) |
| Cache ou estado interno (ALS, ResourceManager) | classe (composta via factory) |
| Formatação determinística de primitivo (`Date`, `number`) | **NÃO** — vira método do domain model |

## Regras de ouro

1. **Funções utilitárias são puras ou quase-puras.** Aceitam input explícito, retornam valor tipado, sem side-effect global. Exceção: `getEnvironment` consulta `process.env` / `xsenv` — efeito de leitura de ambiente é aceitável.
2. **Sem `extractors/` nem `formatters/`.** Esses padrões foram rejeitados (anti-padrão Suzano). Use funções utilitárias em `utils/` para leitura de contexto; use métodos de domain model para formatação.
3. **Funções utilitárias propagam exceção.** Sem `Either<L,R>`.
4. **Interfaces e namespaces das funções vivem em `domain/utils/`** — exemplo: `domain/utils/translator.ts` define a interface `Translator` e os tipos `Translator.LanguageContext` / `Translator.TranslateParams`; `domain/utils/get-user.ts` define o namespace `GetUser` com `LoggedUser`, `RequestWithUser`, etc.

## Tipagem e constantes

Aplicar as 3 regras globais da [seção "Tipagem e constantes" do README da infrastructure-layer](../README.md#tipagem-e-constantes). Pontos específicos para utils:

| Caso | Aplicação |
|---|---|
| Função pura sem classe (`get-user.ts`, `get-environment.ts`) | Helpers internos como `const arrow = () => {...}` são **declarações de função**, não constantes de dado — não violam a regra de `private readonly`. |
| Cache module-level (`let cachedEnvironment: ... | null = null`) | Tolerado **exclusivamente** em funções utilitárias puras que não têm classe. Para qualquer código em classe (adapter, repository, hydrator), use propriedade `private`/`private readonly`. |
| Classe utilitária (`TranslatorImpl`) | Constantes auxiliares (locales, flags) como `private readonly` da classe — nunca `const` em module-level. |
| Tipos auxiliares | Sempre no namespace da interface em `domain/utils/<arquivo>.ts` (`Translator.LanguageContext`, `GetUser.LoggedUser`, `GetEnvironment.Environment`). |

## Anti-padrões

1. **Criar pasta `extractors/` ou `formatters/` na infrastructure.** Foram rejeitados como padrão canônico — a primeira virou funções em `utils/`; a segunda virou métodos de domain model.
2. **Nome `getLoggedUser` em vez de `getUser`.** O nome mais simples é o canônico (decisão do usuário no research).
3. **Translator com parâmetro `locale`** (padrão LE44) em vez de ALS (padrão MRO/RVE). O locale vem do `AsyncLocalStorage`, nunca de parâmetro de método.
4. **Bundle único `messages_pt.properties`** (anti-padrão Suzano) — sempre incluir bundle base `i18n.properties` + bundles por locale.
5. **`let cachedXxx` em module-level dentro de uma classe** — a exceção do cache module-level vale **apenas** para funções utilitárias puras (`getEnvironment`). Dentro de classe, sempre propriedade da instância.
6. **`type`/`interface` declarado em `infrastructure/utils/*.ts`** — todos os tipos vivem em `domain/utils/`.

## Documentos desta seção

- [translator/README.md](./translator/README.md) — `TranslatorImpl` + `AsyncLocalStorage` + factory
- [translator/i18n.md](./translator/i18n.md) — arquivos `.properties`, bundles, configuração `.cdsrc.json`
- [get-user.md](./get-user.md) — extração do usuário logado do request CAP
- [get-environment.md](./get-environment.md) — leitura de credentials BTP via `@sap/xsenv`
