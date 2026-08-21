# Boas práticas de código TypeScript — MCPs Numen DS

Documento canônico de **estilo e qualidade de código TypeScript** para os servidores MCP do monorepo `numen-mcps`. Define as regras de escrita de código que vão **além** da estrutura de pastas: formatação, imports, tipagem, funções, logging, naming e estilo de export.

> **Relação com a estrutura:** este documento **não** trata de layout de diretórios (`src/tools/`, `generators/`, `types/`), aliases ou separação de responsabilidades (`tool-executor.ts`). Isso vive em [`AGENTS.md`](../../../AGENTS.md) §3. Aqui o foco é **como o código TypeScript é escrito**, independentemente de qual MCP ou tool.

## Para quem é este documento

| Público | Como consumir |
|---------|---------------|
| **Agente de IA** (Cursor, Claude Code, Copilot) | Leitura obrigatória antes de escrever/editar qualquer `.ts` nos MCPs. É o checklist anti-erro recorrente. |
| **Desenvolvedor humano** | Referência de estilo; o que o `yarn lint` vai cobrar antes de você descobrir no CI. |
| **Code reviewer** | Checklist de review de código TS. Divergência sem justificativa é débito técnico. |

## A ferramenta é a fonte de verdade

Estas regras **não são opinião** — são o que o pipeline cobra. Antes de considerar qualquer arquivo pronto, rode dentro do pacote tocado:

```bash
yarn lint        # ESLint + Prettier (eslint.config.mjs + .prettierrc)
yarn qualityGate # = typecheck && lint && test:unit && build
```

Fontes de configuração (não duplique regras à mão; edite a config):

| Arquivo | Governa |
|---------|---------|
| `mcps/<pacote>/eslint.config.mjs` | Regras de lint (imports, tipagem, naming, tamanho de função) |
| `mcps/<pacote>/.prettierrc` | Formatação (indentação, aspas, vírgulas, largura) |
| `mcps/<pacote>/tsconfig.json` | Compilação e path aliases (`@/` → `src/`) |

> **Regra de ouro:** se você precisa relaxar uma regra, isso é decisão de ADR/RFC — não um `// eslint-disable` solto. Desabilitar regra de lint inline sem justificativa é proibido. Configs canônicas (ESLint flat, Prettier, scripts) são herdadas de `mcps/cap-clean-arch/` enquanto não houver pacote compartilhado (débito técnico documentado no STATE.md).

---

## 1. Formatação

Definida por Prettier + ESLint. **Nunca formate à mão contra a config** — rode `yarn lint --fix` e deixe a ferramenta resolver.

| Regra | Valor | Origem |
|-------|-------|--------|
| Indentação | **4 espaços** (`SwitchCase: 1`) | ESLint `indent`, Prettier `tabWidth: 4` |
| Aspas | **single quotes** (`'...'`) | ESLint `quotes`, Prettier `singleQuote` |
| Ponto e vírgula | **sempre** | ESLint `semi`, Prettier `semi` |
| Trailing comma | **none** (sem vírgula no último item) | Prettier `trailingComma: none` |
| Arrow parens | **always** (parênteses obrigatórios em arrow functions) | Prettier `arrowParens: always` |
| Largura máxima | **180 colunas** (ignora comentários/strings/templates/regex) | ESLint `max-len`, Prettier `printWidth: 180` |
| Espaço em chaves de objeto | `{ a: 1 }` (com espaço) | ESLint `object-curly-spacing: always` |
| Aspas em chaves | só quando necessário (`{ foo: 1, 'bar-baz': 2 }`) | ESLint `quote-props: as-needed` |
| Newline final de arquivo | obrigatória | ESLint `eol-last` |
| Linha em branco no topo/fim de classe | proibida | ESLint `padded-blocks: { classes: never }` |
| Bloco de `if`/`for`/`while` | **sempre com chaves**, mesmo de 1 linha | ESLint `curly: all` |

❌ **Errado:**

```typescript
if (rows.length === 0) return [];          // sem chaves

const config = {a: 1, b: 2,};              // sem espaço, trailing comma

const name = "price-list";                   // aspas duplas

const fn = x => x + 1;                     // arrow sem parênteses
```

✅ **Certo:**

```typescript
if (rows.length === 0) {
    return [];
}

const config = { a: 1, b: 2 };

const name = 'price-list';

const fn = (x) => x + 1;
```

---

## 2. Imports

A área onde mais nasce divergência. As regras abaixo são cumulativas.

