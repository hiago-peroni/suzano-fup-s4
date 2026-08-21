# Domain layer

A domain layer é o **núcleo da aplicação**: define os contratos (interfaces) e os modelos ricos que descrevem o negócio sem depender de nenhum framework, banco, biblioteca de I/O ou camada superior. É a única camada que **não importa de ninguém** — só ela é importada por todas as outras.

Aqui vivem os artefatos puramente declarativos: `interface XxxRepository`, `interface XxxUseCase`, `class XxxModel`, `class AbstractError` + subclasses HTTP, contratos de adapters, hydrators e utilitários. Toda implementação concreta vive em `infrastructure/` ou `application/`; toda DI acontece em `main/factories/`.

> **Regra de isolamento (a mais importante do monorepo):** a domain layer **não importa nada** de `application/`, `infrastructure/`, `presentation/`, `main/` ou de framework (`@sap/cds`, `@models/*`, `@cds-models/*`, etc.). Dentro de `domain/`, módulos podem se referenciar livremente. O único pacote externo aceito é `@sweet-monads/either` para o tipo `Either<L, R>`.

## Estrutura canônica

A estrutura é **fixa e obrigatória** — o scaffold do MCP cria todas as pastas, mesmo iniciando vazias. Inclusões fora desta lista (`validators/`, `middlewares/`, `extractors/`, `formatters/`, `constants/`, `types/`, `external-apis/` raiz, `translation/`) **são proibidas** — ver seção "Pastas proibidas" abaixo.

```
src/domain/
├── adapters/
│   ├── external-api/
│   │   └── <sistema>/                      → contratos de APIs externas (XxxApi)
│   ├── parsers/                            → contratos de parsers (XxxParser)
│   └── unit-of-work.ts                     → contrato UnitOfWork (batch transacional)
├── errors/
│   ├── abstract.ts                         → AbstractError (base)
│   ├── bad-request.ts                      → BadRequestError (400)
│   ├── unauthorized.ts                     → UnauthorizedError (401)
│   ├── forbidden.ts                        → ForbiddenError (403)
│   ├── not-found.ts                        → NotFoundError (404)
│   ├── conflict.ts                         → ConflictError (409)
│   ├── server.ts                           → ServerError (500)
│   └── index.ts                            → barrel das classes de erro
├── hydrators/
│   └── <entidade-plural>/
│       ├── before-read.ts                  → BeforeReadXxxHydrator (interface)
│       └── before-create.ts                → BeforeCreateXxxHydrator (interface)
├── models/
│   ├── <contexto>/                         → models agrupados por contexto (db/, sap/, etc.)
│   │   └── <entidade>.ts                   → XxxProps + class XxxModel
│   └── <entidade>.ts                       → models de raiz (entidades transversais)
├── repositories/
│   └── <entidade-plural>.ts                → interface XxxRepository + namespace
├── services/
│   └── <kebab-name>.ts                     → interface XxxService + namespace (helper granular de use case — só contrato)
├── use-cases/
│   ├── actions/
│   │   └── <kebab-name>.ts                 → interface XxxUseCase + namespace
│   ├── entity-events/
│   │   └── <entidade-plural>/
│   │       ├── before-create.ts            → BeforeCreateXxxUseCase
│   │       ├── before-read.ts
│   │       ├── before-update.ts
│   │       ├── before-delete.ts
│   │       ├── after-create.ts
│   │       ├── after-read.ts
│   │       ├── after-update.ts
│   │       ├── after-delete.ts
│   │       └── index.ts                    → reexport dos use cases da entidade
│   └── functions/
│       └── <kebab-name>.ts                 → interface XxxUseCase + namespace
└── utils/
    ├── translator.ts                       → interface Translator + namespace
    ├── get-user.ts                         → namespace GetUser (LoggedUser, RequestWithUser, etc.)
    └── get-environment.ts                  → namespace GetEnvironment (Environment shape)
```

## Responsabilidades

