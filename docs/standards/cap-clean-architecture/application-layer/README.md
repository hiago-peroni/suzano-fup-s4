# Application layer

A application layer orquestra os casos de uso da aplicação. Ela recebe chamadas tipadas vindas da presentation layer, executa a lógica de negócio delegando a modelos de domínio e repositórios, e devolve resultados encapsulados em `Either<AbstractError, T>`. Nenhum detalhe de infraestrutura (banco de dados, HTTP, CAP framework) vive aqui — a camada só conhece interfaces de domínio.

> **Regra de isolamento:** a application layer não importa nada de `main/`, `presentation/` ou `infrastructure/`. Importa apenas de `domain/` e de outras partes de `application/` (ex.: services chamados por use cases).

## Estrutura

```
src/application/
├── use-cases/
│   ├── base/
│   │   └── base.ts               → BaseUseCaseImpl abstrata
│   ├── actions/
│   │   └── <kebab-name>.ts       → um use case por action CDS
│   ├── functions/
│   │   └── <kebab-name>.ts       → um use case por function CDS
│   ├── entity-events/
│   │   └── <entidade-plural>/
│   │       ├── before-create.ts
│   │       ├── before-read.ts
│   │       ├── before-update.ts
│   │       ├── before-delete.ts
│   │       ├── after-read.ts
│   │       ├── after-create.ts
│   │       ├── after-update.ts
│   │       └── index.ts          → reexport de todos os use cases da entidade
└── services/
    ├── base/
    │   └── base.ts               → BaseServiceImpl abstrata
    └── <contexto>/
        └── <kebab-name>.ts       → um service por passo de pipeline ou operação auxiliar
```

## Responsabilidades

| Subpasta | Responsabilidade |
|---|---|
| `use-cases/base/` | Classe base abstrata (`BaseUseCaseImpl`) com `handleError` compartilhado |
| `use-cases/actions/` | Use cases de comando — um por `action` declarada no `index.cds` |
| `use-cases/functions/` | Use cases de consulta — um por `function` declarada no `index.cds` |
| `use-cases/entity-events/<entidade>/` | Use cases de hook de entidade (before/after de CREATE, READ, UPDATE, DELETE) |
| `services/base/` | Classe base abstrata (`BaseServiceImpl`) — idêntica à `BaseUseCaseImpl`, nome diferente |
| `services/<contexto>/` | Services auxiliares chamados por outros use cases; passos atômicos de pipelines |

## Regras de ouro

1. **Nenhum use case ou service acessa infraestrutura diretamente.** Banco de dados, CDS queries, sistemas externos — tudo via interfaces de repositório ou service injetadas no constructor.
2. **`execute` é o único método público.** Assinatura: `public async execute(params: XxxUseCase.Params): Promise<XxxUseCase.Result>`.
3. **`Result` é sempre `Either<AbstractError, T>`.** Caminho feliz retorna `right(valor)`; erros retornam `left(new XxxError(...))`. Nunca lança `throw` para fora do `execute`.
4. **Todo use case e service herda a classe base.** `BaseUseCaseImpl` para use cases; `BaseServiceImpl` para services. O `super()` no constructor é obrigatório.
5. **DI exclusivamente via constructor** com `private readonly` tipado pela interface de domínio — nunca pela implementação concreta.
6. **Constantes de classe são permitidas.** `private readonly NOME_CONSTANTE = valor` dentro do corpo da classe é aceitável para valores fixos de configuração local (ex.: nome de arquivo, limite de tamanho, content type). Não criar propriedades mutáveis nem campos que não sejam `readonly`.
7. **Sem tipos ou interfaces soltos no arquivo.** Definições de `type`, `interface` ou `enum` não vivem em arquivos da `application/` — pertencem a `domain/use-cases/`, `domain/models/` ou `domain/errors/`.
8. **Contratos (`Params`/`Result`) vivem em `domain/`**, não em `application/`. Use `XxxUseCase.Params` importado de `@/domain/use-cases/<tipo>/<nome>.js`.
9. **Imports NodeNext exigem extensão `.js`** explícita: `import { BaseUseCaseImpl } from '@/application/use-cases/base/base.js'`.
10. **Services são chamados por use cases**, não por controllers. Se um controller chama diretamente, o candidato é um use case, não um service.
11. **`throw` é permitido em métodos privados** — contanto que seja capturado pelo `catch` do `execute` via `handleError`.

## Naming

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo | `kebab-case.ts` | `save-theme.ts`, `before-create.ts` |
| Classe use case | `PascalCase` + `UseCaseImpl` | `SaveThemeUseCaseImpl` |
| Classe service | `PascalCase` + `ServiceImpl` | `FinalizeDataLoadServiceImpl` |
| Pasta de entidade em entity-events | `kebab-case-plural` | `price-lists/`, `user-preferences/` |

## Documentos desta seção

- [use-cases/base.md](./use-cases/base.md) — `BaseUseCaseImpl` e `BaseServiceImpl`
- [use-cases/actions.md](./use-cases/actions.md) — use cases de actions CDS
- [use-cases/functions.md](./use-cases/functions.md) — use cases de functions CDS
- [use-cases/entity-events.md](./use-cases/entity-events.md) — use cases de hooks de entidade
- [services.md](./services.md) — services auxiliares e pipelines
