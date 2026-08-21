# Infrastructure layer

A infrastructure layer é a camada que **executa o trabalho sujo** da aplicação: persiste dados no CDS/HANA, integra com APIs externas (SAP S4, REST/JSON, AWS), parseia arquivos, gerencia transações, lê variáveis de ambiente, traduz mensagens i18n e muta requests CAP via hydrators.

Toda implementação aqui satisfaz uma interface definida em `domain/`. Nenhuma regra de negócio vive aqui — apenas detalhes técnicos de I/O.

> **Regra de isolamento:** a infrastructure layer importa **apenas** de `domain/` e de outras partes de `infrastructure/`. **Nunca** de `application/`, `presentation/` ou `main/`.

## Estrutura

```
src/infrastructure/
├── adapters/
│   ├── external-api/
│   │   ├── csn/                            → CSN versionados (cds import — apenas OData com $metadata)
│   │   │   └── COMPANY_API.csn
│   │   ├── <dominio>/
│   │   │   ├── <dominio>-api.ts            → XxxApiImpl (real)
│   │   │   └── fake-<dominio>-api.ts       → FakeXxxApiImpl (fixture)
│   │   └── http/
│   │       └── sap-cloud-sdk-http-client.ts → SapCloudSdkHttpClient (genérico Cloud SDK)
│   ├── parsers/
│   │   └── <formato>-parser.ts             → ExcelSpreadsheetParser, CsvParser, etc.
│   └── unit-of-work/
│       └── cds-unit-of-work.ts             → CdsUnitOfWork (batch transacional)
├── repositories/
│   ├── models/
│   │   └── <entidade-plural>.ts            → repositórios de tabelas próprias (db.models.*)
│   └── replication/
│       └── <entidade-plural>.ts            → repositórios de tabelas replicadas (db.replication.*)
├── hydrators/
│   └── <entidade-plural>/
│       ├── before-read.ts                  → BeforeReadXxxHydratorImpl
│       ├── before-create.ts
│       └── before-update.ts
└── utils/
    ├── translator/
    │   ├── translator.ts                   → TranslatorImpl + ResourceManager + ALS
    │   └── i18n/
    │       ├── i18n.properties
    │       ├── i18n_pt.properties
    │       └── i18n_es.properties
    ├── get-user.ts                         → função utilitária: extrai usuário do request CAP
    └── get-environment.ts                  → função utilitária: lê credentials BTP via xsenv
```

## Responsabilidades

| Subpasta | Responsabilidade |
|---|---|
| `adapters/external-api/` | Wrappers de APIs externas — OData S4 (`cds.connect.to`), REST/JSON (`kind: 'rest'`), AWS SDK |
| `adapters/external-api/csn/` | Artefatos `.csn.json` gerados por `cds import` (apenas quando `$metadata` disponível) |
| `adapters/parsers/` | Parsers de arquivos externos (Excel, CSV, JSON) com mapeamento para estruturas de domínio |
| `adapters/unit-of-work/` | Implementação de `UnitOfWork` para batch transacional via `cds.tx` (apenas processing services) |
| `repositories/models/` | Repositórios de tabelas próprias do projeto (`db.models.*`) — acesso direto ao CDS via `cds.ql.*` / `cds.run`, leitura e escrita |
| `repositories/replication/` | Repositórios de tabelas replicadas de sistemas externos (`db.replication.*`) — somente leitura; a escrita é feita pelo job de replicação CDS nativo |
| `hydrators/` | Mutadores de `request.query.SELECT.where` ou `request.body` antes da execução do CDS |
| `utils/translator/` | `TranslatorImpl` com `@sap/textbundle` + `AsyncLocalStorage` para i18n por request |
| `utils/get-user.ts` | Função utilitária pura: extrai o usuário logado do request CAP (sem classe, sem DI) |
| `utils/get-environment.ts` | Função utilitária pura: lê credentials do BTP via `@sap/xsenv` (sem classe, sem DI) |

