# scripts/

Scripts Node.js executados manualmente via `package.json` scripts. **Nunca importados pela aplicação.**

Ambos os scripts abaixo existem em todos os projetos CAP e são praticamente idênticos entre eles — apenas o nome do serviço e os paths variam.

---

## populate-local-db.ts

Faz deploy do schema CDS para um SQLite local (`db.sqlite`), possibilitando rodar o serviço sem conexão com HANA.

```typescript
// src/main/scripts/populate-local-db.ts
import { execSync } from 'child_process';
import { resolve } from 'path';

const populateSqliteDb = (): void => {
    const deployPaths = [
        resolve(process.cwd(), '..', 'db', 'models'),
        resolve(process.cwd(), 'src', 'main', 'routes'),
    ];
    const execSyncOptions = { cwd: process.cwd(), stdio: 'inherit' as const };
    const command = `cds deploy ${deployPaths.join(' ')} -2 sqlite:${process.cwd()}/db.sqlite --with-mocks`;

    execSync(command, execSyncOptions);
};

populateSqliteDb();
```

**Ajustar por projeto:** os paths em `deployPaths` dependem de onde ficam os modelos CDS do projeto. Se houver modelos de replicação, adicionar o path correspondente ao array.

### Como chamar (package.json)

```json
{
    "scripts": {
        "populate:local": "tsx src/main/scripts/populate-local-db.ts"
    }
}
```

---

## replace-csn-source.ts

Após o build, o CAP gera `srv/csn.json` com `@source` apontando para o path de desenvolvimento. Este script corrige o path para o path de produção esperado pelo deploy no BTP.

```typescript
// src/main/scripts/replace-csn-source.ts
import * as fs from 'fs';
import * as path from 'path';

function updateSourceInJsonSync(): void {
    const filePath = path.join(process.cwd(), 'srv', 'csn.json');

    try {
        const data = fs.readFileSync(filePath, 'utf8');
        const jsonData = JSON.parse(data) as {
            definitions: Record<string, { '@source'?: string }>;
        };

        const svcName = 'MyServiceName';                                     // ← ajustar
        const devPath = 'my-service/src/main/routes/index.cds';              // ← ajustar
        const buildPath = 'srv/src/main/routes/index.cds';                   // ← ajustar

        if (jsonData.definitions[svcName]?.['@source'] === devPath) {
            jsonData.definitions[svcName]['@source'] = buildPath;
        }

        fs.writeFileSync(filePath, JSON.stringify(jsonData, null, 2), 'utf8');
    } catch {
        // Arquivo pode não existir ainda durante dev — ignorar silenciosamente
    }
}

updateSourceInJsonSync();
```

**Ajustar por projeto:** `svcName` é o nome do serviço definido no `index.cds` (ex.: `MroApplicationService`); `devPath` e `buildPath` dependem do nome da pasta do serviço.

### Como chamar (package.json)

```json
{
    "scripts": {
        "build:fix-csn": "tsx src/main/scripts/replace-csn-source.ts"
    }
}
```

---

## Regras

- Scripts nunca são importados por outros módulos TS.
- Cada script é autoexecutável — invoca sua função principal na última linha.
- Erros nos scripts de build podem ser ignorados silenciosamente (`try/catch` vazio) quando o arquivo alvo pode não existir ainda.
