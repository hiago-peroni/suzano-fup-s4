# React Clean Architecture — Padrões do Projeto

Documentação dos padrões de Clean Architecture aplicados em projetos React gerados pelo MCP `@numends/mcp-react-clean-arch`. Cada sub-doc (`<layer>/README.md`) cobre as convenções específicas da camada; este README cobre a regra de dependência global, o glossário e o mapeamento com o CAP.

---

## Regra de Dependência — 6 Camadas

As dependências só podem apontar **para dentro** (em direção ao domínio). Nenhuma camada pode importar de uma camada mais externa.

```
shared  ←  domain  ←  application
                  ↓
               infra
                  ↓
           presentation
                  ↓
                main
                  ↑
         (imports de todas)
```

| Camada | Pode importar de |
|---|---|
| `shared` | — (sem dependências) |
| `domain` | `shared` |
| `application` | `domain`, `shared` |
| `infra` | `domain`, `shared` |
| `presentation` | `domain`, `shared` |
| `main` | todas as camadas |

> **Violação típica**: `presentation` importando de `infra` ou `application` diretamente. O acesso a serviços de infra (i18n, stores, tema) em `presentation` deve passar por abstração de `domain` ou por contextos React providos pelo `main`.

---

## Glossário

| Termo | Definição |
|---|---|
| **Domain layer** | Modelos de negócio, erros tipados, interfaces de repositório e protocolos — sem dependências de framework |
| **Application layer** | Implementações de use cases; orquestra repositórios via interfaces de domínio |
| **Infra layer** | Implementações concretas de I/O: HTTP client, stores Zustand, tema MUI, i18n |
| **Presentation layer** | Componentes React, controllers, hooks de estado, páginas e layouts |
| **Main layer** | Entry point, factories de composição de raiz, rotas e wiring de DI |
| **Shared layer** | Utilitários e constantes sem dependência de camada — usados por qualquer camada |
| **AbstractError** | Classe base de todos os erros do domínio; possui `code: number` obrigatório |
| **Either** | Tipo monádico `Left<E> | Right<T>` — `Left` = erro de domínio, `Right` = dado válido |
| **Controller** | Adapter entre o use case e o estado React; recebe o resultado do use case e retorna `BaseControllerState<T>` |
| **Factory** | Função em `main/factories/` que compõe e injeta dependências para um use case, controller ou página |
| **Hook** | Custom hook em `presentation/hooks/` que conecta o controller ao estado React (`useState`, `useCallback`, `useEffect`) |

---

## Mapa de Equivalência com o CAP

O CAP (`cap-clean-arch`) usa nomenclatura de camadas diferente. A tabela abaixo mostra as equivalências:

| React Clean Arch | CAP (cap-clean-arch) | Descrição |
|---|---|---|
| `domain` | `domain` | Modelos, erros, interfaces — idêntico |
| `application` | `application` | Use cases / application services |
| `infra` | `infrastructure` | HTTP, banco, stores, i18n, tema |
| `presentation` | `adapters` | Controllers, presenters, views |
| `main` | `main` | Wiring, entry point, factories |
| `shared` | *(sem equivalente)* | Utilitários transversais — não existe no CAP canônico |

> `shared` não possui equivalente no CAP porque o CAP agrupa utilitários no próprio `domain`. No React Clean Arch, `shared` foi separado para acomodar constantes e utils React-específicos sem poluir o domínio.

---

## Estrutura de diretórios gerada

```
src/
├── domain/
│   ├── errors/         → AbstractError + 6 subclasses HTTP
│   ├── models/         → entidades do domínio
│   ├── protocols/      → interfaces de I/O (HttpClient)
│   ├── repositories/   → interfaces de repositório
│   └── use-cases/      → interfaces de use case
├── application/
│   └── use-cases/      → implementações de use case
├── infra/
│   ├── http/           → FetchHttpClient
│   ├── i18n/           → i18next + translate()
│   ├── repositories/   → implementações de repositório
│   ├── stores/         → Zustand stores
│   └── theme/          → configuração e provider de tema MUI
├── presentation/
│   ├── components/     → componentes reutilizáveis
│   ├── controllers/    → base + controllers de feature
│   ├── hooks/          → custom hooks de feature
│   ├── layouts/        → AppShell e outros layouts
│   └── pages/          → páginas de feature
├── main/
│   ├── app.tsx         → raiz da aplicação React
│   ├── index.tsx       → entry point DOM
│   ├── routes.tsx      → definição de rotas
│   └── factories/      → use-cases/, controllers/, pages/
└── shared/
    ├── constants/      → ROUTES e outras constantes globais
    ├── types/          → tipos transversais
    └── utils/          → utilitários puros
```

Cada camada tem seu próprio `README.md` com Anti-padrões, Regras de ouro e Tipagem e constantes.
