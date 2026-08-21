# Domain layer

A domain layer é o **núcleo da aplicação React**: define os contratos (interfaces) e os modelos ricos que descrevem o negócio sem depender de nenhum framework, biblioteca de UI, HTTP ou camada superior. É a única camada que **não importa de ninguém** — só ela é importada por todas as outras.

Aqui vivem: `interface XxxRepository`, `interface XxxUseCase`, `class XxxModel`, a hierarquia de erros (`AbstractError` + subclasses HTTP) e os protocolos técnicos que o domínio precisa (`interface HttpClient`). Toda implementação concreta vive em `infra/` ou `application/`; toda DI acontece em `main/factories/`.

> **Regra de isolamento (a mais importante da arquitetura):** a domain layer **não importa nada** de `application/`, `infra/`, `presentation/`, `main/`, `shared/` nem de qualquer framework (`react`, `@mui/material`, `zustand`, `i18next`, `react-router-dom`). Dentro de `domain/`, módulos podem se referenciar livremente. O único pacote externo aceito é `@sweet-monads/either` para o tipo `Either<L, R>`.

## Estrutura canônica

A estrutura é **fixa e obrigatória** — o MCP `scaffold-react-project` cria todas as pastas ao inicializar o projeto. Inclusões fora desta lista (`validators/`, `constants/`, `types/`, `services/`, `utils/`) **são proibidas**.

```
src/domain/
├── errors/
│   ├── abstract.ts          → AbstractError (base abstrata)
│   ├── bad-request-error.ts       → BadRequestError (400)
│   ├── unauthorized-error.ts      → UnauthorizedError (401)
│   ├── forbidden-error.ts         → ForbiddenError (403)
│   ├── not-found-error.ts         → NotFoundError (404)
│   ├── conflict-error.ts          → ConflictError (409)
│   └── server-error.ts            → ServerError (500)
├── models/
│   └── <entidade>.ts              → XxxProps + class XxxModel + resposta da API
├── protocols/
│   └── http-client.ts             → interface HttpClient + namespace
├── repositories/
│   └── <entidade>-repository.ts   → interface XxxRepository + namespace
└── use-cases/
    └── <feature>/
        └── <kebab-name>.ts        → interface XxxUseCase + namespace
```

## Responsabilidades

| Subpasta | Responsabilidade |
|---|---|
| `errors/` | `AbstractError` + 6 subclasses HTTP obrigatórias |
| `models/` | Entidades de negócio com factory methods, getters e mapeamento de resposta API |
| `protocols/` | Interfaces técnicas que o domínio precisa (ex.: `HttpClient`) — só contratos, sem implementação |
| `repositories/` | `interface XxxRepository` + namespace com tipos de retorno |
| `use-cases/` | `interface XxxUseCase` + namespace com `Result` usando `Either<AbstractError, T>` |

## Regras de ouro

1. **Nenhuma implementação no domain.** Repositórios e protocolos existem aqui apenas como `interface`. A única exceção autorizada são os **models**, que são classes ricas com lógica de negócio pura (mapeamento, formatação) — sem I/O.
2. **`Either<AbstractError, T>` é o tipo canônico de `Result` em use cases.** Importado direto de `@sweet-monads/either`. Quem produz `left()`/`right()` é a application layer; o domain só declara o tipo.
3. **6 erros HTTP obrigatórios**: `BadRequestError` (400), `UnauthorizedError` (401), `ForbiddenError` (403), `NotFoundError` (404), `ConflictError` (409), `ServerError` (500). Gerados pelo MCP e não devem ser removidos.
4. **Models são ricos.** Cada `XxxModel` concentra mapeamento de `Response → Model` via factory estático. Models anêmicos (type alias sem classe) são **proibidos**.
5. **`with(props)` é a porta única de construção do model** — recebe props já tipadas e é a única factory que chama o constructor privado. Construções a partir de outra origem usam o prefixo `withFrom*`: `withFromResponse(item)` para uma resposta da API e `withFromResponseList(list)` para arrays. Factories `from()`/`fromList()` são **proibidas** (divergiam do CAP).
6. **Use case = `interface XxxUseCase { execute(params?): Promise<XxxUseCase.Result> }` + `namespace XxxUseCase { Result }`.**
7. **Repository = `interface XxxRepository`** com tipos auxiliares no `namespace XxxRepository { FindAll, FindById, ... }`.
8. **Sem barrel exports** dentro do domain — cada arquivo é importado diretamente pelo caminho.
9. **Sem framework.** O domain **não importa** `react`, `@mui/material`, `zustand`, `react-router-dom` nem qualquer SDK externo.

