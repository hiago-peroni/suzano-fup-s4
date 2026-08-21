# Adapters

Adapters são wrappers de serviços técnicos externos: APIs HTTP, parsers de arquivo, transações batch. Diferem de repositórios e hydrators em propósito e alvo:

| Caso | Onde vai |
|---|---|
| Persistir entidade própria via CDS (`cds.ql`) | `repositories/` |
| Chamar API externa SAP/REST/AWS | `adapters/external-api/<dominio>/` |
| Parsear arquivo Excel/CSV/JSON | `adapters/parsers/` |
| Acumular operações para transação batch | `adapters/unit-of-work/` |
| Mutar `SELECT.where` ou `request.body` antes do CDS | `hydrators/` |

**Repositórios** falam diretamente com o CDS sobre entidades da aplicação (`cds.ql.SELECT`, `INSERT`, `UPDATE`). **Adapters** cruzam a fronteira da aplicação em direção a sistemas terceiros — SAP S4, serviços REST, arquivos no filesystem, provedores de cloud. **Hydrators** são uma categoria distinta: mutam o `request` CAP antes que o framework execute — sem chamada externa, sem I/O de arquivo.

## Subpastas canônicas

```
src/infrastructure/adapters/
├── external-api/                   → APIs externas (OData, REST, SDKs)
│   ├── csn/                        → CSN versionados (cds import — só com $metadata)
│   ├── http/                       → Adapter HTTP genérico (Cloud SDK)
│   └── <dominio>/                  → Adapter de domínio real + fake
├── parsers/                        → Parsers de arquivo (Excel, CSV, JSON)
└── unit-of-work/                   → Batch transacional via cds.tx
```

## Naming

| Elemento | Convenção | Exemplo |
|---|---|---|
| Adapter de API externa (real) | `PascalCase` + `ApiImpl` | `CompanyApiImpl`, `CarrierApiImpl` |
| Adapter de API externa (fake) | `Fake` + `PascalCase` + `ApiImpl` | `FakeCompanyApiImpl`, `FakeCarrierApiImpl` |
| Adapter HTTP genérico | `PascalCase` + `HttpClient` | `SapCloudSdkHttpClient` |
| Parser de arquivo | `<Formato>Parser` (sem `Impl`) | `ExcelSpreadsheetParser`, `CsvParser` |
| Unit of Work | `<Tecnologia>UnitOfWork` (sem `Impl`) | `CdsUnitOfWork` |

## Tratamento de erro

Adapters **propagam** exceção — não retornam `Either<L,R>`. O `try/catch` do use case captura via `handleError` da `BaseUseCaseImpl`.

- **Erro genérico de I/O** → deixar propagar.
- **Erro com mensagem de sistema externo (SAP, OData)** → lançar `throw new ServerError(stack, sapMessage)` enriquecendo com `error.response.data.error.message.value`.

## Injeção de dependência

DI via constructor **apenas quando há dependência real**. Exemplos:

- `CdsUnitOfWork` recebe `Telemetry` (emite eventos de observabilidade).
- `SapStockBalanceApiImpl` recebe `SapHttpClient` (abstrai a camada de transporte HTTP).
- `ExcelSpreadsheetParser` **não** recebe DI por padrão (sem dependências externas).

Todos os tipos injetados são **interfaces do domínio**, nunca implementações concretas.

## Tipagem e constantes

Todo adapter segue as 3 regras globais da [seção "Tipagem e constantes" do README da infrastructure-layer](../README.md#tipagem-e-constantes):

1. **Sem DTOs** — todo shape de raw response, params e result vive em `domain/models/<sistema>/` ou no namespace da interface do adapter (`XxxApi.RawResponse`, `XxxApi.Params`).
2. **Interfaces e tipagens só do domain** — zero declaração de `type`/`interface` dentro de `infrastructure/adapters/`.
3. **Constantes como `private readonly`** — `DESTINATION_NAME`, `SERVICE_URL`, `FAKE_DATA`, schemas Zod, etc., todos como propriedades da classe. Nunca `const` em module-level.

Cada documento de adapter detalha como aplicar essas regras na sua especialidade (`external-apis.md`, `parsers.md`, `unit-of-work.md`).

## Anti-padrão: selector dentro do adapter (Suzano facade)

O padrão proibido consiste em criar uma fachada `*AdapterImpl` que resolve internamente qual implementação usar com base em `NODE_ENV`:

```typescript
// ❌ Anti-padrão — selector dentro do adapter
export class CompanyAdapterImpl implements CompanyApi {
    private readonly delegate: CompanyApi;

    constructor() {
        // Selector escondido dentro do adapter — proibido
        this.delegate =
            process.env.NODE_ENV !== 'production'
                ? new FakeCompanyApiImpl()
                : new ConcreteCompanyApiImpl();
    }
}
```

O selector **fica na factory** (`main/factories/adapters/`), que é o único lugar autorizado a decidir qual implementação instanciar:

```typescript
// ✅ Canônico — selector na factory
export const makeCompanyApi = (): CompanyApi =>
    process.env.NODE_ENV !== 'production'
        ? new FakeCompanyApiImpl()
        : new CompanyApiImpl();
```

## Documentos desta seção

- [external-apis.md](./external-apis.md) — `*ApiImpl`, `Fake*ApiImpl`, `cds.requires`, CSN, SDKs próprios
- [parsers.md](./parsers.md) — parsers de arquivo (Excel, CSV, JSON com Zod)
- [unit-of-work.md](./unit-of-work.md) — `CdsUnitOfWork` (batch transacional)
