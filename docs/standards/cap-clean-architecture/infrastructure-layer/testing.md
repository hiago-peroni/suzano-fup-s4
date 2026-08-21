# Testing — Infrastructure Layer

Os testes da infrastructure layer focam em **comportamento observável**: queries CDS, mapeamentos raw → model, mutações de `request`. Não usam o framework CAP real — apenas stubam o `cds.run` quando necessário.

## Estrutura de pastas

A estrutura de `test/unit/infrastructure/` espelha `src/infrastructure/`:

```
test/unit/infrastructure/
├── repositories/
│   ├── models/
│   │   ├── price-lists/
│   │   │   ├── shared-helpers.ts      → makeSut + factories de fixtures
│   │   │   ├── scenarios-overview.md  → OBRIGATÓRIO
│   │   │   └── suites.test.ts         → describe/it cobrindo cenários listados
│   │   └── scenarios-overview.md      → overview da pasta repositories/models
│   └── replication/
│       ├── plants/
│       │   ├── shared-helpers.ts
│       │   ├── scenarios-overview.md
│       │   └── suites.test.ts
│       └── scenarios-overview.md      → overview da pasta repositories/replication
├── adapters/
│   ├── external-api/
│   │   └── company/
│   │       ├── shared-helpers.ts
│   │       ├── scenarios-overview.md
│   │       └── suites.test.ts
│   ├── unit-of-work/
│   │   ├── shared-helpers.ts
│   │   ├── scenarios-overview.md
│   │   └── suites.test.ts
│   └── scenarios-overview.md
├── hydrators/
│   └── user-preferences/
│       └── before-read/
│           ├── shared-helpers.ts
│           ├── scenarios-overview.md
│           └── suites.test.ts
├── utils/
│   └── translator/
│       ├── shared-helpers.ts
│       ├── scenarios-overview.md
│       └── suites.test.ts
└── shared/
    └── stubs/
        ├── base-stub.ts                → BaseStub (suporte a throw controlado)
        └── cds-stub.ts                 → CdsStub (mocka cds.run, cds.ql, cds.connect.to)
```

## `scenarios-overview.md` obrigatório

Toda pasta de teste que agrupa uma unidade (tool, repositório, adapter, hydrator) deve conter um `scenarios-overview.md` mapeando todos os cenários cobertos. Regra global do `AGENTS.md` raiz §5.2 — atualizar no mesmo commit dos testes.

### Few-shot de `scenarios-overview.md`

```markdown
# Scenarios — infrastructure/adapters/unit-of-work

| Arquivo | Cenário | Resultado esperado |
|---|---|---|
| suites.test.ts | `registerInsert` com lista vazia | Nenhum intent enfileirado |
| suites.test.ts | `registerInsert` com 1+ linhas | `pendingCount` aumenta corretamente |
| suites.test.ts | `commit` em dev (`NODE_ENV=development`) | Executa intents sequencialmente sem `cds.tx` |
| suites.test.ts | `commit` em prod | Wrappa intents em `cds.tx` |
| suites.test.ts | `commit` com lista vazia | No-op (sem chamar `cds.run`) |
| suites.test.ts | `commit` propaga exceção | Limpa intents internos antes do throw |
| suites.test.ts | `clear()` | Reset de intents para zero |
```

## Padrão `makeSut`

Cada pasta de teste tem `shared-helpers.ts` que exporta `makeSut()` retornando `{ sut, ...stubs }`. Nunca instanciar o SUT inline nos `it` — isso impede re-uso e dificulta refatoração de DI.

### `makeSut` sem DI

```typescript
// test/unit/infrastructure/adapters/external-api/company/shared-helpers.ts
import type { CompanyApi } from '@/domain/adapters/external-api/company.js';
import { CompanyApiImpl } from '@/infrastructure/adapters/external-api/company/company-api.js';

export type SutTypes = {
    sut: CompanyApi;
};

export const makeSut = (): SutTypes => {
    const sut = new CompanyApiImpl();
    return { sut };
};

export const makeValidParams = (overrides: Partial<CompanyApi.FindByCostCentersParams> = {}): CompanyApi.FindByCostCentersParams => ({
    costCenters: ['CC001', 'CC002'],
    ...overrides
});
```

### `makeSut` com DI (UoW + Telemetry)

```typescript
// test/unit/infrastructure/adapters/unit-of-work/shared-helpers.ts
import type { UnitOfWork } from '@/domain/adapters/unit-of-work.js';
import { CdsUnitOfWork } from '@/infrastructure/adapters/unit-of-work/cds-unit-of-work.js';

import { TelemetryStub } from '@tests/unit/shared/stubs/telemetry-stub.js';

export type SutTypes = {
    sut: UnitOfWork;
    telemetry: TelemetryStub;
};

export const makeSut = (): SutTypes => {
    const telemetry = new TelemetryStub();
    const sut = new CdsUnitOfWork(telemetry);
    return { sut, telemetry };
};
```

## `BaseStub` — suporte a throw controlado

Stubs que precisam simular erros herdam `BaseStub`. O método `setShouldThrow` aceita `true` (lança em qualquer método) ou `string` (lança apenas no método nomeado):

```typescript
// test/unit/shared/stubs/base-stub.ts
export class BaseStub {
    private throwTarget: string | boolean = false;

    public setShouldThrow(value: string | boolean): void {
        this.throwTarget = value;
    }

    protected checkThrow(method: string): void {
        if (this.throwTarget === true || this.throwTarget === method) {
            throw new Error(`Stub error on ${method}`);
        }
    }
}
```

