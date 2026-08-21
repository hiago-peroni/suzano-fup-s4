# Testing — Domain Layer

Testes da domain layer focam em **comportamento puro**: regras de negócio dos models, transições de estado, validações e funções utilitárias com lógica determinística. **Não há cobertura de contratos** (interfaces de repositories, adapters, use cases, hydrators) — interfaces não têm comportamento testável; quem é testado é a implementação na infra/application.

> **O que testar:** models com lógica (validações, transições, derivações) + utils com lógica pura (quando existirem implementações em `infrastructure/utils/` que rodam algoritmos puros — ver §"Cobertura de utils"). **O que não testar:** interfaces, namespaces de tipos, classes de erro (estruturalmente triviais), barrels.

## Estrutura de pastas

A estrutura espelha `src/domain/` mas cobre **somente** as áreas com comportamento testável:

```
test/unit/domain/
├── models/
│   ├── db/
│   │   ├── price-list/
│   │   │   ├── shared-helpers.ts          → makeSut + factories de fixtures
│   │   │   ├── scenarios-overview.md      → OBRIGATÓRIO
│   │   │   └── suites.test.ts             → describe/it cobrindo cenários listados
│   │   ├── data-load/
│   │   │   ├── shared-helpers.ts
│   │   │   ├── scenarios-overview.md
│   │   │   └── suites.test.ts
│   │   └── scenarios-overview.md          → overview da pasta models/db
│   └── sap/
│       └── company/
│           ├── shared-helpers.ts
│           ├── scenarios-overview.md
│           └── suites.test.ts
└── shared/
    └── fixtures/
        └── (opcional — fixtures compartilhadas entre múltiplos models)
```

> Não há `test/unit/domain/repositories/`, `test/unit/domain/use-cases/`, `test/unit/domain/adapters/`, `test/unit/domain/hydrators/` — essas são interfaces. Os testes correspondentes vivem em `test/unit/infrastructure/` (impl) e `test/unit/application/` (impl) das respectivas camadas.

## O que testar (e o que não testar)

| Artefato | Testar? | Onde testa | Justificativa |
|---|---|---|---|
| `XxxModel` com `validate()`/`canBe*()`/transições | ✅ Sim | `test/unit/domain/models/<contexto>/<entidade>/` | Regra de negócio pura — alvo crítico |
| `XxxModel` puro (só getters/`with`/`toObject`) | ⚠️ Opcional | — | Sem lógica = sem regressão possível; cobertura de baixo valor |
| `AbstractError` + subclasses | ❌ Não | — | Estruturalmente triviais (super + code fixo) |
| `interface XxxRepository` / `XxxUseCase` / `XxxApi` / `XxxHydrator` | ❌ Não | — | Interfaces não têm comportamento |
| `namespace XxxUseCase { Params, Result }` | ❌ Não | — | Tipos não têm comportamento |
| `Translator` (interface) | ❌ Não | — | Interface — impl está em `test/unit/infrastructure/utils/translator/` |

## `scenarios-overview.md` obrigatório

Toda pasta de teste no domain (que agrupa um model ou util) deve conter um `scenarios-overview.md` mapeando todos os cenários cobertos. Regra global do `AGENTS.md` raiz §5.2 — atualizar no mesmo commit dos testes.

### Few-shot de `scenarios-overview.md`

```markdown
# Scenarios — domain/models/db/price-list

| Arquivo | Cenário | Resultado esperado |
|---|---|---|
| suites.test.ts | `with(props)` retorna instância | Model criado com props clonados |
| suites.test.ts | `validate()` com nome vazio | `hasError: true` + mensagem `priceList.name.required` |
| suites.test.ts | `validate()` com nome > 100 chars | `hasError: true` + mensagem `priceList.name.tooLong` |
| suites.test.ts | `validate()` com props válidos | `hasError: false`, `errorMessages: []` |
| suites.test.ts | `isActive()` com status `'ACTIVE'` | Retorna `true` |
| suites.test.ts | `isActive()` com status `'DRAFT'` | Retorna `false` |
| suites.test.ts | `canBeArchived()` com `status='ACTIVE'` e `validItemCount > 0` | Retorna `true` |
| suites.test.ts | `canBeArchived()` com `status='ACTIVE'` e `validItemCount = 0` | Retorna `false` |
| suites.test.ts | `decrementValidItemCount()` com count > 0 | Retorna novo model com count - 1 |
| suites.test.ts | `decrementValidItemCount()` com count = 0 | Lança Error `priceList.cannotDecrementBelowZero` |
| suites.test.ts | `toObject()` retorna cópia rasa de props | Modificações no retorno não afetam o model |
```

## Padrão `makeSut` em `shared-helpers.ts`

Cada pasta de teste tem `shared-helpers.ts` que exporta `makeSut()` retornando `{ sut, ...fixtures }`. Nunca instanciar o model inline nos `it` — isso impede reuso e centraliza a criação de fixtures válidos.

