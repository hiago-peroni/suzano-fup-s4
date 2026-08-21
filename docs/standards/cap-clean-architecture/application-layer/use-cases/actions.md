# use-cases/actions/

Actions são use cases de **comando** — operações explicitamente iniciadas pelo usuário que modificam estado ou executam lógica de negócio complexa. Correspondem 1:1 às `action` declaradas no `index.cds`. São chamadas pelos controllers da presentation layer e nunca disparadas automaticamente por hooks de entidade.

Exemplos de actions: `approvePriceList`, `saveTheme`, `massUpdateLossProvisions`, `uploadDataLoad`.

> **Consultas com parâmetros complexos também são actions.** Quando uma operação de leitura recebe objetos ou arrays como input (ex.: busca em lote de materiais), deve ser declarada como `action` no CDS — não como `function`. O binding HTTP de `function` é `GET`, que não suporta body com estrutura complexa. Ver `CatalogMultiSearchUseCase` no Portal MRO (`mro-application-service/src/domain/use-cases/actions/catalog-multi-search.ts`) como exemplo canônico.

---

## Shape canônico

```typescript
// src/application/use-cases/actions/save-theme.ts
import { left, right } from '@sweet-monads/either';

import { BaseUseCaseImpl } from '@/application/use-cases/base/base.js';
import { BadRequestError, NotFoundError } from '@/domain/errors/index.js';
import { TenantThemeModel } from '@/domain/models/tenant-theme.js';
import { TenantModel } from '@/domain/models/tenants.js';
import type { TenantThemeRepository } from '@/domain/repositories/tenant-themes.js';
import type { TenantRepository } from '@/domain/repositories/tenants.js';
import { SaveThemeUseCase } from '@/domain/use-cases/actions/save-theme.js';

export class SaveThemeUseCaseImpl extends BaseUseCaseImpl implements SaveThemeUseCase {
    constructor(
        private readonly themeRepo: TenantThemeRepository,
        private readonly tenantRepo: TenantRepository
    ) {
        super();
    }

    public async execute(params: SaveThemeUseCase.Params): Promise<SaveThemeUseCase.Result> {
        try {
            const { tenantId, ...themeFields } = params;
            const tenantValidation = TenantModel.validateId(tenantId);
            if (tenantValidation.hasError) {
                return left(new BadRequestError(tenantValidation.errorMessages!.join('; ')));
            }
            await this.validateTenantExists(tenantId);
            const model = this.buildAndValidateModel(themeFields);
            await this.themeRepo.save(tenantId, model);
            return right(undefined);
        } catch (error) {
            return left(this.handleError(error));
        }
    }

    private async validateTenantExists(tenantId: string): Promise<void> {
        const tenant = await this.tenantRepo.findById(tenantId);
        if (!tenant) {
            throw new NotFoundError('tenant.notFound');
        }
    }

    private buildAndValidateModel(themeFields: Omit<SaveThemeUseCase.Params, 'tenantId'>): TenantThemeModel {
        const model = TenantThemeModel.from(themeFields);
        const validation = model.validate();
        if (validation.hasError) {
            throw new BadRequestError(validation.errorMessages!.join('; '));
        }
        return model;
    }
}
```

---

## Variação: action com `before-execute`

Quando uma action possui uma etapa de pré-processamento separável — como validar permissões ou enriquecer o payload antes da execução principal — essa etapa pode ser extraída para um use case `BeforeExecuteXxxUseCaseImpl` ao lado do principal.

A presentation layer registra dois controllers distintos: um para `service.before('executeXxx')` e outro para `service.on('executeXxx')`. Cada controller chama seu respectivo use case.

Crie `before-execute` somente quando o ciclo de vida da action no CAP exige dois handlers separados. Não crie por convenção ou antecipação.

### Estrutura de pasta

```
use-cases/actions/loss-provisions/mass-update/
├── before-execute.ts    → BeforeExecuteMassUpdateUseCaseImpl
├── mass-update.ts       → MassUpdateUseCaseImpl
└── index.ts             → export * from './before-execute'; export * from './mass-update';
```

### Few-shot: `before-execute.ts`

