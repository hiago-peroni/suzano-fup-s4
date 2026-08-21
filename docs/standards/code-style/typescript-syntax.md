# TypeScript syntax — padrões do monorepo

> Few-shots extraídos de código real do `mcps/cap-clean-arch/`. **LEIA antes de implementar.** Linkado em `AGENTS.md` §4.2.

## 1. Helpers internos: arrow + const + `;`

Helpers internos (privados ao módulo, não exportados como API pública da unidade) usam **arrow function atribuída a `const`** com trailing semicolon após o corpo da arrow.

### Exemplo extraído de `mcps/cap-clean-arch/src/tools/cap-clean-scaffold-project/generators/package-file.ts`:

```typescript
const getAvailableEngines = (): string => {
    const currentEngine = +process.versions.node.split('.')[0];
    const availableEngines = `^${currentEngine - 2} || ^${currentEngine} || ^${currentEngine + 2} || ^${currentEngine + 4}`;
    return availableEngines;
};
```

### Exemplo extraído de `mcps/cap-clean-arch/src/tools/cap-clean-scaffold-project/generators/mta-files.ts`:

```typescript
const generateMtaHeader = (params: ScaffoldParams): string => {
    return `_schema-version: "3.1"
ID: ${params.name}
version: 0.0.1
description: "${params.description ?? params.name}"`;
};

const generateXsuaaResource = (params: ScaffoldParams): string => {
    return `
    - name: ${params.name}-uaa
      type: org.cloudfoundry.managed-service`;
};
```

Padrão invariante: `const nome = (params): ReturnType => { ... };` — trailing `;` após o `}` que fecha o corpo.

## 2. API pública da unidade: `export function`

A "API pública da unidade" é a função/contrato que outros módulos importam. Usa **declaração de função** com `export function`.

### Exemplo extraído de `mcps/cap-clean-arch/src/tools/cap-clean-scaffold-project/generators/package-file.ts`:

```typescript
export function generatePackageFile(params: ScaffoldParams): FileOutput[] {
    const content = {
        name: params.name,
        description: params.description ?? '',
        version: '1.0.0',
        private: true,
        engines: { node: getAvailableEngines() },
    };

    return [
        {
            path: 'package.json',
            content: JSON.stringify(content, null, 4) + '\n',
        },
    ];
}
```

### Exemplo extraído de `mcps/cap-clean-arch/src/server/mcp-server.ts`:

```typescript
export function createMcpServer(): McpServer {
    const server = new McpServer({ name: 'cap-clean-arch', version: '0.1.0' });
    registerCapCleanScaffoldProject(server);
    return server;
}
```

### Exemplo extraído de `mcps/cap-clean-arch/src/tools/cap-clean-scaffold-project/generators/manifest-file.ts`:

```typescript
export function generateManifestFile(params: ScaffoldParams): FileOutput[] {
    const manifest: CapManifest = {
        version: '1.0.0',
        architecture: 'clean',
        project: { name: params.name, description: params.description },
        services: params.services.map((s) => ({ name: s.name, path: s.path ?? '/api/v1' })),
        entities: [],
    };
    return [{ path: '.cap-mcp.json', content: JSON.stringify(manifest, null, 4) + '\n' }];
}
```

## 3. Regras gerais

- **Indentação:** 4 espaços (não tabs). Verificável via ESLint/Prettier do MCP.
- **Trailing semicolon** após o `}` que fecha o corpo de uma arrow function atribuída a `const`.
- **Imports com alias `@/`:** sempre com extensão `.js` explícita (NodeNext exige):

```typescript
import { foo } from '@/tools/index.js';
import type { ScaffoldParams } from '@/types/scaffold-params.js';
```

- **Helpers vs API pública:** a linha divisória é "este símbolo é importado por outro módulo?". Se sim → `export function`. Se não → `const arrow`.

## 4. Critério de evolução

Este arquivo é minimalista por design (princípio Cursor: *"Start simple. Add rules only when you notice the agent making the same mistake repeatedly."*). Se um agente ressuscitar erro de sintaxe (categoria G5 da taxonomia em `docs/researches/organizacao-projeto/padroes-a-cobrir.md`) após este split, mover o few-shot relevante para inline no `AGENTS.md` §4.2.
