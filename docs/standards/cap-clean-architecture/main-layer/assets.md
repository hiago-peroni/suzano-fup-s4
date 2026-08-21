# assets/

Subpasta **opcional**. Existe apenas quando o serviço precisa servir arquivos estáticos diretamente em modo de desenvolvimento local (ex.: templates para download que em produção viriam de um object store como BTP Object Store ou S3).

## Quando criar

Crie `assets/` somente se **todas** as condições forem verdadeiras:

1. O serviço precisa disponibilizar um arquivo binário/estático (Excel, PDF, imagem) ao frontend.
2. Em produção esse arquivo é servido por um serviço externo (object store, CDN).
3. Em dev local não há acesso ao serviço externo, então o arquivo é servido diretamente pelo CAP via Express static.

Se o arquivo for sempre servido por infra externa (mesmo em dev), `assets/` não é necessário.

## Estrutura interna

Organize por tipo de asset, sem profundidade excessiva:

```
src/main/assets/
└── templates/
    └── price-list-template.xlsx
```

## Como conectar ao Express (few-shot)

A montagem do static server acontece no `routes/index.ts`, condicionada a `NODE_ENV === 'dev'`:

```typescript
// src/main/routes/index.ts
import cds from '@sap/cds';
import express from 'express';
import { join } from 'path';

export default (service: Service) => {
    // ... handlers ...

    if (process.env.NODE_ENV === 'dev') {
        cds.on('served', () => {
            const assetsTemplatesPath = join(process.cwd(), 'src', 'main', 'assets', 'templates');
            cds.app.use('/_local-files/templates', express.static(assetsTemplatesPath));
        });
    }
};
```

## Regras

- Nunca importar arquivos de `assets/` em código TypeScript — eles são servidos via HTTP estático.
- O path de serve (`/_local-files/...`) é livre; use algo que não colida com rotas OData.
- Em produção, a factory que serve o URL do arquivo deve retornar a URL do object store externo (ver `factories/adapters.md` — shape com switch de ambiente).