### Few-shot de stub herdando `BaseStub`

```typescript
// test/unit/shared/stubs/telemetry-stub.ts
import type { Telemetry } from '@/domain/adapters/telemetry.js';

import { BaseStub } from '@tests/unit/shared/stubs/base-stub.js';

export class TelemetryStub extends BaseStub implements Telemetry {
    public readonly events: { name: string; payload?: unknown }[] = [];

    public event(name: string, payload?: unknown): void {
        this.checkThrow('event');
        this.events.push({ name, payload });
    }
}
```

Stubs simples (sem necessidade de injeção de erro) podem implementar a interface diretamente, sem herdar `BaseStub`.

## Mocking de `cds.run` em testes de repositório

Usar `vi.spyOn(cds, 'run')` — nunca mockar o módulo `@sap/cds` inteiro (quebra outras importações). Restaurar no `beforeEach` com `vi.restoreAllMocks()`:

```typescript
// test/unit/infrastructure/repositories/models/price-lists/suites.test.ts (trecho)
import { describe, it, expect, vi, beforeEach } from 'vitest';
import cds from '@sap/cds';

import { makeSut } from './shared-helpers.js';

describe('PriceListRepositoryImpl', () => {
    beforeEach(() => {
        vi.restoreAllMocks();
    });

    it('findById retorna null quando linha não existe', async () => {
        const { sut } = makeSut();
        vi.spyOn(cds, 'run').mockResolvedValueOnce([]);
        const result = await sut.findById('T0001', 'PL-NOT-EXIST');
        expect(result).toBeNull();
    });

    it('findById retorna PriceListModel quando linha existe', async () => {
        const { sut } = makeSut();
        vi.spyOn(cds, 'run').mockResolvedValueOnce([{ id: 'PL-1', tenant_id: 'T0001', name: 'Lista A' }]);
        const result = await sut.findById('T0001', 'PL-1');
        expect(result?.id).toBe('PL-1');
    });

    it('findById propaga exceção quando cds.run lança', async () => {
        const { sut } = makeSut();
        vi.spyOn(cds, 'run').mockRejectedValueOnce(new Error('connection refused'));
        await expect(sut.findById('T0001', 'PL-1')).rejects.toThrow('connection refused');
    });
});
```

## Teste de hydrator

Foco no estado do `request` antes/depois da chamada `hydrate`. Não há I/O — fixtures são objetos plain:

```typescript
// test/unit/infrastructure/hydrators/user-preferences/before-read/suites.test.ts
import { describe, it, expect } from 'vitest';

import { makeSut, makeRequestWithWhere } from './shared-helpers.js';

describe('BeforeReadUserPreferencesHydratorImpl', () => {
    it('adiciona where com userId quando request sem where prévio', () => {
        const { sut } = makeSut();
        const request = makeRequestWithWhere(undefined);
        sut.hydrate({ request, userId: 'U-123' });
        expect(request.query.SELECT.where).toEqual([{ ref: ['userId'] }, '=', { val: 'U-123' }]);
    });

    it('faz AND com where prévio', () => {
        const { sut } = makeSut();
        const request = makeRequestWithWhere([{ ref: ['status'] }, '=', { val: 'ACTIVE' }]);
        sut.hydrate({ request, userId: 'U-123' });
        expect(request.query.SELECT.where).toEqual([
            { ref: ['status'] }, '=', { val: 'ACTIVE' },
            'and',
            { ref: ['userId'] }, '=', { val: 'U-123' }
        ]);
    });
});
```

## Cobertura recomendada

| Tipo de unit | Cobertura mínima | Justificativa |
|---|---|---|
| Repositórios stateless (1 query) | Opcional | Risco baixo — queries simples |
| Repositórios com filtros dinâmicos | 80%+ | Erros silenciosos em construção de WHERE |
| Adapters externos (`*ApiImpl`) | 80%+ | Mapeamento raw → `XxxModel.with(...)` + tratamento de erro são críticos |
| Adapters Fake (`Fake*ApiImpl`) | Não testar | É a própria fixture do teste |
| Hydrators | 80%+ | Mutação in-place é fonte de bug sutil |
| UoW | 90%+ | Coordenação de intents + commit |
| Translator | 80%+ | Resolução de locale + ALS |
| `get-user.ts`, `get-environment.ts` | 70%+ | Funções pequenas mas com branches |

## Regras

1. **`scenarios-overview.md` obrigatório em cada pasta de teste** — atualizar no mesmo commit dos testes.
2. **`makeSut` em `shared-helpers.ts`** — nunca inline nos `it`. Permite re-uso e refatoração de DI centralizada.
3. **`BaseStub` para stubs com error injection.** Stubs simples implementam a interface diretamente; stubs que precisam simular erros herdam `BaseStub`.
4. **`vi.spyOn(cds, 'run')` para repositórios** — não mockar o módulo `@sap/cds` inteiro (quebra outras importações).
5. **`vi.restoreAllMocks()` no `beforeEach`** — evita leak de mocks entre testes.
6. **Nunca testar o framework CAP.** Não testar `cds.connect.to` em si — testar o **uso** que o adapter faz dele.

## Alias canônico

`@tests/` → `test/*` (declarado em `tsconfig.json` + `vitest.config.ts`). Subpasta canônica para stubs compartilhados: `test/unit/shared/stubs/`.
