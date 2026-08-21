# Translator

`TranslatorImpl` traduz chaves de mensagem para texto localizado usando bundles `.properties` do `@sap/textbundle`. O locale é resolvido via `AsyncLocalStorage` (ALS) — define-se uma vez por request e propaga automaticamente para qualquer chamada `translate()` dentro do escopo.

## Por que ALS?

| Abordagem | Como funciona | Problema |
|---|---|---|
| `translate(key, locale)` (LE44) | locale passado em cada chamada | Verboso; polui assinatura de todos os métodos da cadeia |
| `translate(key)` + ALS (MRO/RVE) | locale definido uma vez no middleware, propagado por contexto | Assinatura simples; locale não contamina a cadeia de chamadas |
| locale no constructor | locale fixo na instância | Errado — locale varia por request, não por instância |

A abordagem ALS é o padrão recomendado pelo CAP para i18n por request.

## Estrutura

```
infrastructure/utils/translator/
├── translator.ts
└── i18n/
    ├── i18n.properties              → chaves base (fallback EN)
    ├── i18n_pt.properties           → PT-BR
    └── i18n_es.properties           → ES
```

## Interface no domínio

Todo o contrato (incluindo o `LanguageContext` do ALS e os tipos do `translate`) vive em `domain/utils/translator.ts`:

```typescript
// src/domain/utils/translator.ts
export namespace Translator {
    export type LanguageContext = { language: string };

    export type TranslateParams = string | { text: string; args?: string[] };
}

export interface Translator {
    translate(params: Translator.TranslateParams): string;
    withLanguage<T>(language: string, fn: () => T): T;
}
```

## Implementação canônica

```typescript
// src/infrastructure/utils/translator/translator.ts
import { ResourceManager } from '@sap/textbundle';
import { AsyncLocalStorage } from 'async_hooks';

import type { Translator } from '@/domain/utils/translator.js';

export class TranslatorImpl implements Translator {
    private readonly DEFAULT_LANGUAGE = 'en';
    private readonly PT_BUNDLE = 'pt-BR';
    private readonly ES_BUNDLE = 'es';

    constructor(
        private readonly resourceManager: ResourceManager,
        private readonly asyncLocalStorage: AsyncLocalStorage<Translator.LanguageContext>
    ) {}

    public withLanguage<T>(language: string, fn: () => T): T {
        return this.asyncLocalStorage.run({ language }, fn);
    }

    public translate(params: Translator.TranslateParams): string {
        const text = typeof params === 'string' ? params : params.text;
        const args = typeof params === 'string' ? undefined : params.args;

        const language = this.getCurrentLanguage();
        const bundleLanguage = this.resolveBundleLanguage(language);
        const bundle = this.resourceManager.getTextBundle(bundleLanguage);
        return bundle.getText(text, args) || text;
    }

    private getCurrentLanguage(): string {
        const context = this.asyncLocalStorage.getStore();
        return context?.language || this.DEFAULT_LANGUAGE;
    }

    private resolveBundleLanguage(language: string): string {
        if (language === 'pt' || language === 'pt-BR') {
            return this.PT_BUNDLE;
        }
        if (language === 'es' || language === 'es-ES') {
            return this.ES_BUNDLE;
        }
        return this.DEFAULT_LANGUAGE;
    }
}
```

