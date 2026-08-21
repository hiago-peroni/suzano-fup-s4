# Hydrators

`domain/hydrators/` declara os contratos dos **mutadores de `request`** consumidos pelos use cases de `entity-events`. Cada hydrator acopla-se a um único evento (`before-read`, `before-create`, `before-update`, `before-delete`) e tem uma responsabilidade estreita: mutar `request.query.SELECT.where` (filtros) ou `request.body`/`request.data` (payload de entrada) antes da execução do CDS.

A implementação fica em `infrastructure/hydrators/<entidade>/<evento>.ts` (ver [infrastructure-layer/hydrators/README.md](../../infrastructure-layer/hydrators/README.md)). O domain só descreve o contrato.

> **Grafia canônica:** `hydrators/` (com **y**) e método `hydrate()` (com **y**). Nunca `hidrators`/`hidrate` — esse é typo histórico de LE44/Suzano que deve ser refatorado nesses projetos.

## Estrutura

A organização é **plana** — uma pasta por entidade plural, um arquivo por evento. Não usar subpastas `custom-where/` ou `manipulate-body/` (anti-padrão Suzano): o nome do evento já comunica a natureza da mutação.

```
src/domain/hydrators/
├── user-preferences/
│   ├── before-read.ts             → BeforeReadUserPreferencesHydrator
│   └── before-create.ts           → BeforeCreateUserPreferencesHydrator
├── purchase-order-items/
│   ├── before-read.ts
│   └── before-create.ts
└── role-collections/
    ├── before-create.ts
    └── before-update.ts
```

| Evento típico | O que muta |
|---|---|
| `before-read.ts` | `request.query.SELECT.where` (filtros) — ex.: injetar `tenantId`, `userId`, restrições por permissão |
| `before-create.ts` | `request.data` — ex.: gerar `id`, normalizar campos, aplicar defaults, lowercase em emails |
| `before-update.ts` | `request.data` — ex.: aplicar valores derivados, normalizar shape |
| `before-delete.ts` | (raro) `request.params` — ex.: validar pré-condição mínima |

## Shape canônico — `BeforeRead*Hydrator`

```typescript
// src/domain/hydrators/user-preferences/before-read.ts
export namespace BeforeReadUserPreferencesHydrator {
    export type Params = {
        request: BeforeReadUserPreferencesHydrator.MutableRequest;
        userId: string;
    };
    export type Result = void;

    export type MutableRequest = {
        query: {
            SELECT: {
                where?: unknown[];
            };
        };
    };
}

export interface BeforeReadUserPreferencesHydrator {
    hydrate(params: BeforeReadUserPreferencesHydrator.Params): BeforeReadUserPreferencesHydrator.Result;
}
```

Pontos canônicos:

- **Método único `hydrate(params): Result`** — sempre. Não há outro método público.
- **`Result = void`** — hydrator muta `request` in-place; não retorna nada.
- **`MutableRequest` no namespace** — descreve **apenas** os campos do `request` que o hydrator vai tocar. Substitui qualquer cast `as unknown[]` espalhado na implementação.
- **`Params` carrega o `request` + dados auxiliares** — `userId`, `tenantId`, `now`, etc. Quem extrai esses campos do request é o use case (via `getUser`), não o hydrator.

## Shape canônico — `BeforeCreate*Hydrator`

Hydrator de `before-create` muta `request.data`. O `MutableRequest` documenta o subset do payload que vai ser tocado:

```typescript
// src/domain/hydrators/role-collections/before-create.ts
export namespace BeforeCreateRoleCollectionsHydrator {
    export type Params = {
        request: BeforeCreateRoleCollectionsHydrator.MutableRequest;
    };
    export type Result = void;

    export type MutableRequest = {
        data: {
            id?: string;
            emails?: string[];
            status?: 'PENDING' | 'ACTIVE' | 'ARCHIVED';
        };
    };
}

export interface BeforeCreateRoleCollectionsHydrator {
    hydrate(params: BeforeCreateRoleCollectionsHydrator.Params): BeforeCreateRoleCollectionsHydrator.Result;
}
```

