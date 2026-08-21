# Repositories

Repositórios implementam as interfaces declaradas em `domain/repositories/*` e são o **único ponto de acesso** da aplicação a `cds.ql.*` para entidades CDS do serviço (tabelas em `db/*.cds`). Nenhuma outra camada executa queries CDS diretamente — toda operação de leitura ou escrita passa por aqui.

## Repositório vs Adapter externo

| Critério | Repositório | Adapter externo |
|---|---|---|
| Entidade alvo | Tabela própria (`db.models.*`) ou replicada (`db.replication.*`) | Sistema externo (S4, REST API, AWS) |
| Mecanismo | `cds.ql.*` + `cds.run` (CDS local) | `cds.connect.to(API_NAME)` + `.read/.send` ou Cloud SDK |
| Localização | `infrastructure/repositories/` | `infrastructure/adapters/external-api/` |
| Naming | `XxxRepositoryImpl` | `XxxApiImpl` |

## Estrutura

A subdivisão é por **namespace CDS** — `models/` espelha `db.models.*` (tabelas próprias do projeto, leitura e escrita) e `replication/` espelha `db.replication.*` (tabelas replicadas de sistemas externos, somente leitura). Não há `repositories/` "raiz" misturando os dois — toda entidade fica em uma das duas subpastas.

```
infrastructure/repositories/
├── models/                         → tabelas próprias (db.models.*)
│   ├── price-lists.ts              → PriceListRepositoryImpl
│   ├── tenants.ts                  → TenantRepositoryImpl
│   └── part-numbers.ts             → PartNumberRepositoryImpl (recebe UoW via DI)
└── replication/                    → tabelas replicadas (db.replication.*)
    ├── plants.ts                   → PlantRepositoryImpl
    ├── materials.ts                → MaterialRepositoryImpl
    └── suppliers.ts                → SupplierRepositoryImpl
```

> A subpasta `replication/` substitui o antigo padrão `sap/` — replicação não é exclusiva do SAP. Qualquer espelho local de master data de sistema externo (S4, SuccessFactors, Salesforce, MDM próprio) entra em `replication/`.

| Subpasta | Namespace CDS | Operações | Quando criar |
|---|---|---|---|
| `models/` | `db.models.*` | Leitura e escrita | Toda entidade própria do serviço |
| `replication/` | `db.replication.*` | Somente leitura | Apenas se o projeto tiver tabelas de replicação |

## Naming

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo | kebab-case plural | `price-lists.ts`, `purchase-order-items.ts` |
| Classe | PascalCase + `RepositoryImpl` | `PriceListRepositoryImpl` |
| Interface (no domínio) | PascalCase + `Repository` | `PriceListRepository` |
| Constante `ENTITY` | UPPER_SNAKE como `private readonly` da classe | `private readonly ENTITY = 'db.models.PriceLists'` |
| Tipo de linha de banco | Namespace da interface no domínio | `PriceListRepository.DbRow` |
| Subpasta do namespace | `models/` ou `replication/` | — |

## Regras de ouro

1. **Sem DI por padrão.** O CDS é singleton global. Exceção: repositórios que usam `UnitOfWork` para operações batch (`constructor(private readonly uow: UnitOfWork) {}`).

2. **Propaga exceção.** Sem `try/catch` defensivo. Sem `Either<L,R>`. Quem captura é o use case via `handleError` da `BaseUseCaseImpl`.

3. **Anti-padrão: catch silencioso → null.** Oculta falhas reais de infraestrutura (timeout, constraint violation):

    ```typescript
    // ❌ ERRADO
    try {
        const row = await cds.run(cds.ql.SELECT.one.from(this.ENTITY).where({ id }));
        return row ? UserModel.with(row) : null;
    } catch {
        return null; // oculta falhas de infra (timeout, constraint violation)
    }
    ```

4. **`findById` retorna `null` se não encontrar; `findAll` retorna `[]`.** Não lançar `NotFoundError` do repositório — quem decide se ausência é erro é o use case.

5. **Sempre que o método retornar registros, retornar via domain model.** Todo método que devolve dados de uma entidade serializa via `XxxModel.with(raw)` antes de sair do repositório — nunca expor `DbRow` direto para a application layer. Métodos que **não retornam entidades** (`save`/`update`/`delete` com `Promise<void>`, `count`/`exists` com primitivos) ficam isentos — não há o que serializar. Detalhes em [conventions.md — Domain model como fronteira de retorno](./conventions.md#domain-model-como-fronteira-de-retorno).

6. **Uma entidade por repository.** Cada `XxxRepositoryImpl` opera sobre uma única `db.models.<Entidade>` (ou `db.replication.<Entidade>`). Não herdar entidades terciárias dentro do mesmo repositório. Joins são permitidos via `cds.ql.SELECT.from(this.ENTITY, columns => { ... })` — quando o join trouxer dados de entidade auxiliar, declarar essa entidade como `private readonly` ao lado da `ENTITY` pai. Detalhes em [conventions.md — Uma entidade por repository](./conventions.md#uma-entidade-por-repository).

7. **SQL pleno apenas quando o CAP não atende.** Regra geral: sempre usar o query builder (`cds.ql.SELECT/INSERT/UPDATE/DELETE`). SQL bruto via `cds.run({ sql, values })` é exceção justificada — TVFs HANA, stored procedures, hints específicos do banco. Quando usado, sempre como `private readonly XXX_SQL = '...'` na classe e com comentário explicando a justificativa. Detalhes em [conventions.md — SQL pleno como exceção](./conventions.md#sql-pleno-como-exceção).

8. **Constante `ENTITY` como `private readonly` da classe.** Centraliza o nome da tabela CDS e facilita renomear sem grep. Não declarar `const ENTITY = ...` em module-level — viola a [regra global de tipagem e constantes](../README.md#tipagem-e-constantes).

9. **`cds.ql.*` + `cds.run` é o padrão.** Globais `SELECT.from(...).forUpdate()` aceitos para locking HANA. `cds.ql.UPDATE.entity(...).set(...).where(...)` é o padrão para updates parciais.

10. **Tipos de linha de banco vivem no domain.** Declarar `XxxRepository.DbRow` no namespace da interface em `domain/repositories/<entidade>.ts` (ou reutilizar `XxxModel.Row` em `domain/models/<entidade>.ts`). Nada de `type XxxDbRow` ou `infrastructure/types/` dentro de `infrastructure/`.

11. **Sem barrel.** `factories/repositories/` importa por caminho direto (`@/infrastructure/repositories/price-lists.js`).

## Tratamento de erros

| Situação | Comportamento canônico |
|---|---|
| Linha não encontrada (`findById`) | `return null` |
| Lista vazia (`findAll`) | `return []` |
| Erro de conexão / timeout / constraint | Propagar exceção (não capturar) |
| Erro com mensagem específica útil | `throw new ServerError(stack, message)` opcional para enriquecer |

## Documentos desta seção

- [conventions.md](./conventions.md) — `cds.ql.*`, constante `ENTITY`, mapeamento para Models, `forUpdate`, TVF HANA, anti-padrões
- [replication.md](./replication.md) — subpasta `replication/` para entidades `db.replication.*` (espelhos locais de master data externo)
