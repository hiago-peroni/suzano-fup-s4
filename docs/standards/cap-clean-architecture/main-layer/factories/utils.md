# factories/utils/

Singletons transversais que atravessam múltiplas camadas. O único item obrigatório em projetos CAP com i18n é o `translator`.

## translator.ts

O translator é um singleton que gerencia contexto de idioma por requisição via `AsyncLocalStorage`. Exportado como instância pronta para ser usada no `routes/index.ts`.

```typescript
// src/main/factories/utils/translator.ts
import { ResourceManager } from '@sap/textbundle';
import { AsyncLocalStorage } from 'async_hooks';

import { TranslatorImpl } from '@/infrastructure/translator/translator';

export const makeTranslator = () => {
    const resourceManager = new ResourceManager('../../../infrastructure/i18n/i18n');
    return new TranslatorImpl(resourceManager, new AsyncLocalStorage());
};

export const translator = makeTranslator();
```

### Como usar no `routes/index.ts`

```typescript
// src/main/routes/index.ts
import { translator } from '@/main/factories/utils/translator';

service.on('myAction', async (request: any) => {
    return translator.withLanguage(request._language, () => handleMyAction(request));
});
```

O `request._language` é preenchido pelo middleware global definido no `export default` do `routes/index.ts` (ver [routes.md](../routes.md)).

## Quando adicionar outros utils

Adicione um arquivo em `utils/` somente se:

1. O singleton é usado em 3 ou mais handlers/factories.
2. Não pertence a nenhuma outra categoria (`adapters`, `services`, etc.).

Exemplos legítimos além do translator: cache local em memória, circuit breaker compartilhado, feature flag resolver global.