## Regras de ouro

1. **Nenhuma regra de negócio vive aqui.** Toda lógica de domínio pertence a `domain/models/` ou `application/use-cases/`. A infra apenas executa I/O e mapeia raw → model.
2. **Repositórios e adapters propagam exceção.** Nada de `Either<L,R>` na infrastructure — quem captura é o `try/catch` do use case via `handleError` da `BaseUseCaseImpl`. Lançar `throw new ServerError(...)` é aceitável para enriquecer a mensagem; deixar propagar a exceção crua também é aceitável.
3. **Contratos vivem em `domain/`.** Toda interface implementada por `infrastructure/` tem origem em `domain/repositories/`, `domain/adapters/`, `domain/hydrators/`, `domain/utils/`, etc. Nunca declarar `interface`/`type` exportado dentro de `infrastructure/`.
4. **DI via constructor apenas quando há dependência real.** Repositórios sem DI por padrão (CDS é singleton global). Adapters com dependências externas (HTTP client, UoW, telemetry) recebem via constructor com `private readonly` tipado pela interface do domínio.
5. **`console.error` apenas — nunca `console.log`.** O `stdout` é reservado para o framework CAP e JSON-RPC. Logs estruturados via adapter de telemetria injetado.
6. **Sem `infrastructure/services/`.** Services pertencem a `application/services/`. Operações com I/O que orquestram múltiplos repositórios/adapters viram services de aplicação consumindo as interfaces do domínio.
7. **Sem `infrastructure/extractors/` e `infrastructure/formatters/`.** Leitores de contexto runtime viram funções em `infrastructure/utils/`; formatação de primitivos vira método de domain model.
8. **Sem barrel `index.ts` por padrão.** Factories importam por caminho direto (`@/infrastructure/repositories/price-lists.js`). Barrels parciais são anti-padrão.
9. **Imports NodeNext exigem extensão `.js`.** `import { TranslatorImpl } from '@/infrastructure/utils/translator/translator.js'` — mesmo em arquivos `.ts`.
10. **Nenhum `infrastructure/types/`.** Sem subpasta de tipos. Sem `type`/`interface` local. Ver seção "Tipagem e constantes" abaixo.

## Tipagem e constantes

Estas 3 regras espelham o padrão da application layer ([`application-layer/use-cases/base.md`](../application-layer/use-cases/base.md) seção "Anti-padrão: tipos e interfaces declarados no arquivo da application layer") e valem para **toda** a infrastructure layer — adapters, repositories, hydrators e utils.

### Regra 1 — Sem DTOs no infrastructure

O termo "DTO" não faz parte do vocabulário deste monorepo. Não declarar `interface XxxDto`, `interface XxxResponse`, `type XxxRaw` ou similares dentro de arquivos da infrastructure. Tudo passa por **models de domínio** (`XxxModel` + `XxxProps`) e tipos em **namespaces do contrato no domain**.

| Conceito | Onde fica |
|---|---|
| Shape de um registro raw (API externa, linha de banco, payload de arquivo) | `domain/models/<contexto>/<entidade>.ts` exportando `XxxProps`/`XxxRow` |
| Envelope de response (ex.: `{ d: { results: [...] } }`, `{ items: [...], total: N }`) | Namespace da interface do adapter/repositório em `domain/adapters/...` ou `domain/repositories/...` |
| `Params`/`Result` de método público | Namespace do contrato (`XxxApi.FindByXxxParams`, `XxxRepository.Filters`) |
| Auxiliares internos do contrato (intents do UoW, filtros dinâmicos, etc.) | Namespace do contrato (`UnitOfWork.Intent`, `DataLoadRepository.DbRow`) |
| Transformação raw → domínio | Método `XxxModel.with()` no `domain/models/...` |

### Regra 2 — Interfaces e tipagens exclusivamente do domain layer

