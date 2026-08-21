# Code Style — MCPs Numen DS

Seção de **estilo e qualidade de código** dos servidores MCP do monorepo `numen-mcps`. Complementa o [`AGENTS.md`](../../AGENTS.md): enquanto aquele define **estrutura** (layout de diretórios, stack, generators, tool-executor), esta seção define **como o código é escrito** (formatação, imports, tipagem, naming, estilo de export).

> A fonte de verdade é a ferramenta (`eslint.config.mjs`, `.prettierrc`, `tsconfig.json`). Estes documentos explicam e exemplificam as regras para humanos e agentes.

## Documentos desta seção

| Documento | Escopo | Status |
|-----------|--------|--------|
| [**typescript.md**](./typescript.md) | Código TypeScript (formatação, imports, tipagem, funções, logging, naming, export style) | ✅ Disponível |
| [**typescript-syntax.md**](./typescript-syntax.md) | Few-shots de sintaxe TS extraídos de código real (helpers vs API pública) | ✅ Disponível |
| `sql.md` | CDS / HANA SQL | ⏳ Planejado |
| `tsx.md` | Frontend React 19 + MUI 7 (TSX) | ⏳ Planejado |

## Quando ler

| Situação | Documento |
|----------|-----------|
| Antes de escrever/editar qualquer `.ts` nos MCPs | [typescript.md](./typescript.md) |
| Dúvida sobre sintaxe de helpers vs export function | [typescript-syntax.md](./typescript-syntax.md) |
| Review de PR (checklist de código TS) | [typescript.md](./typescript.md) §9 |

## Referências cruzadas

- [`AGENTS.md`](../../AGENTS.md) — porta de entrada do agente no monorepo
- [MRO `code-style/typescript.md`](../../../numen-mro/docs/standards/code-style/typescript.md) — documento de referência do monorepo irmão
