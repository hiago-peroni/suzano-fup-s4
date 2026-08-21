# getEnvironment

`getEnvironment()` lê variáveis de ambiente e credentials de serviços BTP (via `@sap/xsenv`) e retorna um objeto tipado pronto para uso. Substitui o `EnvironmentImpl` do Suzano sem a complexidade de classe + DI.

## Por que função e não classe?

A operação é uma leitura tipada — quase-pura, dado que `xsenv` consulta variáveis de ambiente. Sem estado de instância, sem DI real. Uma classe seria desnecessária e violaria a decisão #10 do research da infrastructure layer.

> **Exceção à regra global de "constantes na classe":** o cache module-level (`let cachedEnvironment`) é tolerado **exclusivamente** para funções utilitárias puras (`get-*.ts`) que não têm classe onde colocar `private readonly`. Para qualquer código que viva dentro de uma classe (adapters, repositories, hydrators), a regra de [`private readonly`](../README.md#tipagem-e-constantes) continua valendo. Não use esse padrão como pretexto para introduzir `let cachedXxx` em adapters.

## Tipo de domínio

O tipo `Environment` é declarado no domínio e customizado por projeto — os campos variam conforme os serviços BTP contratados.

```typescript
// src/domain/utils/get-environment.ts
export namespace GetEnvironment {
    export type Environment = {
        carrierApi: {
            clientId: string;
            clientSecret: string;
            baseUrl: string;
        };
        aws?: {
            region: string;
            accessKeyId: string;
            secretAccessKey: string;
        };
        features: {
            [flag: string]: boolean;
        };
    };
}
```

## Implementação canônica

```typescript
// src/infrastructure/utils/get-environment.ts
import xsenv from '@sap/xsenv';

import { GetEnvironment } from '@/domain/utils/get-environment.js';

let cachedEnvironment: GetEnvironment.Environment | null = null;

export function getEnvironment(): GetEnvironment.Environment {
    if (!cachedEnvironment) {
        cachedEnvironment = readEnvironment();
    }
    return cachedEnvironment;
}

const readEnvironment = (): GetEnvironment.Environment => {
    const userProvidedServices = xsenv.getServices({ env: 'app-user-env' }).env as Record<string, string>;
    return {
        carrierApi: {
            clientId: userProvidedServices.CARRIER_CLIENT_ID,
            clientSecret: userProvidedServices.CARRIER_CLIENT_SECRET,
            baseUrl: userProvidedServices.CARRIER_BASE_URL
        },
        aws: process.env.AWS_REGION
            ? {
                  region: process.env.AWS_REGION,
                  accessKeyId: process.env.AWS_ACCESS_KEY_ID!,
                  secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!
              }
            : undefined,
        features: parseFeatureFlags()
    };
};

const parseFeatureFlags = (): Record<string, boolean> => {
    const raw = process.env.FEATURE_FLAGS ?? '';
    return raw
        .split(',')
        .filter(Boolean)
        .reduce<Record<string, boolean>>((acc, flag) => {
            acc[flag.trim()] = true;
            return acc;
        }, {});
};
```

## Configuração local — `default-env.json`

Em desenvolvimento local, `@sap/xsenv` carrega as credentials de `default-env.json` na raiz do service (nunca versionar este arquivo).

```json
{
    "app-user-env": {
        "env": {
            "CARRIER_CLIENT_ID": "fake-id",
            "CARRIER_CLIENT_SECRET": "fake-secret",
            "CARRIER_BASE_URL": "http://localhost:3999"
        }
    }
}
```

## Uso típico em adapter externo

```typescript
// src/infrastructure/adapters/external-api/carrier/carrier-api.ts (trecho)
import { getEnvironment } from '@/infrastructure/utils/get-environment.js';

export class CarrierApiImpl implements CarrierApi {
    private readonly config = getEnvironment().carrierApi;

    public async authenticate(): Promise<string> {
        const response = await fetch(`${this.config.baseUrl}/auth`, {
            method: 'POST',
            body: JSON.stringify({
                clientId: this.config.clientId,
                clientSecret: this.config.clientSecret
            })
        });
        // ...
    }
}
```

## Regras

1. **Função, não classe.**
2. **Cache module-level.** Variáveis BTP são imutáveis em runtime — cachear evita reler `xsenv` a cada chamada em hot paths.
3. **Helpers internos como arrow + const** (`const readEnvironment = (): GetEnvironment.Environment => { ... };`).
4. **Tipo `Environment` no domínio** — todos os campos opcionais conforme aplicabilidade do projeto.
5. **`xsenv.getServices({ env: '<binding-name>' })` para credentials BTP.** `process.env.XXX` aceitável para flags simples ou credentials sem binding BTP nativo (ex.: AWS).
6. **Validar credentials obrigatórias no startup, não no acesso.** Ideal: lançar `Error('Missing CARRIER_CLIENT_ID')` dentro de `readEnvironment` quando a chave for ausente em produção — falha rápida antes da primeira requisição. (Recomendado; depende do projeto.)

## Anti-padrões

1. **Classe `EnvironmentImpl`** — viola decisão #10. Sem estado real de instância, a classe não agrega valor.
2. **`process.env.XXX!` espalhado em vários adapters** — centralizar tudo no `getEnvironment()` (single source of truth para configuração de runtime).
3. **Sem cache** — reler `xsenv` a cada chamada tem custo em hot paths de alta concorrência.
4. **`console.error(env)` para debug** — vazamento de credentials sensíveis em logs de produção.