| Subpasta | Responsabilidade |
|---|---|
| `adapters/external-api/<sistema>/` | Contratos de wrappers de APIs externas (S/4, REST, AWS) — `interface XxxApi` |
| `adapters/parsers/` | Contratos de parsers de arquivo (Excel, CSV, JSON) — `interface XxxParser` |
| `adapters/unit-of-work.ts` | Contrato `UnitOfWork` para batch transacional |
| `errors/` | `AbstractError` + 6 subclasses HTTP obrigatórias + barrel `index.ts` |
| `hydrators/<entidade>/` | Contratos de mutadores de `request` por evento de entidade |
| `models/` | Models ricos com `XxxProps`, factory `with(props)`, getters e regras de negócio |
| `repositories/` | `interface XxxRepository` + namespace com tipos auxiliares (`Params`, `Result`, `DbRow`, `Filters`) |
| `services/` | `interface XxxService` + namespace `{ Params, Result }` para **helpers granulares de use case** sem I/O direto. Mesmo shape de use case (`execute(params): Promise<Either<AbstractError, T>>`), mas não exposto em `index.cds` — é orquestração interna entre use cases. **Apenas contratos**; implementação vive em `application/services/`. |
| `use-cases/actions/` | Contratos de use cases de comando — um por `action` CDS |
| `use-cases/functions/` | Contratos de use cases de consulta — um por `function` CDS |
| `use-cases/entity-events/<entidade>/` | Contratos de hooks de entidade (before/after CRUD) |
| `utils/translator.ts` | Interface `Translator` + namespace `Translator.LanguageContext`/`TranslateParams` |
| `utils/get-user.ts` | Namespace `GetUser` com `LoggedUser`, `RequestWithUser`, `IdpLoggedUser` |
| `utils/get-environment.ts` | Namespace `GetEnvironment` com o tipo `Environment` (shape de credentials BTP) |

## Regras de ouro

1. **Nenhuma implementação no domain.** Repositórios, adapters, hydrators e utils existem aqui apenas como `interface`. A única exceção autorizada são os **models**, que são classes ricas com lógica de negócio pura (validação, formatação, serialização) — sem I/O.
2. **`Either<AbstractError, T>` é o tipo canônico de `Result` em use cases.** Importado direto de `@sweet-monads/either` — sem wrapper local. Quem produz `left()`/`right()` é a application layer; o domain só declara o tipo.
3. **`AbstractError` + 6 subclasses HTTP obrigatórias** (`BadRequestError` 400, `UnauthorizedError` 401, `ForbiddenError` 403, `NotFoundError` 404, `ConflictError` 409, `ServerError` 500). 502/503 opcionais para integrações externas. Nomes em inglês alinhados ao status HTTP — proibido fundir 401 em 403.
4. **Models são ricos.** Cada `XxxModel` concentra (a) validações internas, (b) formatações de dados, (c) serialização/deserialização. Models anêmicos (type alias sem classe) são **proibidos**. Lógica que envolva I/O nunca vive no model — vai para `application/`.
5. **Factory canônica do model: `static with(props)`.** Esta é a **única porta de construção** (input → model). Nomes alternativos como `from`, `fromDbRow`, `basic`, `static forCreate(props)`, `static forUpdate(props)` são **proibidos como factory** (anti-padrão MRO/RVE). Quando houver origem variada (raw da API externa, row do DB, payload bruto), criar overloads do `with` ou métodos auxiliares com prefixo `withFrom*` — sem multiplicar nomes de factory.
6. **Serialização é instance method, não factory.** Métodos do tipo `toObject()`, `toJSON()`, `toRow()`, `toDraftObject()`, `toCreationObject()`, `toUpdateObject()`, `forCreate()`, `forUpdate()` são **métodos de instância** legítimos quando o payload de saída diverge por operação (criação, update, draft, response HTTP, INSERT/UPDATE no DB). A regra geral é: **se constrói o model a partir de algo, é factory e tem que se chamar `with` (ou `withFrom*`); se produz algo a partir do model, é instance method e pode ter o nome semântico que descrever a saída**.
7. **Use case = `interface XxxUseCase { execute(params): Promise<XxxUseCase.Result> }` + `namespace XxxUseCase { Params, Result }`.** O método é sempre `execute`; o `Result` é sempre `Either<AbstractError, T>`.
8. **Repository = `interface XxxRepository`** com tipos auxiliares no `namespace XxxRepository { DbRow, Filters, Patch, ... }`. Mesmo padrão dos demais standards do monorepo (`infrastructure-layer/repositories/conventions.md`).
9. **Service no domain = só contrato, sem I/O.** `interface XxxService { execute(params): Promise<XxxService.Result> }` + `namespace XxxService { Params, Result }`. Mesmo shape de use case, porém **não exposto em `index.cds`** — é granularidade interna entre use cases (ex.: `PermissionCheckerService`, `ImportDataLoadService`). **Implementação concreta em `domain/services/` é proibida** (anti-padrão MRO `OciLineBuilders`); toda implementação vai para `application/services/`. **Service que toca rede/disco/SDK/framework não é service — é `adapter`** (vai em `domain/adapters/`).
10. **Hydrator = `interface XxxHydrator { hydrate(params): Result }`.** Grafia canônica é `hydrators/` + `hydrate()` (com "y") — `hidrators/`/`hidrate()` (com "i") é typo histórico de LE44/Suzano.
11. **Sem barrel exports** dentro do domain — exceto:
    - `errors/index.ts` (reexporta `AbstractError` + subclasses) — único barrel obrigatório.
    - `use-cases/entity-events/<entidade>/index.ts` (reexporta os contratos da entidade) — convenção compartilhada com a application layer.