Note que o `args` no `translate()` agora é tipado como `string[]` direto no domain — o cast `args as string[]` desapareceu, alinhado à regra de [tipagem sem cast inline](../../README.md#tipagem-e-constantes).

## Factory

A factory vive em `main/factories/utils/translator.ts` — única camada onde `let instance` em module-level é tolerado (exceção da regra global de [constantes na classe](../../README.md#tipagem-e-constantes), exclusiva da camada `main/factories/`).

```typescript
// src/main/factories/utils/translator.ts
import { ResourceManager } from '@sap/textbundle';
import { AsyncLocalStorage } from 'async_hooks';

import type { Translator } from '@/domain/utils/translator.js';
import { TranslatorImpl } from '@/infrastructure/utils/translator/translator.js';

let instance: Translator | null = null;

export const makeTranslator = (): Translator => {
    if (!instance) {
        const resourceManager = new ResourceManager('./src/infrastructure/utils/translator/i18n/i18n');
        const als = new AsyncLocalStorage<Translator.LanguageContext>();
        instance = new TranslatorImpl(resourceManager, als);
    }
    return instance;
};

export const translator = makeTranslator();
```

O `ResourceManager` recebe o caminho do bundle **sem extensão e sem sufixo de locale** — `@sap/textbundle` resolve automaticamente `i18n.properties`, `i18n_pt.properties`, etc. O singleton lazy garante que os bundles sejam carregados apenas uma vez. O ALS é tipado com `Translator.LanguageContext` do domain (não com `{ language: string }` literal inline).

## Uso no middleware / route

Chamar `translator.withLanguage(req.locale, fn)` no início de cada request para estabelecer o locale no ALS. Qualquer `translator.translate('key')` dentro do callback — inclusive dentro de erros de domínio — resolve no locale correto.

```typescript
// src/main/routes/index.ts (trecho)
service.before('*', (req) => {
    const language = req.locale ?? 'en';
    return translator.withLanguage(language, () => Promise.resolve());
});
```

## Uso na classe de erro

`AbstractError` em `domain/errors/` pode invocar `translator.translate(this.code)` no `toJSON` para fornecer a mensagem traduzida ao controller, sem receber o translator por parâmetro (ele vem do singleton exportado pela factory).

## Tipagem e constantes

Aplicar as 3 regras globais da [seção "Tipagem e constantes" do README da infrastructure-layer](../../README.md#tipagem-e-constantes). Especificamente para o translator:

- **`Translator.LanguageContext` e `Translator.TranslateParams`** vivem no namespace da interface em `domain/utils/translator.ts`. Zero declaração local.
- **Locales suportados (`'en'`, `'pt-BR'`, `'es'`)** ficam como `private readonly DEFAULT_LANGUAGE`, `PT_BUNDLE`, `ES_BUNDLE` da classe — não como string literal espalhada nem como `const` em module-level.
- **O cast `args as string[]` desaparece** porque `Translator.TranslateParams.args` já é tipado como `string[]` no domain.

## Regras

1. **Sempre via DI no constructor** — `ResourceManager` + `AsyncLocalStorage` injetados pela factory. Não instanciar dentro da classe.
2. **Singleton (instância única).** O `ResourceManager` carrega bundles na primeira chamada; não recriar.
3. **Sem parâmetro `locale` no método `translate()`** — o locale vem exclusivamente do ALS.
4. **Fallback no `||`:** se a chave não existir no bundle, retornar a própria chave (`bundle.getText(text, args) || text`). Evita texto vazio na UI.
5. **Mapeamento de locale centralizado em `resolveBundleLanguage`.** Suportar pelo menos `pt`, `pt-BR`, `es`, `es-ES`, `en`. Estender quando necessário.
6. **Locales suportados como `private readonly` da classe** — `DEFAULT_LANGUAGE`, `PT_BUNDLE`, `ES_BUNDLE`. Adicionar novos locales é editar a classe, não distribuir strings literais nos `if`/`return`.

## Anti-padrões

1. **`translate(key, locale)` com parâmetro** — padrão LE44 (mais verboso, sem ALS).
2. **`ResourceManager` instanciado dentro da classe** (sem DI) — dificulta teste e viola SRP.
3. **Hardcode de `path.join(__dirname, ...)` no constructor** — usar caminho relativo gerenciado pela factory.
4. **`const DEFAULT_LANG = 'en'`** em module-level — quebra a regra global de constantes. Sempre `private readonly` da classe.
5. **`args as string[]` no `bundle.getText`** — tipar `args` como `string[]` no namespace `Translator.TranslateParams` do domain remove o cast.
