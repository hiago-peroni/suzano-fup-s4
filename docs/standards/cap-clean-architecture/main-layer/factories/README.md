# factories/

A pasta `factories/` é o **composition root** da aplicação. É aqui que as implementações concretas de todas as camadas são instanciadas e compostas via DI manual.

## Regra fundamental

> Uma factory **nunca** contém lógica de negócio. Ela apenas instancia, injeta e retorna. Se há um `if` que não seja `NODE_ENV`, a lógica pertence a outra camada.

## Shape universal

Toda factory segue o mesmo contrato:

```typescript
// Assinatura padrão
export const make<Nome> = (): <InterfaceDomínio> => {
    return new <NomeImpl>(...dependências);
};

// Singleton exportado (quando usado em routes/)
export const <nomeInstância> = make<Nome>();
```

## Tipos de factory e quando cada um existe

| Tipo | Quando criar | Quem consome |
|---|---|---|
| `adapters/` | Implementações de `domain/adapters/*` (HTTP, storage, telemetria, parsers) | `use-cases/` |
| `controllers/` | Um por controller de `presentation/controllers/*` | `routes/index.ts` |
| `repositories/` | Um por repositório de `domain/repositories/*` | `use-cases/` |
| `services/` | Quando `domain/services/*` existe e precisa de DI | `use-cases/` |
| `use-cases/` | Um por use-case de `application/use-cases/*` | `controllers/` |
| `utils/` | Singletons transversais (translator i18n) | `routes/index.ts`, `use-cases/` |

## Subpastas de `controllers/` e `use-cases/`

As subpastas espelham os tipos de operação do CAP:

| Subpasta | Tipo CAP | Registro em `routes/index.ts` |
|---|---|---|
| `actions/` | `action` no CDS | `service.on('myAction', ...)` |
| `functions/` | `function` no CDS | `service.on('myFunction', ...)` |
| `entity-events/<entidade>/` | Hooks before/after de entidade | `service.before('UPDATE', 'Entity', ...)` |

**`controllers/` e `use-cases/` são sempre espelhados:** para cada `controllers/actions/X.ts` existe exatamente um `use-cases/actions/X.ts` com o mesmo nome de arquivo.

## Naming

- Arquivos: `kebab-case.ts` (ex.: `approve-price-list.ts`, `price-lists.ts`).
- Funções factory: `make<NomePascalCase>` (ex.: `makeApprovePriceListController`).
- Singletons exportados: `<nomeCamelCase>` (ex.: `approvePriceListController`).
- Repositórios: nome da entidade no **plural** (`price-lists.ts`, `tenants.ts`).
- `entity-events/<entidade>/`: entidade em kebab-case plural (`price-lists/`, `tenant-erp-connections/`).

## Documentos desta seção

- [adapters.md](./adapters.md)
- [controllers.md](./controllers.md)
- [repositories.md](./repositories.md)
- [services.md](./services.md)
- [use-cases.md](./use-cases.md)
- [utils.md](./utils.md)
