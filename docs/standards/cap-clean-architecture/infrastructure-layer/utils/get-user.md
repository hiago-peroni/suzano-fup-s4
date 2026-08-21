# getUser

`getUser(request)` extrai o usuário logado do request CAP, normalizando entre o formato de produção (IDP/JWT token) e o formato de desenvolvimento local (basic-auth com `attr`). Substitui o `RequestUserExtractorImpl` do Suzano sem a complexidade de classe + DI.

## Por que função e não classe?

A operação é pura: recebe `request`, retorna `User`. Sem estado, sem DI. Uma classe seria overkill e violaria a decisão #10 do research da infrastructure layer — utilitários simples de leitura de contexto são funções.

## Tipo de domínio

```typescript
// src/domain/utils/get-user.ts
import type { EventContext } from '@sap/cds';

export namespace GetUser {
    export type LoggedUser = {
        email: string;
        id?: string;
        tenantId?: string;
        roles?: string[];
    };

    export type RequestWithUser = {
        user: IdpLoggedUser | LocalLoggedUser;
    };

    export type IdpLoggedUser = {
        id: string;
        tokenInfo: {
            getEmail: () => string;
        };
    };

    export type LocalLoggedUser = EventContext['user'];
}
```

## Implementação canônica

```typescript
// src/infrastructure/utils/get-user.ts
import { GetUser } from '@/domain/utils/get-user.js';

export function getUser(request: GetUser.RequestWithUser): GetUser.LoggedUser {
    if (isLocalEnvironment()) {
        return extractLocalUser(request.user as GetUser.LocalLoggedUser);
    }
    return extractIdpUser(request.user as GetUser.IdpLoggedUser);
}

const isLocalEnvironment = (): boolean => {
    return Boolean(process.env.NODE_ENV?.match(/test|local|development/i));
};

const extractLocalUser = (user: GetUser.LocalLoggedUser): GetUser.LoggedUser => {
    return {
        email: user.id,
        id: user.id,
        tenantId: user.attr?.tenantId as string | undefined,
        roles: user.attr?.Groups as string[] | undefined
    };
};

const extractIdpUser = (user: GetUser.IdpLoggedUser): GetUser.LoggedUser => {
    const email = user.tokenInfo?.getEmail()?.toLowerCase();
    return {
        email,
        id: user.id
    };
};
```

## Uso típico no use case

```typescript
// src/application/use-cases/entity-events/user-preferences/before-read.ts (trecho)
import { getUser } from '@/infrastructure/utils/get-user.js';

public async execute(params: BeforeReadUserPreferencesUseCase.Params): Promise<...> {
    try {
        const user = getUser(params.request);
        this.hydrator.hydrate({ request: params.request, userId: user.id! });
        return right(undefined);
    } catch (error) {
        return left(this.handleError(error));
    }
}
```

## Regras

1. **Função, não classe.** Sem DI, sem instância.
2. **Helpers internos como arrow + const** (`const isLocalEnvironment = (): boolean => { ... };`) — alinhado com `docs/standards/typescript-syntax.md`. Esses são **declarações de função**, não constantes de dado; portanto não violam a [regra global de constantes](../README.md#tipagem-e-constantes).
3. **API pública via `export function`** (não arrow).
4. **Tipo `GetUser.LoggedUser` no domínio** — namespace explícito, não declarar types dentro de `infrastructure/`.
5. **Tratamento de ambiente via `NODE_ENV`** com match `/test|local|development/i` — cobre mock-server local e Vitest.
6. **Sem cache.** Cada chamada normaliza independentemente — a operação é barata e o input varia por request.

## Anti-padrões

1. **Nome `getLoggedUser`** — usar `getUser` (mais simples; decisão canônica).
2. **Classe `RequestUserExtractorImpl`** — viola decisão #10. Funções simples são preferíveis a classes sem estado ou DI.
3. **Mutar `request.user`** — sempre criar novo objeto `LoggedUser` sem alterar o input.
4. **Importar `@sap/cds` diretamente no infrastructure** para tipar o `user` — usar o tipo `EventContext['user']` re-exportado pelo domínio via `GetUser.LocalLoggedUser`.