```typescript
// src/application/use-cases/actions/loss-provisions/mass-update/before-execute.ts
import { left, right } from '@sweet-monads/either';

import { BaseUseCaseImpl } from '@/application/use-cases/base/base.js';
import { BadRequestError } from '@/domain/errors/index.js';
import type { UsersByUserGroupRepository } from '@/domain/repositories/users-by-user-group.js';
import type { PermissionCheckerService } from '@/domain/services/permission-checker.js';
import { BeforeExecuteMassUpdateUseCase } from '@/domain/use-cases/actions/loss-provisions/mass-update/before-execute.js';

export class BeforeExecuteMassUpdateUseCaseImpl extends BaseUseCaseImpl implements BeforeExecuteMassUpdateUseCase {
    constructor(
        private readonly usersByUserGroupRepository: UsersByUserGroupRepository,
        private readonly permissionChecker: PermissionCheckerService
    ) {
        super();
    }

    public async execute(params: BeforeExecuteMassUpdateUseCase.Params): Promise<BeforeExecuteMassUpdateUseCase.Result> {
        try {
            const userEmail = this.extractValidEmail(params.request);
            const groups = await this.usersByUserGroupRepository.findGroupsByEmail(userEmail);
            const hasPermission = await this.permissionChecker.checkPermission(groups, 'MassUpdate');
            if (!hasPermission) {
                return left(new BadRequestError('userDoesntHavePermission'));
            }
            return right(undefined);
        } catch (error) {
            return left(this.handleError(error));
        }
    }

    private extractValidEmail(request: BeforeExecuteMassUpdateUseCase.Params['request']): string {
        const email = request.user?.email;
        if (!email) {
            throw new BadRequestError('auth.userNotAuthenticated');
        }
        return email;
    }
}
```

### Few-shot: `mass-update.ts` (use case principal)

```typescript
// src/application/use-cases/actions/loss-provisions/mass-update/mass-update.ts
import { left, right } from '@sweet-monads/either';

import { BaseUseCaseImpl } from '@/application/use-cases/base/base.js';
import { BadRequestError } from '@/domain/errors/index.js';
import type { LossProvisionRepository } from '@/domain/repositories/loss-provision.js';
import { MassUpdateUseCase } from '@/domain/use-cases/actions/loss-provisions/mass-update/mass-update.js';

export class MassUpdateUseCaseImpl extends BaseUseCaseImpl implements MassUpdateUseCase {
    constructor(private readonly lossProvisionRepository: LossProvisionRepository) {
        super();
    }

    public async execute(params: MassUpdateUseCase.Params): Promise<MassUpdateUseCase.Result> {
        try {
            if (!params.items || params.items.length === 0) {
                return left(new BadRequestError('massUpdate.emptyItems'));
            }
            const updated = await this.lossProvisionRepository.massUpdate(params.items);
            return right(updated);
        } catch (error) {
            return left(this.handleError(error));
        }
    }
}
```

### Few-shot: `index.ts` barrel da pasta

```typescript
// src/application/use-cases/actions/loss-provisions/mass-update/index.ts
export { BeforeExecuteMassUpdateUseCaseImpl } from './before-execute.js';
export { MassUpdateUseCaseImpl } from './mass-update.js';
```

---

## Anti-padrão: autorização via segundo parâmetro `roles`

❌ **Nunca faça isso:**

```typescript
// ERRADO — segundo parâmetro viola o contrato do domínio
public async execute(params: XxxUseCase.Params, roles: string[]): Promise<XxxUseCase.Result> {
    if (!roles.includes('admin')) {
        return left(new BadRequestError('noPermission'));
    }
    // ...
}
```

O contrato `XxxUseCase` definido no domínio declara `execute(params): Promise<Result>`. Adicionar `roles` como segundo parâmetro viola a interface e expõe detalhes de autorização ao chamador.

✅ **Faça assim — injete o checker no constructor:**

```typescript
// CERTO — autorização via serviço injetado
export class ApproveOrderUseCaseImpl extends BaseUseCaseImpl implements ApproveOrderUseCase {
    constructor(
        private readonly orderRepository: OrderRepository,
        private readonly permissionChecker: PermissionCheckerService
    ) {
        super();
    }

    public async execute(params: ApproveOrderUseCase.Params): Promise<ApproveOrderUseCase.Result> {
        try {
            const hasPermission = await this.permissionChecker.checkPermission(
                params.userRoles,
                'ApproveOrder'
            );
            if (!hasPermission) {
                return left(new BadRequestError('noPermissionToApproveOrder'));
            }
            // ...
            return right(undefined);
        } catch (error) {
            return left(this.handleError(error));
        }
    }
}
```

---

## Estrutura de pastas (referência)

```
use-cases/actions/
├── approve-price-list.ts
├── checkout.ts
├── release-loss-provisions.ts
├── save-theme.ts
└── loss-provisions/
    └── mass-update/
        ├── before-execute.ts
        ├── mass-update.ts
        └── index.ts
```

---

## Naming

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo (action simples) | `kebab-case.ts` | `save-theme.ts` |
| Arquivo (before-execute) | `before-execute.ts` (nome fixo) | `before-execute.ts` |
| Pasta (action com before-execute) | `kebab-case` da action | `mass-update/` |
| Classe (action principal) | `PascalCase` + `UseCaseImpl` | `SaveThemeUseCaseImpl` |
| Classe (before-execute) | `BeforeExecute` + `PascalCase` + `UseCaseImpl` | `BeforeExecuteMassUpdateUseCaseImpl` |
| Contrato no domínio | `PascalCase` + `UseCase` | `SaveThemeUseCase` |
