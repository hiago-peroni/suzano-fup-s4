# Hydrators

Hydrators mutam o objeto `request` do CAP **antes** da execução do handler. Cada hydrator acopla-se a um único evento de entidade (`before-read`, `before-create`, `before-update`, `before-delete`) e é consumido pelo use case correspondente em `application/use-cases/entity-events/<entidade>/<evento>.ts`.

## O que é — e o que não é

| É | Não é |
|---|---|
| Mutador de `request.query.SELECT.where` ou `request.body` | Tradutor de DTO ↔ domínio (responsabilidade dos models) |
| Função pura no contrato (sem I/O remoto) | Wrapper de serviço técnico (isso é adapter) |
| Acoplado a um evento (`before-read`, etc.) | Reader de contexto runtime (isso é `utils/get-user.ts`) |
| Retorna `void` mutando in-place | Retorna estrutura nova (isso é um service) |

## Estrutura canônica

A estrutura é **plana** — uma pasta por entidade plural, um arquivo por evento. Não usamos subpastas `custom-where/` ou `manipulate-body/` (anti-padrão Suzano): a intenção fica clara pelo nome do próprio evento (`before-read` → muta where; `before-create` → muta body).

```
src/infrastructure/hydrators/
├── user-preferences/
│   ├── before-read.ts             → BeforeReadUserPreferencesHydratorImpl
│   └── before-create.ts           → BeforeCreateUserPreferencesHydratorImpl
├── purchase-order-items/
│   ├── before-read.ts
│   └── before-create.ts
└── role-collections/
    └── before-create.ts
```

> **Não usar** subpastas `custom-where/` ou `manipulate-body/`. A estrutura plana com nome de evento é suficiente e mais legível.

## Naming

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo | `kebab-case.ts` (nome do evento) | `before-read.ts`, `before-create.ts` |
| Classe | `Before/After<Evento><EntidadePlural>HydratorImpl` | `BeforeReadUserPreferencesHydratorImpl` |
| Interface (no domínio) | `Before/After<Evento><EntidadePlural>Hydrator` | `BeforeReadUserPreferencesHydrator` |
| Pasta de entidade | `kebab-case-plural` | `user-preferences/`, `purchase-order-items/` |
| Método público | `hydrate` (não `hidrate`) | `public hydrate(params): void` |

> **Grafia canônica:** `hydrators` (com "y") e método `hydrate`. Nunca `hidrators`/`hidrate` — typo histórico de projetos LE44/Suzano.

## Contrato no domínio

A interface vive em `domain/hydrators/<entidade>/<evento>.ts`. O namespace agrupa **todos** os tipos do contrato — incluindo o shape do `Request` CAP que será mutado e dos campos esperados no `data`:

```typescript
// src/domain/hydrators/user-preferences/before-read.ts
import type { Request } from '@sap/cds';

export namespace BeforeReadUserPreferencesHydrator {
    export type Params = {
        request: Request;
        userId: string;
    };
    export type Result = void;
    export type MutableRequest = {
        query: { SELECT: { where?: unknown[] } };
    };
}

export interface BeforeReadUserPreferencesHydrator {
    hydrate(params: BeforeReadUserPreferencesHydrator.Params): BeforeReadUserPreferencesHydrator.Result;
}
```

```typescript
// src/domain/hydrators/role-collections/before-create.ts
import type { Request } from '@sap/cds';

export namespace BeforeCreateRoleCollectionsHydrator {
    export type Params = {
        request: Request;
    };
    export type Result = void;
    export type MutableRequest = {
        data: {
            emails?: string[];
            status?: string;
        };
    };
}

export interface BeforeCreateRoleCollectionsHydrator {
    hydrate(params: BeforeCreateRoleCollectionsHydrator.Params): BeforeCreateRoleCollectionsHydrator.Result;
}
```

## Few-shot 1 — `before-read` mutando `SELECT.where`

```typescript
// src/infrastructure/hydrators/user-preferences/before-read.ts
import type { BeforeReadUserPreferencesHydrator } from '@/domain/hydrators/user-preferences/before-read.js';

export class BeforeReadUserPreferencesHydratorImpl implements BeforeReadUserPreferencesHydrator {
    private readonly USER_ID_FIELD = 'userId';
    private readonly AND_OPERATOR = 'and';

    public hydrate(params: BeforeReadUserPreferencesHydrator.Params): BeforeReadUserPreferencesHydrator.Result {
        const { request, userId } = params;
        const customWhere = this.buildCustomWhereByUserId(userId);
        this.appendCustomWhere(request as unknown as BeforeReadUserPreferencesHydrator.MutableRequest, customWhere);
    }

    private appendCustomWhere(
        request: BeforeReadUserPreferencesHydrator.MutableRequest,
        customWhere: unknown[]
    ): void {
        const { where } = request.query.SELECT;
        if (where && where.length > 0) {
            request.query.SELECT.where = [...where, this.AND_OPERATOR, ...customWhere];
        } else {
            request.query.SELECT.where = customWhere;
        }
    }

    private buildCustomWhereByUserId(userId: string): unknown[] {
        return [{ ref: [this.USER_ID_FIELD] }, '=', { val: userId }];
    }
}
```

## Few-shot 2 — `before-create` mutando `request.data`

