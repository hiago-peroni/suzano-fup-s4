# Utils

`domain/utils/` declara os contratos dos utilitários transversais — artefatos que **não pertencem** a nenhum domínio específico mas precisam de tipagem compartilhada para que múltiplos use cases consumam de forma uniforme. São três contratos canônicos:

| Arquivo | Contrato | Implementação na infra |
|---|---|---|
| `translator.ts` | `interface Translator` (i18n via `AsyncLocalStorage`) | `infrastructure/utils/translator/translator.ts` (classe `TranslatorImpl`) |
| `get-user.ts` | `namespace GetUser` (sem interface — só tipos) | `infrastructure/utils/get-user.ts` (função pura `getUser(request)`) |
| `get-environment.ts` | `namespace GetEnvironment` (sem interface — só tipos) | `infrastructure/utils/get-environment.ts` (função pura `getEnvironment()`) |

> **Diferença vs adapters:** o `Translator` é uma classe com estado (`ResourceManager`, `AsyncLocalStorage`) e DI real → fica em `utils/`. `getUser` e `getEnvironment` são funções puras de leitura de contexto runtime — não têm classe ou DI, só shape de dados → ficam em `utils/` apenas pelo namespace de tipos.

## Estrutura

```
src/domain/utils/
├── translator.ts                  → Translator (interface + namespace)
├── get-user.ts                    → namespace GetUser (LoggedUser, RequestWithUser, IdpLoggedUser)
└── get-environment.ts             → namespace GetEnvironment (Environment shape)
```

## Translator — interface + namespace

`Translator` é o contrato de internacionalização. Recebe uma chave i18n (ou objeto `{ text, args }`) e devolve a string traduzida. O locale **não é parâmetro do método** — é resolvido via `AsyncLocalStorage` ([infrastructure-layer/utils/translator/README.md](../../infrastructure-layer/utils/translator/README.md)).

```typescript
// src/domain/utils/translator.ts
export namespace Translator {
    export type LanguageContext = {
        language: string;
    };

    export type TranslateParams = string | { text: string; args?: string[]; };
}

export interface Translator {
    translate(params: Translator.TranslateParams): string;
    withLanguage<T>(language: string, fn: () => T): T;
}
```

Pontos canônicos:

- **`translate(params)`** aceita string crua (chave i18n) ou objeto com `args` para interpolação.
- **`withLanguage<T>(language, fn)`** estabelece o locale no `AsyncLocalStorage` para a duração de `fn`. Tipicamente chamado uma vez por request no middleware.
- **Sem `locale` como parâmetro de `translate`** — viria do ALS via `withLanguage`. Isso desacopla o consumidor da gestão de locale.
- **`LanguageContext` no namespace** — usado pela infra para tipar o `AsyncLocalStorage<Translator.LanguageContext>`.

> **Localização canônica:** `domain/utils/translator.ts` (arquivo único). Não criar pasta `utils/translator/` no domain — pasta com bundles `.properties` é da infra. Renomeação de `domain/translation/translator.ts` (LE44) → `domain/utils/translator.ts` entra como débito técnico.

## GetUser — namespace de tipos

`getUser` é uma **função pura** que extrai o usuário logado do `request` CAP, normalizando entre formatos (IDP/JWT em produção, basic-auth em dev). O contrato no domain é apenas o namespace `GetUser` com os tipos — sem interface (a função é importada direto).

```typescript
// src/domain/utils/get-user.ts
export namespace GetUser {
    export type LoggedUser = {
        email: string;
        id?: string;
        tenantId?: string;
        roles?: string[];
    };

    export type RequestWithUser = {
        user: GetUser.IdpLoggedUser | GetUser.LocalLoggedUser;
    };

    export type IdpLoggedUser = {
        id: string;
        tokenInfo: {
            getEmail: () => string;
        };
    };

    export type LocalLoggedUser = {
        id: string;
        attr?: {
            tenantId?: string;
            Groups?: string[];
            [key: string]: unknown;
        };
    };
}
```

Pontos canônicos:

- **`LoggedUser` é o shape canônico** — o que a aplicação usa após normalização.
- **`IdpLoggedUser` e `LocalLoggedUser` documentam as duas variantes** que entram (produção vs dev) — sem importar `EventContext['user']` do `@sap/cds`.
- **Sem interface `GetUser`** — a implementação é função pura (`export function getUser(request)`), não classe.