## Pastas proibidas

| Pasta | Motivo | Substituto canônico |
|---|---|---|
| `validators/` | Validação fica encapsulada nos **models** | Método no `XxxModel` |
| `constants/` | Constantes de negócio ficam como enum ou `as const` dentro do model | Dentro de `models/<entidade>.ts` |
| `types/` | Tipos auxiliares vivem nos namespaces dos próprios contratos | `namespace XxxRepository { ... }` |
| `services/` | Não existe service no domain React — lógica vai para `application/use-cases/` | `application/use-cases/<feature>/` |
| `utils/` | Utilitários vão para `shared/utils/` | `src/shared/utils/` |

## Naming

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo | `kebab-case.ts` | `sales-order-header.ts`, `customer-repository.ts` |
| Classe de model | `PascalCase` + `Model` | `CustomerModel`, `SalesOrderHeaderModel` |
| Tipo de props | `PascalCase` + `Props` | `CustomerProps`, `SalesOrderHeaderProps` |
| Tipo de resposta API | `PascalCase` + `Response` | `CustomerResponse`, `SalesOrderHeaderResponse` |
| Interface repository | `PascalCase` + `Repository` | `CustomerRepository`, `SalesOrderRepository` |
| Interface use case | `PascalCase` descritivo | `LoadCustomers`, `CloneSalesOrder` |
| Interface protocol | `PascalCase` descritivo | `HttpClient` |
| Namespace colateral | Mesmo nome da interface | `namespace CustomerRepository { FindAll }` |

**Sufixos proibidos no domain:** `Impl`, `Fake`, `Mock`, `Dto`, `Entity`, `UseCase`. No domain, use cases são interfaces sem sufixo — o sufixo `UseCase` é do CAP backend.

## Quem importa o quê

| Camada | Pode importar de `domain/`? |
|---|---|
| `domain/` (própria) | ✅ Entre subpastas livremente |
| `application/` | ✅ Implementações consomem contratos do domain |
| `infra/` | ✅ Implementações consomem contratos do domain |
| `presentation/` | ✅ Controllers e hooks usam tipos de retorno |
| `main/` | ✅ Factories instanciam `XxxImpl implements XxxInterface` |
| `shared/` | ❌ Shared não importa domain — é transversal neutro |

| O que o `domain/` importa | Permitido? |
|---|---|
| `application/`, `infra/`, `presentation/`, `main/`, `shared/` | ❌ Nunca |
| `react`, `@mui/material`, `zustand`, `react-router-dom` | ❌ Nunca |
| `@sweet-monads/either` | ✅ Único pacote externo aceito |

## Documentos desta seção

- [models.md](./models.md) — `XxxProps`, `class XxxModel`, `from()`, `fromList()`, mapeamento API → Model
- [errors.md](./errors.md) — `AbstractError` + 6 subclasses HTTP + uso com `Either`
- [protocols.md](./protocols.md) — `interface HttpClient` + padrão de namespace para tipos
- [repositories.md](./repositories.md) — `interface XxxRepository` + namespace com tipos de retorno
- [use-cases.md](./use-cases.md) — `interface XxxUseCase` + namespace + `Either<AbstractError, T>`