```typescript
// src/infrastructure/hydrators/role-collections/before-create.ts
import type { BeforeCreateRoleCollectionsHydrator } from '@/domain/hydrators/role-collections/before-create.js';

export class BeforeCreateRoleCollectionsHydratorImpl implements BeforeCreateRoleCollectionsHydrator {
    private readonly DEFAULT_STATUS = 'PENDING';

    public hydrate(params: BeforeCreateRoleCollectionsHydrator.Params): BeforeCreateRoleCollectionsHydrator.Result {
        const request = params.request as unknown as BeforeCreateRoleCollectionsHydrator.MutableRequest;
        this.normalizeEmails(request);
        this.fillDefaultStatus(request);
    }

    private normalizeEmails(request: BeforeCreateRoleCollectionsHydrator.MutableRequest): void {
        const { emails } = request.data;
        if (!emails) {
            return;
        }
        request.data.emails = emails.map((email) => email.trim().toLowerCase());
    }

    private fillDefaultStatus(request: BeforeCreateRoleCollectionsHydrator.MutableRequest): void {
        if (!request.data.status) {
            request.data.status = this.DEFAULT_STATUS;
        }
    }
}
```

O cast `params.request as unknown as XxxHydrator.MutableRequest` é necessário porque o `Request` do `@sap/cds` é genérico demais para acessar os campos específicos que o hydrator vai mutar. O tipo `MutableRequest` no domain documenta exatamente quais campos serão tocados.

## Como o use case orquestra o hydrator

O use case recebe o hydrator via DI no constructor e chama `hydrate` dentro do `try/catch` — erros propagam normalmente para o `handleError`:

```typescript
// src/application/use-cases/entity-events/user-preferences/before-read.ts (trecho)
public async execute(params: BeforeReadUserPreferencesUseCase.Params): Promise<BeforeReadUserPreferencesUseCase.Result> {
    try {
        const userId = getUser(params.request).id;
        this.hydrator.hydrate({ request: params.request, userId });
        return right(undefined);
    } catch (error) {
        return left(this.handleError(error));
    }
}
```

> O hydrator não tem DI no caso simples — é stateless. O use case é quem recebe o hydrator via DI no constructor.

## Tipagem e constantes

Aplicar as 3 regras globais da [seção "Tipagem e constantes" do README da infrastructure-layer](../README.md#tipagem-e-constantes). Especificamente para hydrators:

- **`Params`, `Result`, `MutableRequest`** vivem no namespace da interface em `domain/hydrators/<entidade>/<evento>.ts`. O `MutableRequest` documenta os campos específicos do `request` CAP que o hydrator vai mutar — substitui qualquer cast `as string[]` espalhado no método.
- **Strings que parametrizam a mutação** (nomes de campos, operadores SQL, valores default) ficam como `private readonly` da classe — não em module-level e não como string literal espalhada nos métodos privados.
- **Zero `interface`/`type` declarado dentro de `infrastructure/hydrators/`**. Mesmo para tipos auxiliares como o shape do `request.data`.

## Regras

1. **Mutar in-place, retornar `void`.** Não retornar estrutura nova — o CAP segue usando o mesmo objeto `request`.
2. **Sem DI por padrão.** O hydrator é stateless. Aceitar DI apenas quando precisa de outro hydrator ou utilitário (raro).
3. **Sem `try/catch` defensivo.** Erros são raros (validação de payload é responsabilidade do use case). Se algo falhar, propagar para o use case.
4. **Sem subpastas `custom-where/`/`manipulate-body/`.** Estrutura plana: `hydrators/<entidade-plural>/<evento>.ts`. A intenção é óbvia pelo nome do evento.
5. **Grafia `hydrators` / método `hydrate`** — nunca `hidrators`/`hidrate` (typo histórico).
6. **Um arquivo por evento.** Não juntar `before-read` e `before-create` no mesmo arquivo.
7. **Métodos privados extraem complexidade.** O método público `hydrate` apenas orquestra; lógica fica em privados (`buildCustomWhereByUserId`, `normalizeEmails`).
8. **`MutableRequest` no domain documenta o shape mutado.** Em vez de `request.data.emails as string[]` (cast inline), declare o tipo no namespace do hydrator e converta `request` uma única vez no método público.
9. **Constantes auxiliares (`USER_ID_FIELD`, `DEFAULT_STATUS`, `AND_OPERATOR`) como `private readonly`** da classe — não como string literal espalhada nem como `const` em module-level.

## Anti-padrões

1. **Subdivisão por tipo de mutação** (`custom-where/`, `manipulate-body/`) — padrão Suzano. O nome do evento já comunica a intenção; subpastas diluem sem agregar.
2. **Grafia `hidrators`/`hidrate`** — typo histórico de LE44/Suzano.
3. **Retornar struct em vez de mutar in-place** — confunde quem lê o use case (parece um service).
4. **Hydrator chamando repositório/adapter** — se precisa de I/O remoto, é um service de aplicação, não um hydrator.
5. **Sufixo `Hidrator` ou ausência de `Impl`** — sempre `HydratorImpl` na implementação; a interface no domínio não leva `Impl`.
6. **`type`/`interface` declarado dentro do arquivo do hydrator** — todo tipo vem do namespace da interface no domain.
7. **String literal repetida em métodos privados** (`'and'`, `'PENDING'`, `'userId'`) — promover para `private readonly` da classe.
8. **Cast inline `request.data.foo as Xxx`** — declarar `MutableRequest` no domain e converter `params.request` uma única vez no método público.