### 2.1 Path alias `@/` obrigatório — caminhos relativos proibidos

`no-restricted-imports` bloqueia `./*` e `../*`. Use sempre o alias.

❌ **Errado:**

```typescript
import { ScaffoldParams } from '../types/scaffold-params';
import { generateManifest } from './generators/manifest-file';
```

✅ **Certo:**

```typescript
import { ScaffoldParams } from '@/types/scaffold-params.js';
import { generateManifest } from '@/tools/scaffold-project/generators/manifest-file.js';
```

| Alias | Aponta para | Uso |
|-------|-------------|-----|
| `@/` | `src/` | Código de produção |

> **Exceção:** arquivos `index.ts` (barrels) têm `no-restricted-imports` desligado.

### 2.2 Extensão `.js` explícita (NodeNext)

`module: NodeNext` exige que imports referenciem o output compilado. Mesmo escrevendo `.ts`, o import termina com `.js`.

```typescript
import { createMcpServer } from '@/server/mcp-server.js';
import type { ScaffoldParams } from '@/types/scaffold-params.js';
```

### 2.3 Ordem e agrupamento de imports

`import/order` cobra agrupamento com **linha em branco entre grupos** e ordenação alfabética dentro de cada grupo:

1. `builtin` + `external` (ex.: `node:fs`, `@modelcontextprotocol/sdk`, `path`)
2. `internal` (`@/**`) + `parent`/`sibling`/`index`
3. `unknown` (`@tests/**` quando configurado, sempre por último)

✅ **Exemplo canônico** (extraído de `mcps/cap-clean-arch`):

```typescript
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';

import { createMcpServer } from '@/server/mcp-server.js';
import type { ScaffoldParams } from '@/types/scaffold-params.js';
```

> `sort-imports` está com `ignoreDeclarationSort: true` — quem ordena é o `import/order`. Não brigue com a ferramenta reordenando manualmente.

### 2.4 `import type` para tipos

Quando importa apenas para anotação de tipo (interface, type, contrato), use `import type`. Isso mantém a fronteira clara entre o que é runtime e o que é só compilação.

```typescript
import type { ScaffoldParams } from '@/types/scaffold-params.js';
import type { FileOutput } from '@/types/tool-output.js';
```

---

## 3. Tipagem

### 3.1 Modificadores de acesso explícitos

`explicit-member-accessibility` é **erro**: todo método/propriedade de classe declara `public` / `private` / `protected`. Exceção: o construtor não leva `public` (`overrides: { constructors: 'no-public' }`).

❌ **Errado:**

```typescript
export class ToolExecutor {
    async execute(params) { /* ... */ }     // faltou public
    writeFile() { /* ... */ }                // faltou private
}
```

✅ **Certo:**

```typescript
export class ToolExecutor {
    public async execute(params: ScaffoldParams): Promise<void> {
        /* ... */
    }

    private writeFile(output: FileOutput): void {
        /* ... */
    }
}
```

### 3.2 `any` é permitido pela config, mas é último recurso

`no-explicit-any` está **desligado** — o lint não bloqueia. Isso **não** é licença para usar `any`. Prefira, nesta ordem:

1. O tipo concreto.
2. `unknown` + narrowing quando o tipo é realmente indeterminado (ex.: `catch (error)` → console.error + rethrow).
3. `any` apenas em fronteiras com SDKs sem tipos, isolado e comentado com o porquê.

### 3.3 Variáveis não usadas

`no-unused-vars` é **erro**. Em `catch`, prefixe com `_` se realmente não for usar (`argsIgnorePattern: '^_'`).

```typescript
try {
    /* ... */
} catch (_error) {
    console.error('Operation failed');
}
```

---

## 4. Funções

### 4.1 Helpers internos vs API pública

A linha divisória é: "este símbolo é importado por outro módulo?".

| Contexto | Sintaxe | Exemplo |
|----------|---------|---------|
| Helper interno (privado ao módulo) | `const x = (): T => { ... };` (arrow + `const` + `;` após corpo) | `const getAvailableEngines = (): string => { ... };` |
| API pública da unidade | `export function x(...): T { ... }` (declaração de função) | `export function generatePackageFile(params: ScaffoldParams): FileOutput[] { ... }` |

> Veja `typescript-syntax.md` para exemplos completos extraídos do `cap-clean-arch`.

### 4.2 Máximo 30 linhas por função

`max-lines-per-function: ['warn', 30]`. É warning, não error — mas é regra. Função que passa de 30 linhas deve ser quebrada em helpers.