> O contrato comunica claramente **o que será mutado** — `id`, `emails`, `status`. Implementação tem zero liberdade para tocar campos não declarados.

## Por que `MutableRequest` no namespace e não `@sap/cds.Request`

O `Request` do `@sap/cds` é genérico demais (`request.query` é `any`, `request.data` é `unknown`) e arrasta acoplamento de framework para o domain. O `MutableRequest` resolve dois problemas:

1. **Documenta o contrato** — quem lê o hydrator sabe exatamente quais campos serão tocados.
2. **Isola o framework** — o domain não importa `@sap/cds`. A implementação faz `params.request as unknown as XxxHydrator.MutableRequest` uma única vez no `hydrate`.

## Hydrator vs Hook (use case de entity-event)

| Conceito | Localização | Responsabilidade |
|---|---|---|
| **Hook (use case)** | `domain/use-cases/entity-events/<entidade>/<evento>.ts` | Orquestra: valida payload, lança erro de domínio, chama hydrator, chama repository |
| **Hydrator** | `domain/hydrators/<entidade>/<evento>.ts` | Muta `request` in-place; sem regra de negócio, sem I/O remoto |

O fluxo canônico: o controller chama o use case do entity-event; o use case orquestra (valida, busca dados auxiliares, chama o hydrator); o hydrator muta o `request` antes do CDS executar.

```typescript
// src/application/use-cases/entity-events/user-preferences/before-read.ts (trecho)
public async execute(params: BeforeReadUserPreferencesUseCase.Params): Promise<BeforeReadUserPreferencesUseCase.Result> {
    try {
        const user = getUser(params.request);
        this.hydrator.hydrate({ request: params.request, userId: user.id! });
        return right(undefined);
    } catch (error) {
        return left(this.handleError(error));
    }
}
```

## Hydrator não tem regra de negócio

O hydrator é **operacional**, não decisional. Quem decide se algo é válido, se uma transição é permitida, ou se um erro deve ser lançado é o **use case** ou o **model**. O hydrator apenas executa a mutação determinada.

```typescript
// ❌ ERRADO — hydrator decide regra de negócio
export interface BeforeCreateUserPreferencesHydrator {
    hydrate(params: BeforeCreateUserPreferencesHydrator.Params): BeforeCreateUserPreferencesHydrator.Result;
}

// Implementação:
public hydrate(params): void {
    if (!params.userId) {
        throw new BadRequestError('userId required');  // ← regra de negócio no hydrator
    }
    // ...
}
```

```typescript
// ✅ CORRETO — use case valida, hydrator muta
public async execute(params: BeforeCreateUseCase.Params): Promise<BeforeCreateUseCase.Result> {
    if (!params.userId) {
        return left(new BadRequestError('userId required'));  // ← regra no use case
    }
    this.hydrator.hydrate({ request: params.request, userId: params.userId });  // ← muta in-place
    return right(undefined);
}
```

## Hydrator não tem I/O

Hydrator **não consome** repository, adapter, parser, ou qualquer artefato com I/O remoto. Se o caso de uso precisa buscar dados externos para construir a mutação (ex.: olhar permissões antes de filtrar `where`), o use case faz a busca **antes** e passa o resultado pronto via `Params`.

```typescript
// ❌ ERRADO — hydrator chama repository
export interface BeforeReadOrdersHydrator {
    hydrate(params: BeforeReadOrdersHydrator.Params): Promise<void>;  // ← Promise = I/O
}
```

```typescript
// ✅ CORRETO — hydrator é síncrono e puro; use case busca e injeta dados
export interface BeforeReadOrdersHydrator {
    hydrate(params: BeforeReadOrdersHydrator.Params): void;  // ← Result = void síncrono
}

export namespace BeforeReadOrdersHydrator {
    export type Params = {
        request: BeforeReadOrdersHydrator.MutableRequest;
        userId: string;
        allowedTenants: string[];  // ← dados auxiliares já buscados pelo use case
    };
}
```

## Hydrator é stateless