```typescript
// test/unit/domain/models/db/price-list/shared-helpers.ts
import { PriceListModel, type PriceListProps } from '@/domain/models/db/price-list.js';

export type SutTypes = {
    sut: PriceListModel;
};

export const makeValidProps = (overrides: Partial<PriceListProps> = {}): PriceListProps => ({
    id: 'PL-001',
    tenantId: 'T0001',
    name: 'Lista A',
    description: undefined,
    status: 'ACTIVE',
    validItemCount: 5,
    createdAt: new Date('2026-01-01T00:00:00Z'),
    ...overrides,
});

export const makeSut = (overrides: Partial<PriceListProps> = {}): SutTypes => {
    const sut = PriceListModel.with(makeValidProps(overrides));
    return { sut };
};
```

> O `makeValidProps` é exportado separado do `makeSut` para que cenários que precisem **só** dos props (ex.: testar erros de `validate()` antes de instanciar) possam reutilizar.

## Few-shot — `suites.test.ts`

```typescript
// test/unit/domain/models/db/price-list/suites.test.ts
import { describe, it, expect } from 'vitest';

import { PriceListModel } from '@/domain/models/db/price-list.js';

import { makeSut, makeValidProps } from './shared-helpers.js';

describe('PriceListModel', () => {
    describe('with()', () => {
        it('retorna instância de PriceListModel', () => {
            const { sut } = makeSut();
            expect(sut).toBeInstanceOf(PriceListModel);
        });

        it('expõe props via getters', () => {
            const { sut } = makeSut({ id: 'PL-XYZ', name: 'Custom' });
            expect(sut.id).toBe('PL-XYZ');
            expect(sut.name).toBe('Custom');
        });
    });

    describe('validate()', () => {
        it('retorna hasError=true com nome vazio', () => {
            const sut = PriceListModel.with(makeValidProps({ name: '' }));
            const result = sut.validate();
            expect(result.hasError).toBe(true);
            expect(result.errorMessages).toContain('priceList.name.required');
        });

        it('retorna hasError=true com nome > 100 chars', () => {
            const sut = PriceListModel.with(makeValidProps({ name: 'a'.repeat(101) }));
            const result = sut.validate();
            expect(result.hasError).toBe(true);
            expect(result.errorMessages).toContain('priceList.name.tooLong');
        });

        it('retorna hasError=false com props válidos', () => {
            const { sut } = makeSut();
            const result = sut.validate();
            expect(result.hasError).toBe(false);
            expect(result.errorMessages).toEqual([]);
        });
    });

    describe('canBeArchived()', () => {
        it('retorna true quando status=ACTIVE e validItemCount > 0', () => {
            const { sut } = makeSut({ status: 'ACTIVE', validItemCount: 5 });
            expect(sut.canBeArchived()).toBe(true);
        });

        it('retorna false quando status=ACTIVE e validItemCount = 0', () => {
            const { sut } = makeSut({ status: 'ACTIVE', validItemCount: 0 });
            expect(sut.canBeArchived()).toBe(false);
        });

        it('retorna false quando status != ACTIVE', () => {
            const { sut } = makeSut({ status: 'DRAFT' });
            expect(sut.canBeArchived()).toBe(false);
        });
    });

    describe('decrementValidItemCount()', () => {
        it('retorna novo model com count - 1', () => {
            const { sut } = makeSut({ validItemCount: 5 });
            const decremented = sut.decrementValidItemCount();
            expect(decremented.validItemCount).toBe(4);
        });

        it('não muta o model original (imutabilidade)', () => {
            const { sut } = makeSut({ validItemCount: 5 });
            sut.decrementValidItemCount();
            expect(sut.validItemCount).toBe(5);
        });

        it('lança Error quando count = 0', () => {
            const { sut } = makeSut({ validItemCount: 0 });
            expect(() => sut.decrementValidItemCount()).toThrow('priceList.cannotDecrementBelowZero');
        });
    });

    describe('toObject()', () => {
        it('retorna cópia rasa de props', () => {
            const { sut } = makeSut();
            const obj = sut.toObject();
            expect(obj).toEqual(makeValidProps());
        });

        it('mutações no objeto retornado não afetam o model', () => {
            const { sut } = makeSut();
            const obj = sut.toObject();
            obj.name = 'Mutado';
            expect(sut.name).not.toBe('Mutado');
        });
    });
});
```

## Cobertura recomendada

| Tipo | Cobertura mínima | Justificativa |
|---|---|---|
| Model com `validate()`/`canBe*()`/transições | 90%+ | Regra crítica do domínio — qualquer regressão é silenciosa |
| Model puro (só getters + `with` + `toObject`) | Opcional (50%+ se incluir) | Estruturalmente trivial; valor agregado baixo |
| Fakes/fixtures (`makeValidProps`) | Não testar | É helper de teste, não código de produção |

## Cobertura de utils

`domain/utils/` declara apenas **contratos** (interfaces e namespaces de tipos). Não há comportamento aqui. Portanto **não há `test/unit/domain/utils/`**. As implementações vivem em `infrastructure/utils/` e seus testes ficam em `test/unit/infrastructure/utils/` ([infrastructure-layer/testing.md](../infrastructure-layer/testing.md)).

