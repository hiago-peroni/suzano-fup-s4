# controllers/

Todo controller da presentation layer herda de `BaseController` (definido em `controllers/base/controller.ts`) e expõe um único método público: `handle`.

## Shape universal

```typescript
export class XxxController extends BaseController {
    constructor(private readonly useCase: XxxUseCase) {
        super();
    }

    public async handle(params: XxxUseCase.Params): Promise<BaseControllerResponse> {
        const result = await this.useCase.execute(params);
        if (result.isLeft()) {
            return this.error(result.value.code, result.value.toErrorDetails());
        }
        return this.success(result.value);
    }
}
```

## Regras

- Um arquivo por controller; nome em `kebab-case`.
- Classe em `PascalCase` com sufixo `Controller` (ex.: `ApprovePriceListController`).
- `private readonly useCase` com tipo de **interface** do domain — nunca a impl concreta.
- `handle` é sempre `public async` e retorna `Promise<BaseControllerResponse>`.
- Mapeamento de `Either`: `result.isLeft()` → `this.error(...)`; caso contrário → `this.success(...)`.
- **Sem `throw`.** Sem chamadas CAP. Sem acesso a `fs` ou dependências de infra.

## Subtipos de controller

| Subpasta | Tipo de operação CDS | Registro em `routes/index.ts` |
|---|---|---|
| `actions/` | `action MyAction(...)` | `service.on('myAction', ...)` |
| `functions/` | `function MyFn(...) returns ...` | `service.on('myFn', ...)` |
| `entity-events/<entidade>/` | Hooks before/after de entidade | `service.before/after('CREATE'\|'READ'\|'UPDATE'\|'DELETE', 'Entity', ...)` |

## Documentos desta seção

- [base.md](./base.md) — `BaseController`, `BaseControllerResponse`, `ErrorDetails`
- [actions.md](./actions.md) — controllers de actions
- [functions.md](./functions.md) — controllers de functions
- [entity-events.md](./entity-events.md) — controllers de hooks de entidade