Hydrators **não têm DI no caso comum** — o domain declara apenas o método `hydrate`. Quando excepcionalmente precisarem de um colaborador (outro hydrator, helper de formatação), a DI vai na infrastructure (`infrastructure/hydrators/`) — o contrato no domain segue só a interface.

## Imports permitidos

| O que pode importar | Exemplo |
|---|---|
| Tipos básicos | (nenhum import necessário no caso comum — `MutableRequest` é declarado no próprio namespace) |
| Models (raro — quando hydrator entrega struct para `request.data`) | `import type { PriceListProps } from '@/domain/models/db/price-list.js';` |

| O que **não** pode importar | Motivo |
|---|---|
| `@sap/cds`, `@models/*` | Acoplamento com framework — usar `MutableRequest` |
| `application/`, `infrastructure/`, `presentation/`, `main/` | Quebra de isolamento |
| Repositories, adapters | Hydrator não tem I/O |

## Naming

| Elemento | Convenção | Exemplo |
|---|---|---|
| Pasta de entidade | `kebab-case-plural` | `user-preferences/`, `purchase-order-items/`, `role-collections/` |
| Arquivo (evento) | `<evento>-<ação>.ts` (sempre nome do evento) | `before-read.ts`, `before-create.ts`, `before-update.ts` |
| Interface | `<Evento><Ação><EntidadePlural>Hydrator` | `BeforeReadUserPreferencesHydrator` |
| Namespace colateral | mesmo nome da interface | `namespace BeforeReadUserPreferencesHydrator { ... }` |
| Método público | `hydrate` (com **y**) | `public hydrate(params): void` |

## Regras

1. **Grafia `hydrators/` + `hydrate()`** — com **y**. Nunca `hidrators`/`hidrate`.
2. **Método único `hydrate(params): Result`** — síncrono no caso comum (`Result = void`).
3. **`Result = void`** — hydrator muta in-place; não retorna estrutura nova.
4. **`MutableRequest` declarado no namespace** — documenta exatamente quais campos serão tocados; substitui imports de `@sap/cds`.
5. **Sem regra de negócio no contrato** — validações ficam no use case ou no model.
6. **Sem I/O no contrato** — `hydrate` é síncrono; busca de dados auxiliares fica no use case.
7. **Estrutura plana** — `hydrators/<entidade-plural>/<evento>.ts`. Sem `custom-where/`/`manipulate-body/`.
8. **Um arquivo por evento** — não juntar `before-read` e `before-create` no mesmo arquivo.
9. **Sem `@sap/cds`** — `MutableRequest` é o wrapper canônico.

## Anti-padrões

### 1. Grafia `hidrators`/`hidrate` (com **i**)

```
❌  src/domain/hidrators/user-preferences/before-read.ts
❌  public hidrate(params): void { /* ... */ }

✅  src/domain/hydrators/user-preferences/before-read.ts
✅  public hydrate(params): void { /* ... */ }
```

Typo histórico em LE44 e Suzano. Renomeação entra como débito técnico nesses projetos.

### 2. Subdivisão por tipo de mutação

```
❌  src/domain/hydrators/user-preferences/custom-where/before-read.ts
❌  src/domain/hydrators/role-collections/manipulate-body/before-create.ts

✅  src/domain/hydrators/user-preferences/before-read.ts
✅  src/domain/hydrators/role-collections/before-create.ts
```

Padrão Suzano de subpastas `custom-where/`/`manipulate-body/` é anti-padrão. O nome do evento já comunica a natureza da mutação.

### 3. `Result` não é `void`

```typescript
// ❌ ERRADO — retorna struct nova
export namespace BeforeReadHydrator {
    export type Result = { customWhere: unknown[] };
}
```

```typescript
// ✅ CORRETO — muta request in-place
export namespace BeforeReadHydrator {
    export type Result = void;
}
```

> Hydrator que retorna struct é **service** ou **model**, não hydrator. A semântica de "muta request in-place" é o que distingue o hydrator dos demais artefatos.

### 4. `Promise<void>` em vez de `void`

```typescript
// ❌ ERRADO — Promise sinaliza I/O
export interface BeforeReadHydrator {
    hydrate(params: BeforeReadHydrator.Params): Promise<void>;
}
```