12. **Sem framework.** O domain **não importa** `@sap/cds`, `@models/*`, `@cds-models/*`, `@sap/textbundle`, `@sap/xsenv` nem nenhum SDK externo. Quando precisar do tipo de um `Request`/`Transaction`/`EventContext` do CAP, declare wrapper local no namespace do contrato (`XxxHydrator.MutableRequest`, `XxxRepository.Transaction`).
13. **Imports NodeNext exigem extensão `.js`** explícita: `import type { PriceListRepository } from '@/domain/repositories/price-lists.js';`.

## Anti-padrão crítico — pastas proibidas

Pastas que **não devem existir** em `src/domain/` e que historicamente apareceram nos projetos modelo:

| Pasta | Motivo | Substituto canônico |
|---|---|---|
| `validators/` | Validação fica encapsulada nos **models** (`validate()`, `validateForCreate()`, etc.). O tipo `{ hasError, errorMessages }` é declarado localmente no model. | Método `validate()` no `XxxModel` |
| `middlewares/` | Middleware é conceito de execução, não de regra de negócio. Pertence à `application/` ou `infrastructure/`. | `application/` (pipeline de step) ou `infrastructure/` (adapter) |
| `constants/` | Constantes locais ficam como `private readonly` na classe que consome (application). Enums de domínio ficam dentro de `models/`. | `private readonly` na classe; enum no model |
| `extractors/` | Leitura de contexto runtime (env, user) fica em `domain/utils/<nome>.ts` (contrato) + `infrastructure/utils/<nome>.ts` (impl). | `utils/get-user.ts`, `utils/get-environment.ts` |
| `formatters/` | Formatação genérica fica em `utils/`; formatação específica de campo fica como método do **model**. | `utils/` ou método do model |
| `types/` | Tipos auxiliares vivem nos namespaces dos próprios contratos (`XxxRepository.DbRow`, `XxxUseCase.Params`). Sem pasta dedicada. | Namespace da interface |
| `external-apis/` (raiz) | Contratos de API externa vivem em `adapters/external-api/<sistema>/` — espelha a infra-layer. | `adapters/external-api/<sistema>/` |
| `translation/` | Renomeada para `utils/translator.ts` — espelha a infra-layer. | `utils/translator.ts` |
| `parsers/` (raiz) | Parsers vivem em `adapters/parsers/`. | `adapters/parsers/` |

## Naming

| Elemento | Convenção | Exemplo |
|---|---|---|
| Pasta | `kebab-case` (plural para agrupadores) | `repositories/`, `use-cases/`, `entity-events/`, `external-api/` |
| Arquivo | `kebab-case.ts` (singular para entidade, plural para repository) | `loss-provision.ts`, `price-lists.ts`, `before-create.ts` |
| Interface (model) | `XxxModel` (classe, não interface) | `PriceListModel`, `LossProvisionBaseModel` |
| Tipo de propriedades do model | `XxxProps` | `PriceListProps`, `LossProvisionBaseProps` |
| Interface Repository | `XxxRepository` | `PriceListRepository`, `PartNumberRepository` |
| Interface Use Case | `XxxUseCase` ou `<Evento><Ação><Entidade>UseCase` | `CheckoutUseCase`, `BeforeCreatePriceListUseCase` |
| Interface Service | `XxxService` | `PermissionCheckerService`, `ImportDataLoadService`, `ValidateServiceNumberService` |
| Interface Adapter (API externa) | `XxxApi` | `CompanyApi`, `CarrierApi`, `SapHttpClient` |
| Interface Adapter (técnico) | `XxxAdapter` ou nome próprio | `UnitOfWork`, `Telemetry`, `ExcelParser` |
| Interface Hydrator | `<Evento><EntidadePlural>Hydrator` | `BeforeReadUserPreferencesHydrator` |
| Classe de erro | `XxxError` | `BadRequestError`, `NotFoundError`, `ServerError` |
| Namespace colateral | mesmo nome da interface (PascalCase) | `namespace XxxRepository { DbRow, Filters }` |
| Método público de hydrator | `hydrate` (com "y") | `public hydrate(params): void` |
| Método público de use case | `execute` | `public execute(params): Promise<Result>` |
| Factory de model | `static with(props)` | `PriceListModel.with(props)` |

**Sufixos proibidos no domain:** `Impl`, `Concrete`, `Fake`, `Mock`, `Dto`, `Entity`. Esses pertencem à infrastructure/application.

## Quem importa o quê