> Caso histórico: RVE colocou implementações puras (`maskCpf`, `validateFlightDate`) dentro de `domain/utils/` e escreveu testes em `test/unit/domain/utils/`. Isso é anti-padrão — a implementação devia estar em `infrastructure/utils/` ou como método do model. O standard exclui essa opção: implementação fora do model nunca vive em `domain/`.

## Regras

1. **`scenarios-overview.md` obrigatório** em cada pasta de teste — atualizar no mesmo commit dos testes (`AGENTS.md §5.2`).
2. **`makeSut` em `shared-helpers.ts`** — nunca instanciar inline nos `it`. Permite reuso e centralização de fixtures.
3. **`makeValidProps` separado do `makeSut`** — para cenários que precisam dos props sem o model.
4. **Testar models com lógica de negócio** (`validate()`, transições, derivações). Models puros (só getters/`with`) são opcionais.
5. **Não testar interfaces, namespaces de tipos, ou barrels** — sem comportamento testável.
6. **Não testar classes de erro** — estruturalmente triviais.
7. **Sem mocks de I/O no domain** — models são puros; se você precisa mockar `cds.run`, está testando coisa errada (vai para `test/unit/infrastructure/`).
8. **Cobertura mínima 90% para models com `validate()` ou transições.** Para models puros, opcional.
9. **Imutabilidade testada** — quando o model retorna novo model em transições (`decrementValidItemCount`), incluir cenário "não muta o original".

## Anti-padrões

### 1. Instanciação inline nos `it`

```typescript
// ❌ ERRADO — props duplicados em cada teste
it('valida nome vazio', () => {
    const sut = PriceListModel.with({
        id: 'PL-1',
        tenantId: 'T0001',
        name: '',
        // ... 5 outros campos
    });
    expect(sut.validate().hasError).toBe(true);
});
```

```typescript
// ✅ CORRETO — makeSut centraliza
it('valida nome vazio', () => {
    const sut = PriceListModel.with(makeValidProps({ name: '' }));
    expect(sut.validate().hasError).toBe(true);
});
```

### 2. Testar interface

```typescript
// ❌ ERRADO — testar interface não faz sentido
import type { PriceListRepository } from '@/domain/repositories/price-lists.js';

describe('PriceListRepository', () => {
    it('declara findById', () => {
        // ... como testar uma interface?
    });
});
```

```
✅ CORRETO — não criar test/unit/domain/repositories/
                       Testar PriceListRepositoryImpl em test/unit/infrastructure/repositories/models/price-lists/
```

### 3. Testar `AbstractError` + subclasses

```typescript
// ❌ ERRADO — testar classe estruturalmente trivial
describe('BadRequestError', () => {
    it('tem code 400', () => {
        const err = new BadRequestError('test');
        expect(err.code).toBe(400);
    });
});
```

```
✅ CORRETO — não criar test/unit/domain/errors/
                       O code fixo é parte do design; sem regressão possível.
```

### 4. `scenarios-overview.md` ausente ou desatualizado

```markdown
❌ Pasta tem suites.test.ts mas não tem scenarios-overview.md.
✅ Toda pasta de teste no domain tem scenarios-overview.md atualizado no mesmo commit dos testes.
```

### 5. Mock de `cds.run` em teste de model

```typescript
// ❌ ERRADO — model não chama cds.run
import cds from '@sap/cds';
import { vi } from 'vitest';

describe('PriceListModel', () => {
    it('save chama cds.run', async () => {
        vi.spyOn(cds, 'run').mockResolvedValueOnce([]);
        // ... model.save() não existe — save é do repository
    });
});
```

> Models não fazem I/O. Se o teste precisa mockar `cds.run`, o teste está na pasta errada (vai para `test/unit/infrastructure/repositories/`).

### 6. Testar `with`/`toObject` de model puro com cobertura alta

```typescript
// ❌ EXCESSO — model anêmico só com getters/with/toObject
// 20 testes para um model que não tem lógica
```

> Cobertura de model puro é opcional. Foco no domínio rico — onde há regras, validação, transição. Models puros sem lógica são candidatos a refatoração para **anêmico legítimo** (ex.: representação direta de payload externo) ou consolidação (ex.: virar campo de um model maior).

### 7. Testar imutabilidade omitindo o cenário-chave

```typescript
// ❌ ERRADO — testa transição mas não a imutabilidade
it('decrementa o count', () => {
    const sut = PriceListModel.with({ validItemCount: 5 });
    const decremented = sut.decrementValidItemCount();
    expect(decremented.validItemCount).toBe(4);
});
// Falta: expect(sut.validItemCount).toBe(5);
```

```typescript
// ✅ CORRETO — testa transição E imutabilidade do original
it('decrementa o count', () => {
    const sut = PriceListModel.with({ validItemCount: 5 });
    const decremented = sut.decrementValidItemCount();
    expect(decremented.validItemCount).toBe(4);
});

it('não muta o model original', () => {
    const sut = PriceListModel.with({ validItemCount: 5 });
    sut.decrementValidItemCount();
    expect(sut.validItemCount).toBe(5);
});
```
