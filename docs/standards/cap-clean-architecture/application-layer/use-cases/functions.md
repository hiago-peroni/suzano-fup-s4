# use-cases/functions/

Functions são use cases de **consulta** — operações que retornam dados sem causar side-effects de persistência. Correspondem 1:1 às `function` declaradas no `index.cds`. São chamadas pelos controllers da presentation layer.

Exemplos de functions: `findLossProvisionPeriod`, `getOrderSummary`, `listTenantThemes`.

---

## Distinção semântica em relação às actions

| Critério | Action | Function |
|---|---|---|
| Intenção | Comando — modifica estado | Consulta — lê e retorna dados |
| Side-effects de persistência | Permitidos | Não permitidos (`INSERT`, `UPDATE`, `DELETE`) |
| Complexidade dos parâmetros | Objetos ou arrays como input | Apenas escalares (`string`, `number`, `boolean`) |
| Binding HTTP no CAP | `POST` | `GET` |
| Declaração no CDS | `action <nome>(...)` | `function <nome>(...)` |
| Estrutura de código | Idêntica | Idêntica |

A distinção é **semântica e de nomenclatura**, não estrutural. O shape do código TypeScript é o mesmo. O que diferencia é a intenção e a complexidade dos parâmetros.

> **Regra de ouro:** se a consulta recebe objetos ou arrays como input (ex.: lista de materiais para busca em lote), ela deve ser declarada como `action` no CDS — mesmo que seja read-only. O HTTP binding do CAP para `function` é `GET`, que não suporta body com estrutura complexa.

### Exemplo do Portal MRO

`CatalogSearchUseCase` permanece **function** porque os parâmetros são todos escalares:

```typescript
// src/domain/use-cases/functions/catalog-search-use-case.ts
export namespace CatalogSearchUseCase {
    export type Params = {
        tenantId: string;
        language: string;
        search: string;           // ← escalar
        searchType?: CatalogSearchType;  // ← escalar
        plantId: string;          // ← escalar
        top: number;              // ← escalar
        skip: number;             // ← escalar
        // ... demais filtros escalares
    };
}
```

`CatalogMultiSearchUseCase` é **action** porque recebe `items: SearchItem[]` — um array de objetos:

```typescript
// src/domain/use-cases/actions/catalog-multi-search.ts
export namespace CatalogMultiSearchUseCase {
    export type SearchItem = {
        material: string;
        contractId: string;
        supplierId: string;
    };

    export type Params = {
        tenantId: string;
        plantId: string;
        items: SearchItem[];   // ← array de objetos → obriga action
        top: number;
        skip: number;
        // ...
    };
}
```

---

## Shape canônico

```typescript
// src/application/use-cases/functions/find-loss-provision-period.ts
import { left, right } from '@sweet-monads/either';

import { BaseUseCaseImpl } from '@/application/use-cases/base/base.js';
import type { LossProvisionPeriodRepository } from '@/domain/repositories/loss-provision-period.js';
import { FindLossProvisionPeriodUseCase } from '@/domain/use-cases/functions/find-loss-provision-period.js';

export class FindLossProvisionPeriodUseCaseImpl extends BaseUseCaseImpl implements FindLossProvisionPeriodUseCase {
    constructor(private readonly lossProvisionPeriodRepository: LossProvisionPeriodRepository) {
        super();
    }

    public async execute(params: FindLossProvisionPeriodUseCase.Params): Promise<FindLossProvisionPeriodUseCase.Result> {
        try {
            const locale = params.request.locale;
            const lossProvisionPeriod = await this.getLossProvisionPeriod(locale);
            return right(lossProvisionPeriod?.toObject() ?? null);
        } catch (error) {
            return left(this.handleError(error));
        }
    }

    private async getLossProvisionPeriod(locale: string) {
        const periods = await this.lossProvisionPeriodRepository.findAll();
        if (!periods || periods.length === 0) {
            return null;
        }
        return periods.find((p) => p.locale === locale) ?? periods[periods.length - 1];
    }
}
```

---

## Regras

1. **Nenhum side-effect de persistência.** Uma function nunca chama `repository.save()`, `repository.update()`, `repository.delete()` ou equivalentes.
2. **Métodos privados para lógica de seleção ou transformação** — extraia do `execute` para manter o método público legível.
3. **Resultado pode ser `null` ou array vazio** — prefira retornar `right(null)` ou `right([])` em vez de um erro quando "não encontrado" é um resultado válido do domínio.
4. **O contrato vive em `domain/use-cases/functions/<nome>.ts`** — nunca declare `Params` ou `Result` dentro de `application/`.

---

## Naming

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo | `kebab-case.ts` | `find-loss-provision-period.ts` |
| Classe | `PascalCase` + `UseCaseImpl` | `FindLossProvisionPeriodUseCaseImpl` |
| Contrato no domínio | `PascalCase` + `UseCase` | `FindLossProvisionPeriodUseCase` |
| Pasta | `use-cases/functions/` (plana) | — |