```typescript
// ✅ CORRETO — síncrono
export interface BeforeReadHydrator {
    hydrate(params: BeforeReadHydrator.Params): void;
}
```

> Se o hydrator precisa de async (chamada a outro service), repensar a arquitetura — o async deveria estar no use case que orquestra.

### 5. Importar `@sap/cds.Request`

```typescript
// ❌ ERRADO — Request do CAP no contrato
import type { Request } from '@sap/cds';

export namespace BeforeReadHydrator {
    export type Params = {
        request: Request;
        userId: string;
    };
}
```

```typescript
// ✅ CORRETO — MutableRequest no namespace
export namespace BeforeReadHydrator {
    export type Params = {
        request: BeforeReadHydrator.MutableRequest;
        userId: string;
    };
    export type MutableRequest = {
        query: { SELECT: { where?: unknown[] } };
    };
}
```

### 6. Cast inline `as string[]` na implementação

```typescript
// ❌ ERRADO — cast espalhado na infra (BeforeCreateRoleCollectionsHydratorImpl)
public hydrate(params): void {
    const emails = params.request.data.emails as string[];  // ← cast inline
    // ...
}
```

```typescript
// ✅ CORRETO — MutableRequest tipa o subset; cast acontece uma única vez
// domain/hydrators/role-collections/before-create.ts
export namespace BeforeCreateRoleCollectionsHydrator {
    export type MutableRequest = {
        data: {
            emails?: string[];  // ← já tipado
        };
    };
}

// infrastructure/hydrators/role-collections/before-create.ts
public hydrate(params): void {
    const request = params.request as unknown as BeforeCreateRoleCollectionsHydrator.MutableRequest;
    const { emails } = request.data;  // ← sem cast inline
    // ...
}
```

### 7. Hydrator com regra de negócio

```typescript
// ❌ ERRADO — hydrator valida e lança erro
public hydrate(params): void {
    if (params.userId.length < 3) {
        throw new BadRequestError('userId.tooShort');  // ← regra no hydrator
    }
    // ...
}
```

```typescript
// ✅ CORRETO — validação no use case; hydrator só muta
// use case:
if (params.userId.length < 3) {
    return left(new BadRequestError('userId.tooShort'));
}
this.hydrator.hydrate({ request: params.request, userId: params.userId });
```

### 8. Hydrator chamando repository ou adapter

```typescript
// ❌ ERRADO — hydrator faz I/O
export interface BeforeReadOrdersHydrator {
    hydrate(params: BeforeReadOrdersHydrator.Params): Promise<void>;
}

// Impl:
public async hydrate(params): Promise<void> {
    const tenants = await this.tenantRepo.findByUser(params.userId);  // ← I/O
    // ...
}
```

```typescript
// ✅ CORRETO — use case busca dados; hydrator recebe pronto
// use case:
const tenants = await this.tenantRepo.findByUser(params.userId);
this.hydrator.hydrate({ request: params.request, allowedTenants: tenants });
```

### 9. Sufixo `Hidrator` ou ausência de `Hydrator` na interface

```typescript
// ❌ ERRADO
export interface BeforeReadUserPreferences { /* ... */ }
export interface BeforeReadUserPreferencesHidrator { /* ... */ }
```

```typescript
// ✅ CORRETO
export interface BeforeReadUserPreferencesHydrator { /* ... */ }
```

### 10. Múltiplos hooks no mesmo arquivo

```typescript
// ❌ ERRADO — before-read e before-create misturados
// domain/hydrators/user-preferences/index.ts
export interface BeforeReadUserPreferencesHydrator { /* ... */ }
export interface BeforeCreateUserPreferencesHydrator { /* ... */ }
```

```typescript
// ✅ CORRETO — um arquivo por evento
// domain/hydrators/user-preferences/before-read.ts
export interface BeforeReadUserPreferencesHydrator { /* ... */ }

// domain/hydrators/user-preferences/before-create.ts
export interface BeforeCreateUserPreferencesHydrator { /* ... */ }
```
