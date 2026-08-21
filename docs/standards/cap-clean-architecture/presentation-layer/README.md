# Presentation layer

A presentation layer é a fronteira entre o framework CAP e o domínio da aplicação. Ela traduz requisições HTTP/OData em chamadas tipadas para os use cases e traduz as respostas de volta para o envelope que o CAP entende.

Nenhuma lógica de negócio vive aqui. Nenhuma chamada ao CAP (`req.reject`, `req.reply`, `service.on`) fica aqui. A presentation apenas **recebe, delega e devolve**.

## Estrutura

```
src/presentation/
└── controllers/
    ├── base/
    │   └── controller.ts     → BaseController, BaseControllerResponse, ErrorDetails
    ├── actions/
    │   └── <kebab-name>.ts   → um controller por action CDS
    ├── functions/
    │   └── <kebab-name>.ts   → um controller por function CDS
    └── entity-events/
        └── <entidade-plural>/
            ├── before-create.ts
            ├── before-read.ts
            ├── before-update.ts
            ├── before-delete.ts
            ├── after-read.ts
            ├── after-create.ts
            ├── after-delete.ts
            └── index.ts      → reexport de todos os controllers da entidade
```

## Responsabilidades

| Subpasta | Responsabilidade |
|---|---|
| `controllers/base/` | Contrato base (`BaseController`), tipos de resposta (`BaseControllerResponse`, `ErrorDetails`) |
| `controllers/actions/` | Um controller por `action` definida no `index.cds` |
| `controllers/functions/` | Um controller por `function` definida no `index.cds` |
| `controllers/entity-events/<entidade>/` | Um controller por hook de entidade (before/after de CREATE, READ, UPDATE, DELETE) |

## Regras de ouro

1. **Nenhum controller contém lógica de negócio.** Toda lógica pertence ao use case ou ao domínio.
2. **Nenhum controller chama `req.reject`, `req.reply` ou qualquer API CAP.** Esse trabalho fica em `main/routes`.
3. **Nenhum controller lança `throw`.** Erros são retornados como `BaseControllerResponse` com `status >= 400`.
4. **O método público de todo controller é `handle`.** Assinatura: `public async handle(...): Promise<BaseControllerResponse>`.
5. **A injeção do use case é sempre via construtor**, com `private readonly` e tipagem de interface (nunca a impl concreta).
6. **Contratos vivem em `domain/use-cases/`**, não em `presentation/`. Os controllers referenciam `XxxUseCase.Params` diretamente.

## Documentos desta seção

- [controllers/README.md](./controllers/README.md) — visão geral dos controllers
  - [controllers/base.md](./controllers/base.md) — `BaseController`, `BaseControllerResponse`, `ErrorDetails`
  - [controllers/actions.md](./controllers/actions.md) — controllers de actions CDS
  - [controllers/functions.md](./controllers/functions.md) — controllers de functions CDS
  - [controllers/entity-events.md](./controllers/entity-events.md) — controllers de hooks de entidade
