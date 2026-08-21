# factories/repositories/

Instanciam implementações concretas dos repositórios definidos em `domain/repositories/*`.

## Regras

- O tipo de retorno **sempre** é a interface do domínio (`): PriceListRepository`), nunca a implementação concreta.
- Repositórios são **stateless** por padrão: nova instância a cada chamada (sem singleton).
- Um arquivo por repositório, nomeado no plural correspondendo à entidade: `price-lists.ts`, `tenants.ts`, `part-numbers.ts`.
- A implementação (`*RepositoryImpl`) recebe suas dependências internas (CDS, DB) no próprio construtor — a factory não passa infra diretamente.

---

## Shape canônico

```typescript
// src/main/factories/repositories/price-lists.ts
import type { PriceListRepository } from '@/domain/repositories/price-lists';
import { PriceListRepositoryImpl } from '@/infrastructure/persistence/price-lists-repository';

export const makePriceListRepository = (): PriceListRepository => {
    return new PriceListRepositoryImpl();
};
```

---

## Variação com dependência explícita

Quando o repositório precisa de um adapter externo (ex.: um cliente HTTP para buscar dados de um sistema externo), receba-o via factory de adapter:

```typescript
// src/main/factories/repositories/sap-suppliers.ts
import type { SapSupplierRepository } from '@/domain/repositories/sap-suppliers';
import { SapSupplierRepositoryImpl } from '@/infrastructure/persistence/sap-suppliers-repository';
import { makeSapHttpClient } from '@/main/factories/adapters/sap-http-client';

export const makeSapSupplierRepository = (): SapSupplierRepository => {
    return new SapSupplierRepositoryImpl(makeSapHttpClient());
};
```

---

## Naming

| Entidade | Nome do arquivo | Nome da função |
|---|---|---|
| `PriceList` | `price-lists.ts` | `makePriceListRepository` |
| `Tenant` | `tenants.ts` | `makeTenantRepository` |
| `PartNumber` | `part-numbers.ts` | `makePartNumberRepository` |
| `TenantErpConnection` | `tenant-erp-connections.ts` | `makeTenantErpConnectionRepository` |
