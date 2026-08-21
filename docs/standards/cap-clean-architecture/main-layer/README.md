# Main layer

A main layer é a camada que se "sacrifica" na arquitetura limpa. É a única que conhece todas as outras — e justamente por isso nenhuma outra camada pode depender dela.

Aqui vive a **composição**: factories que montam o grafo de dependências (DI manual), as rotas que conectam o framework ao domínio, e os scripts de desenvolvimento local.

## Estrutura

```
src/main/
├── assets/        → arquivos estáticos servidos em dev (opcional)
├── config/        → aliases de path e variáveis de ambiente
├── factories/     → DI manual: instancia e compõe todas as camadas
│   ├── adapters/
│   ├── controllers/
│   │   ├── actions/
│   │   ├── entity-events/
│   │   └── functions/
│   ├── repositories/
│   ├── services/
│   ├── use-cases/
│   │   ├── actions/
│   │   ├── entity-events/
│   │   └── functions/
│   └── utils/
├── routes/        → definição OData (`.cds`) + handlers TypeScript
└── scripts/       → utilitários de dev (populate, patch CSN)
```

## Responsabilidades

| Subpasta | Responsabilidade |
|---|---|
| `assets/` | Servir arquivos estáticos em dev (Excel templates, etc.) |
| `config/` | Registrar aliases de path (`@/`) e expor config de ambiente |
| `factories/` | Instanciar e compor todas as implementações concretas |
| `routes/` | Definir o serviço OData e registrar handlers no CAP |
| `scripts/` | Deploy SQLite local, patches de build — nunca importados pela app |

## Regra de ouro

> **Nenhum arquivo da `main/` contém lógica de negócio.** Toda lógica pertence ao domínio ou à aplicação. A main layer apenas instancia, injeta e conecta.

## Documentos desta seção

- [assets.md](./assets.md) — quando e como usar `assets/`
- [config.md](./config.md) — `module-alias.ts` e `environment.ts`
- [factories/README.md](./factories/README.md) — visão geral das factories
  - [factories/adapters.md](./factories/adapters.md)
  - [factories/controllers.md](./factories/controllers.md)
  - [factories/repositories.md](./factories/repositories.md)
  - [factories/services.md](./factories/services.md)
  - [factories/use-cases.md](./factories/use-cases.md)
  - [factories/utils.md](./factories/utils.md)
- [routes.md](./routes.md) — `index.cds` + `index.ts`
- [scripts.md](./scripts.md) — `populate-local-db.ts` + `replace-csn-source.ts`