> Em arquivos de teste (`test/**`) o limite está desligado.

### 4.3 Retorno explícito sempre em `export function`

Toda função exportada declara o tipo de retorno explicitamente. Isso é contrato da API pública da unidade.

```typescript
export function generatePackageFile(params: ScaffoldParams): FileOutput[] {
```

### 4.4 Guard clauses no topo

Trate os casos de saída antecipada primeiro; evite aninhar o caminho feliz dentro de `if`.

```typescript
const buildScaffoldParams = (input: ScaffoldInput): ScaffoldParams => {
    if (!input.name) {
        throw new Error('Project name is required');
    }
    /* caminho principal, sem indentação extra */
};
```

---

## 5. Logging

`no-console` é **erro**, mas `console.error` é explicitamente permitido. Isso é uma **divergência intencional do MRO**: o MRO bloqueia todo `console` (força o uso de `cds.log()`); os MCPs permitem `console.error` porque servidores STDIO usam `stderr` como canal de log — `stdout` é reservado para o protocolo JSON-RPC.

| Uso | Permitido? |
|-----|-----------|
| `console.log` / `console.info` / `console.warn` / `console.debug` | ❌ Nunca |
| `console.error` | ✅ Único permitido |

❌ **Errado:** `console.log('Starting MCP server...');`
✅ **Certo:** `console.error('cap-clean-arch MCP server running on stdio');`

> Log de negócio/estado não é `console.error` — é retorno de tool no formato JSON-RPC. `console.error` é para eventos operacionais do servidor (startup, erro inesperado) que você quer rastrear.

---

## 6. Variáveis

| Regra | Detalhe |
|-------|---------|
| `const` por padrão | Use `let` só quando há reatribuição real. Nunca `var`. |
| `no-unused-vars` (erro) | Variável/import não usado quebra o build. Em `catch`, prefixe com `_` (`catch (_error)`) se realmente não for usar. |
| `prefer-const` | Variável declarada com `let` que nunca é reatribuída é erro. |
| Sem variável intermediária inútil | Não crie `const x = y; return x;` — retorne direto. |

---

## 7. Naming (código TS)

| Elemento | Convenção | Exemplo |
|----------|-----------|---------|
| Arquivo | `kebab-case.ts` | `scaffold-react-project.ts`, `tool-executor.ts` |
| Classe / Interface / Type | `PascalCase` | `ToolExecutor`, `ScaffoldParams`, `FileOutput` |
| Função / variável | `camelCase` | `generateManifest`, `targetPath` |
| Constante de módulo (dados) | `UPPER_SNAKE_CASE` | `DEFAULT_SERVICES`, `MAX_RETRIES` |
| Booleano | prefixo `is`/`has`/`should`/`can` | `isEnabled`, `hasFiles` |
| Identifiers, throws, comentários de código | **inglês** | (PT-BR só em conteúdo gerado e documentação) |

---

## 8. Export style

### 8.1 Named exports apenas — `export default` proibido

O código dos MCPs usa exclusivamente named exports. `export default` não é usado em nenhum arquivo de produção e não deve ser introduzido.

❌ **Errado:** `export default class ToolExecutor { ... }`
✅ **Certo:** `export class ToolExecutor { ... }`

### 8.2 Retorno explícito em toda função exportada

Toda `export function` declara o tipo de retorno. Isso é contrato e documentação da API da unidade.

---

## 9. Referências

- [`typescript-syntax.md`](./typescript-syntax.md) — few-shots de sintaxe extraídos de código real do `cap-clean-arch` (helpers vs API pública)
- [MRO `typescript.md`](../../../../numen-mro/docs/standards/code-style/typescript.md) — documento de referência do monorepo irmão (padrão que originou este)
- [`AGENTS.md`](../../../AGENTS.md) — porta de entrada do agente no monorepo (estrutura, stack, generators)
- [`mcps/cap-clean-arch/eslint.config.mjs`](../../../mcps/cap-clean-arch/eslint.config.mjs) — config canônica de lint
- [`mcps/cap-clean-arch/.prettierrc`](../../../mcps/cap-clean-arch/.prettierrc) — config canônica de formatação

## Escopo e próximos documentos

Este documento cobre **apenas TypeScript**. Estilo de **SQL** (CDS/HANA) e **TSX** (frontend React/MUI) serão documentos próprios nesta mesma seção, quando houver demanda — ver [`README.md`](./README.md).