Todo `import type`/`import interface` em arquivo da infrastructure vem de `@/domain/...`. Zero declaração de `type`/`interface` dentro de `infrastructure/` — nem locais, nem exportados, nem `infrastructure/types/`. Quando precisar de narrowing ad-hoc em runtime, prefira navegação via `key in current` em vez de cast com tipo inline.

❌ **Errado — tipo declarado dentro da infra:**

```typescript
// src/infrastructure/repositories/data-loads.ts
type DataLoadDbRow = {
    id: string;
    tenant_id: string;
};

export class DataLoadRepositoryImpl implements DataLoadRepository {
    // ...
}
```

✅ **Certo — tipo no domain, importado pela infra:**

```typescript
// src/domain/repositories/data-loads.ts
export namespace DataLoadRepository {
    export type DbRow = {
        id: string;
        tenant_id: string;
    };
}

// src/infrastructure/repositories/data-loads.ts
import type { DataLoadRepository } from '@/domain/repositories/data-loads.js';

const rows: DataLoadRepository.DbRow[] = await cds.run(/* ... */);
```

### Regra 3 — Constantes e variáveis dentro da classe como `private readonly`

Sem `const FOO = ...` em module-level dentro de `infrastructure/`. Toda constante de unidade é `private readonly` da classe. Mesmo padrão da application layer (regra 6 do [`application-layer/README.md`](../application-layer/README.md)). Variáveis mutáveis também ficam dentro da classe — sem `let cachedXxx` em module-level.

❌ **Errado — `const`/`let` em module-level:**

```typescript
const DESTINATION_NAME = 'S4DEST_PROD';
const FAKE_DATA = [ /* ... */ ];
let cachedEnvironment: Environment | null = null;

export class XxxApiImpl implements XxxApi { /* ... */ }
```

✅ **Certo — propriedades `private readonly` da classe:**

```typescript
export class XxxApiImpl implements XxxApi {
    private readonly DESTINATION_NAME = 'S4DEST_PROD';
    private readonly FAKE_DATA: XxxProps[] = [ /* ... */ ];
    private cachedEnvironment: Environment | null = null;
    // ...
}
```

**Exceções toleradas:**

1. **Factories (`main/factories/`)** — singleton lazy com `let instance: Xxx | null = null` é aceitável **somente** dentro de factories. Essa é a camada de composição; não vale para `infrastructure/`.
2. **Funções utilitárias puras (`utils/get-*.ts`)** — não têm classe onde colocar `private readonly`. Helpers internos seguem o padrão `const arrow` documentado em [`docs/standards/code-style/typescript-syntax.md`](../../code-style/typescript-syntax.md), que são **helpers funcionais** (declarações de função), não constantes de dado. Cache module-level (`let cachedEnvironment`) é tolerado **apenas** nesses arquivos `utils/get-*.ts`.

## Naming

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo | `kebab-case.ts` | `price-lists.ts`, `before-read.ts`, `cds-unit-of-work.ts` |
| Classe (regra geral) | `PascalCase` + `Impl` | `PriceListRepositoryImpl`, `TranslatorImpl` |
| Classe — adapter externo (API) | `PascalCase` + `ApiImpl` | `CompanyApiImpl`, `CarrierApiImpl`, `SapHoursCyclesApiImpl` |
| Classe — fake de API externa | `Fake` + `PascalCase` + `ApiImpl` | `FakeCompanyApiImpl`, `FakeCarrierApiImpl` |
| Classe — adapter genérico HTTP | `PascalCase` + `HttpClient` | `SapCloudSdkHttpClient`, `RestHttpClient` |
| Classe — hydrator | `PascalCase` + evento + entidade + `HydratorImpl` | `BeforeReadUserPreferencesHydratorImpl` |
| Classe — UoW | `<Tecnologia>UnitOfWork` (sem `Impl`) | `CdsUnitOfWork` |
| Classe — parser | `<Formato>Parser` (sem `Impl`) | `ExcelSpreadsheetParser` |
| Função utilitária | `kebab-case.ts` + `camelCase` | `get-user.ts` → `getUser()`, `get-environment.ts` → `getEnvironment()` |
| Pasta de entidade em repositories/hydrators | `kebab-case-plural` | `price-lists/`, `user-preferences/`, `purchase-order-items/` |
| Pasta de domínio em adapters/external-api | `kebab-case-singular` | `company/`, `carrier/`, `email/` |

