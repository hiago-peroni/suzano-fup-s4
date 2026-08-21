# config/

Registra aliases de path e expõe configuração de ambiente. Em backends CAP o único arquivo obrigatório é `module-alias.ts`.

## module-alias.ts (obrigatório em todos os backends)

Registra o alias `@` apontando para `src/` em runtime. Isso complementa o `tsconfig paths` (que funciona só no TypeScript) para que os imports `@/domain/...` funcionem no JavaScript compilado.

```typescript
// src/main/config/module-alias.ts
import * as moduleAlias from 'module-alias';
import { join } from 'path';

moduleAlias.addAlias('@', join(__dirname, '..', '..'));
```

### Regras

- Deve ser o **primeiro import** do `routes/index.ts`, antes de qualquer import de domínio ou infra.
- Nunca importar `module-alias` fora deste arquivo — um único ponto de registro.
- O `__dirname` aponta para `src/main/config/`; dois `..` sobem para `src/`.

## environment.ts (opcional — variáveis de ambiente)

Use quando o serviço precisa expor configuração de ambiente para outras partes da main layer (ex.: URL de bucket por stage, tipo de banco).

```typescript
// src/main/config/environment.ts
export const getBucket = (): string => {
    if (process.env.NODE_ENV === 'production') {
        return process.env.PROD_BUCKET ?? ''
    };
    if (process.env.NODE_ENV === 'quality') {
        return process.env.STG_BUCKET ?? ''
    };
    return 'local-bucket';
};
```

### Quando não usar

Se a variável de ambiente é usada somente dentro de uma factory específica, declare-a diretamente na factory — não crie um `environment.ts` só para isso. O `environment.ts` é justificado quando 3 ou mais factories precisam do mesmo valor.