| Camada que importa | Pode importar de `domain/`? |
|---|---|
| `domain/` (própria) | ✅ Sim — entre subpastas livremente (use case importa repository, model importa erros, etc.) |
| `application/` | ✅ Sim — toda implementação consome contratos do domain |
| `infrastructure/` | ✅ Sim — toda implementação consome contratos do domain |
| `presentation/` | ✅ Sim — controllers usam `XxxUseCase.Params` |
| `main/` (factories) | ✅ Sim — factories instanciam `XxxImpl implements XxxInterface` |

| Camada que o `domain/` importa | Permitido? |
|---|---|
| `application/`, `infrastructure/`, `presentation/`, `main/` | ❌ Nunca |
| `@sap/cds`, `@models/*`, `@cds-models/*` | ❌ Nunca — usar wrapper local |
| `@sap/textbundle`, `@sap/xsenv`, SDKs (AWS, etc.) | ❌ Nunca |
| `@sweet-monads/either` | ✅ Sim — única dependência externa aceita |

## Tipagem e constantes

Estas regras espelham a [seção "Tipagem e constantes" do README da infrastructure-layer](../infrastructure-layer/README.md#tipagem-e-constantes), aplicadas ao domain.

### Regra 1 — Tipos auxiliares de contrato vivem no namespace da própria interface

`Params`, `Result`, `DbRow`, `Filters`, `Patch`, `MutableRequest` — qualquer tipo auxiliar de um contrato fica no `namespace XxxInterface { ... }` declarado no mesmo arquivo da interface. Sem arquivos `xxx.types.ts` separados; sem pasta `types/`.

```typescript
// src/domain/repositories/price-lists.ts
export namespace PriceListRepository {
    export type DbRow = {
        tenant_id: string;
        id: string;
        name: string;
    };
    export type Patch = Partial<DbRow>;
    export type Filters = { status?: string; createdAfter?: string };
}

export interface PriceListRepository {
    findById(tenantId: string, id: string): Promise<PriceListModel | null>;
    findAllByTenant(tenantId: string): Promise<PriceListModel[]>;
    save(model: PriceListModel): Promise<void>;
    update(tenantId: string, id: string, patch: PriceListRepository.Patch): Promise<void>;
    delete(tenantId: string, id: string): Promise<void>;
}
```

### Regra 2 — Sem `interface`/`type` solto fora dos contratos canônicos

Em `domain/`, todo `interface` ou `type` exportado pertence a:
- Um **contrato** (`XxxRepository`, `XxxUseCase`, `XxxApi`, `XxxHydrator`, `Translator`).
- Um **namespace** colateral desse contrato (`XxxRepository.DbRow`).
- Um **`XxxProps`** ao lado de `class XxxModel`.

Sem arquivos `xxx.types.ts`, sem pasta `types/`, sem enums órfãos. Enums de domínio ficam dentro de `models/<entidade>.ts` ou exportados pelo namespace da interface que os usa.

### Regra 3 — Constantes vivem no consumidor, não no domain

Não declarar `const FOO = 'bar'` em arquivos de `domain/`. Constantes de configuração local pertencem à classe que consome (`private readonly` na application/infra). Enums de domínio são exceção legítima — declarados ao lado do model que os usa.

❌ **Errado — `const` solta no domain:**

```typescript
// src/domain/constants/field-max-lengths.ts
export const FIELD_MAX_LENGTHS = { name: 100, description: 500 };
```

✅ **Certo — `private readonly` na classe que consome:**

```typescript
// src/application/use-cases/actions/save-product.ts
export class SaveProductUseCaseImpl extends BaseUseCaseImpl implements SaveProductUseCase {
    private readonly NAME_MAX_LENGTH = 100;
    private readonly DESCRIPTION_MAX_LENGTH = 500;
    // ...
}
```

## Documentos desta seção

- [models/README.md](./models/README.md) — `XxxProps`, `class XxxModel`, factory `with(props)`, getters, `validate()`, serialização
- [errors/README.md](./errors/README.md) — `AbstractError` + 6 subclasses HTTP + barrel + uso com `Either`
- [repositories/README.md](./repositories/README.md) — `interface XxxRepository` + namespace com `DbRow`, `Filters`, `Patch`
- [use-cases/README.md](./use-cases/README.md) — `interface XxxUseCase` + namespace, `actions/`, `functions/`, `entity-events/`
- [services/README.md](./services/README.md) — `interface XxxService` + namespace (helper granular de use case sem I/O direto)
- [adapters/README.md](./adapters/README.md) — contratos de `external-api/`, `parsers/`, `unit-of-work.ts`
- [hydrators/README.md](./hydrators/README.md) — contratos `Before*Hydrator` com `hydrate()` (grafia "y")
- [utils/README.md](./utils/README.md) — `Translator`, `GetUser`, `GetEnvironment` (contratos de utilitários)
- [testing.md](./testing.md) — testes de domain (somente models e utils com lógica) + `scenarios-overview.md`