> A implementação na infra (`infrastructure/utils/get-user.ts`) é uma função, não classe. Ver [infrastructure-layer/utils/get-user.md](../../infrastructure-layer/utils/get-user.md). O nome canônico é `getUser` (não `getLoggedUser` — anti-padrão Suzano).

## GetEnvironment — namespace de tipos

`getEnvironment()` lê variáveis de ambiente e credentials de serviços BTP. Retorna um objeto tipado (`Environment`). O contrato no domain é apenas o namespace `GetEnvironment` com o shape — implementação na infra.

```typescript
// src/domain/utils/get-environment.ts
export namespace GetEnvironment {
    export type Environment = {
        carrierApi: GetEnvironment.CarrierApiCredentials;
        s4Destination?: GetEnvironment.S4DestinationCredentials;
        aws?: GetEnvironment.AwsCredentials;
        features: GetEnvironment.FeatureFlags;
    };

    export type CarrierApiCredentials = {
        clientId: string;
        clientSecret: string;
        baseUrl: string;
    };

    export type S4DestinationCredentials = {
        destinationName: string;
        clientId: string;
        clientSecret: string;
    };

    export type AwsCredentials = {
        region: string;
        accessKeyId: string;
        secretAccessKey: string;
    };

    export type FeatureFlags = Record<string, boolean>;
}
```

Pontos canônicos:

- **`Environment` é o shape consolidado** que a aplicação consome.
- **Cada subseção é um tipo do namespace** (`CarrierApiCredentials`, `AwsCredentials`) — facilita reuso e clareza.
- **Campos opcionais para ambientes que não usam todos os serviços** (`s4Destination?`, `aws?`).
- **Customizar por projeto** — cada serviço tem credentials diferentes; o shape do `Environment` reflete os bindings BTP do projeto.

> Igual ao `GetUser`: sem interface, só namespace de tipos. A implementação na infra é função pura `getEnvironment()` com cache module-level — exceção tolerada para utils sem classe ([infrastructure-layer/utils/get-environment.md](../../infrastructure-layer/utils/get-environment.md)).

## Quando criar contrato em `utils/` vs em outro lugar

| Caso | Onde fica o contrato | Por quê |
|---|---|---|
| Tradução de mensagens i18n | `domain/utils/translator.ts` | Múltiplos consumidores (use cases, erros, controllers) precisam |
| Extração de usuário do request | `domain/utils/get-user.ts` | Função pura usada por todos os use cases de entity-events |
| Leitura de credentials BTP | `domain/utils/get-environment.ts` | Adapters externos consomem credentials |
| API externa (HTTP, OData, REST) | `domain/adapters/external-api/` | Atravessa fronteira da aplicação |
| Parser de arquivo | `domain/adapters/parsers/` | Atravessa fronteira (filesystem) |
| Coordenação transacional batch | `domain/adapters/unit-of-work.ts` | Adapter de tecnologia (CDS tx) |
| Validação de campo de model | método do model | Pertence ao próprio model |
| Formatação de data específica de campo | método do model | Pertence ao próprio model |
| Constante local de classe | `private readonly` no consumidor | Não vai ao domain |

## Imports permitidos

| O que pode importar | Exemplo |
|---|---|
| Tipos básicos (genéricos, primitivos, `Record`) | — |

| O que **não** pode importar | Motivo |
|---|---|
| `@sap/cds`, `@models/*` | Acoplamento com framework — wrapper local (ver `IdpLoggedUser`/`LocalLoggedUser`) |
| `application/`, `infrastructure/`, `presentation/`, `main/` | Quebra de isolamento |
| Outros artefatos do domain (errors, models, repos, etc.) | Utils são folha — quem consome utils são os outros artefatos |

## Regras

1. **Três contratos canônicos:** `translator.ts`, `get-user.ts`, `get-environment.ts`. Adicionar novos utilitários **só** quando múltiplos consumidores precisarem da mesma abstração.
2. **`Translator` tem interface + namespace** — única classe real em `domain/utils/`.
3. **`GetUser` e `GetEnvironment` têm só namespace** (sem interface) — implementação é função pura.
4. **Sem `Translator.translate(key, locale)`** — locale vem do ALS, nunca de parâmetro.
5. **Sem importar `@sap/cds`** — `IdpLoggedUser`/`LocalLoggedUser` documentam as variantes que entram.
6. **Customização por projeto** — `GetEnvironment.Environment` reflete os bindings BTP do projeto específico.
7. **Sem barrel `utils/index.ts`** — imports diretos por arquivo.