> **Por que sufixo `Impl` é a regra mas hydrators e o adapter HTTP genérico têm variações?**
> A regra geral é `*Impl`. Hydrators acumulam evento + entidade no nome (`BeforeReadXxxHydratorImpl`) — o sufixo `Impl` se mantém. O adapter HTTP genérico (`SapCloudSdkHttpClient`) tem nome próprio porque é único na codebase, sem variação real/fake; quando há essa variação (APIs de domínio), o padrão `XxxApiImpl` + `FakeXxxApiImpl` aplica.

## Tratamento de erros

A camada infrastructure **não** retorna `Either<L,R>`. O contrato com a application layer é:

- **Caminho feliz** → retornar o valor tipado (model, array, void).
- **Não encontrado** → retornar `null` (para `findById`) ou `[]` (para `findAll`) — nunca lançar `NotFoundError` daqui.
- **Erro de I/O** → propagar a exceção. O `try/catch` do use case captura via `handleError` da `BaseUseCaseImpl` e converte em `ServerError`.
- **Erro com mensagem específica de sistema externo** → lançar `throw new ServerError(stack, message)` enriquecendo com detalhe do erro SAP/OData (ver `SapCloudSdkHttpClient` no padrão de adapters HTTP).

Anti-padrões observados (e proibidos):

- **`catch { return null }`** sem log — oculta falhas reais de infraestrutura.
- **`Either<AbstractError, T>`** em métodos de repositório/adapter — esse contrato pertence à application layer.
- **`throw new BadRequestError(...)`** dentro de repositório/adapter — erros de validação pertencem ao use case, não à infra.

## Quem importa o quê

| Camada que importa | Pode importar de `infrastructure/`? |
|---|---|
| `domain/` | ❌ Nunca |
| `application/` | ❌ Nunca (consome apenas interfaces de `domain/`) |
| `presentation/` | ❌ Nunca |
| `main/` (factories) | ✅ Sim — único consumidor direto, instancia as implementações |
| `infrastructure/` (própria camada) | ✅ Sim — adapter de domínio pode receber adapter genérico via DI |

## Documentos desta seção

- [adapters/README.md](./adapters/README.md) — visão geral dos adapters
  - [adapters/external-apis.md](./adapters/external-apis.md) — `*ApiImpl`, `Fake*ApiImpl`, `cds.requires`, CSN
  - [adapters/parsers.md](./adapters/parsers.md) — parsers de arquivo (Excel, CSV)
  - [adapters/unit-of-work.md](./adapters/unit-of-work.md) — `CdsUnitOfWork` (batch transacional)
- [repositories/README.md](./repositories/README.md) — visão geral de repositórios
  - [repositories/conventions.md](./repositories/conventions.md) — `cds.ql.*`, `ENTITY`, mapeamento p/ Models, `forUpdate`
  - [repositories/replication.md](./repositories/replication.md) — subpasta `sap/` para `db.replication.*`
- [hydrators/README.md](./hydrators/README.md) — mutadores de `SELECT.where` e `request.body`
- [utils/README.md](./utils/README.md) — visão geral dos utilitários
  - [utils/translator/README.md](./utils/translator/README.md) — `TranslatorImpl` + ALS
  - [utils/translator/i18n.md](./utils/translator/i18n.md) — `.properties`, bundles, `.cdsrc.json`
  - [utils/get-user.md](./utils/get-user.md) — extração do usuário logado do request CAP
  - [utils/get-environment.md](./utils/get-environment.md) — leitura de credentials BTP via `xsenv`
- [testing.md](./testing.md) — `makeSut`, `BaseStub`, `scenarios-overview.md`