## Anti-padrões

### 1. Pasta `domain/translation/` em vez de `utils/translator.ts`

```
❌  src/domain/translation/translator.ts   (LE44)
✅  src/domain/utils/translator.ts         (canônico)
```

Pasta `translation/` fora de `utils/` é divergência histórica do LE44.

### 2. `Translator.translate(text, language, args?)` com locale como parâmetro

```typescript
// ❌ ERRADO — locale como parâmetro contamina toda a cadeia de chamada
export interface Translator {
    translate(params: { text: string; language: string; args?: string[] }): string;
}
```

```typescript
// ✅ CORRETO — locale via ALS
export interface Translator {
    translate(params: Translator.TranslateParams): string;
    withLanguage<T>(language: string, fn: () => T): T;
}
```

### 3. Nome `getLoggedUser` em vez de `getUser`

```
❌  domain/utils/get-logged-user.ts
✅  domain/utils/get-user.ts
```

`getUser` é o nome canônico — mais simples, mesmo padrão da infra-layer.

### 4. Interface `Environment` em vez de namespace

```typescript
// ❌ ERRADO — interface implica classe + DI
export interface Environment {
    getUserProvidedVariables(): UserProvidedVariables;
}
```

```typescript
// ✅ CORRETO — namespace de tipos; função pura na infra
export namespace GetEnvironment {
    export type Environment = { /* shape */ };
}
```

> Suzano usa `interface Environment` com método — implica classe `EnvironmentImpl` na infra. A decisão canônica é função pura `getEnvironment()`, então o domain só precisa do shape do retorno.

### 5. Implementações em `domain/utils/`

```typescript
// ❌ ERRADO — implementação no domain
// domain/utils/date-validation.ts
export function validateFlightDate(date: Date): boolean { /* ... */ }

// ❌ ERRADO — implementação no domain
// domain/utils/sensitive-data-masking.ts
export function maskCpf(cpf: string): string { /* ... */ }
```

```typescript
// ✅ CORRETO — funções puras de domain ficam como métodos no model
export class FlightModel {
    public isFlightDateValid(): boolean { /* ... */ }
}

export class DocumentModel {
    public getMaskedCpf(): string { /* ... */ }
}
```

> RVE coloca implementações puras (`maskCpf`, `validateFlightDate`) em `domain/utils/`. Anti-padrão — pertence ao model. `domain/utils/` declara **contratos**, não implementa.

### 6. Tipos top-level fora do namespace

```typescript
// ❌ ERRADO — tipo top-level
export type LoggedUser = { /* ... */ };
export type RequestWithUser = { /* ... */ };
```

```typescript
// ✅ CORRETO — agrupado no namespace
export namespace GetUser {
    export type LoggedUser = { /* ... */ };
    export type RequestWithUser = { /* ... */ };
}
```

### 7. Acoplamento com `EventContext['user']`

```typescript
// ❌ ERRADO — importa EventContext do @sap/cds
import type { EventContext } from '@sap/cds';

export namespace GetUser {
    export type LocalLoggedUser = EventContext['user'];
}
```

```typescript
// ✅ CORRETO — wrapper local que documenta o subset usado
export namespace GetUser {
    export type LocalLoggedUser = {
        id: string;
        attr?: {
            tenantId?: string;
            Groups?: string[];
            [key: string]: unknown;
        };
    };
}
```

### 8. Pasta `utils/` com helpers ad-hoc

```
❌  domain/utils/language-codes.ts
❌  domain/utils/sap-language.ts
❌  domain/utils/odata-date.ts
❌  domain/utils/timezone.ts
```

```
✅  domain/utils/translator.ts
✅  domain/utils/get-user.ts
✅  domain/utils/get-environment.ts
```

> MRO acumula 4 helpers em `utils/` e RVE acumula 5 — virou pasta-coringa. O standard mantém **3 contratos canônicos** apenas. Helpers como conversão de língua, normalização de data OData, timezone, etc. ficam:
>
> - **Como método do model** quando são específicos de um campo (`OperationFlightTypeModel.formatDateForOData()`).
> - **Como `private readonly` na classe** que consome (`SapHttpClientImpl.SAP_LANGUAGES = [...]`).
> - **Em outro contrato do domain** se múltiplos consumidores precisarem (raro — geralmente é "model" disfarçado).
