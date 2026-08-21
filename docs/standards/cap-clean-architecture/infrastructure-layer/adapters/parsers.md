# Parsers de arquivo

Parsers transformam dados externos — arquivos, streams, buffers — em estruturas de domínio. São classes stateless (ou funções puras) que implementam uma interface de `domain/adapters/parsers/`.

## Estrutura

```
src/infrastructure/adapters/parsers/
├── excel-spreadsheet-parser.ts    → ExcelSpreadsheetParser
├── csv-parser.ts                  → CsvParser
└── demands-parser.ts              → DemandsParserImpl
```

## Tipos comuns de parser

| Formato | Lib recomendada | Observação |
|---|---|---|
| Excel (`.xlsx`) | `exceljs` (streaming) | `WorkbookReader` para arquivos grandes; evitar `readFile` síncrono |
| CSV | `csv-parse` (stream API) | Pipe com `Transform` para mapeamento linha a linha |
| JSON estruturado | `zod` (validação + parse) | Schema + `z.array(...).parse(raw)` em uma passagem |
| JSON sem schema | `JSON.parse` nativo | Aceitável apenas para payloads pequenos e bem conhecidos |

## Naming

`<Formato>Parser` — sem sufixo `Impl`. A classe já é concreta por natureza (não há variação real/fake; fakes de parser usam fixture de arquivo ou buffer).

## Few-shot 1 — `ExcelSpreadsheetParser` (streaming via ExcelJS)

```typescript
// src/infrastructure/adapters/parsers/excel-spreadsheet-parser.ts
import ExcelJS from 'exceljs';

import type { SpreadsheetParser } from '@/domain/adapters/parsers/spreadsheet-parser.js';

export class ExcelSpreadsheetParser implements SpreadsheetParser {
    public async *parseStream(filePath: string): AsyncGenerator<SpreadsheetParser.ParsedRow> {
        const workbookReader = new ExcelJS.stream.xlsx.WorkbookReader(filePath, {
            sheets: 'emit',
            sharedStrings: 'cache',
            hyperlinks: 'ignore',
            styles: 'ignore',
            entries: 'emit'
        });

        let isFirstRow = true;
        let headers: string[] = [];

        for await (const worksheetReader of workbookReader) {
            for await (const row of worksheetReader) {
                if (isFirstRow) {
                    headers = (row.values as (string | undefined)[])
                        .slice(1)
                        .map((v) => String(v ?? '').trim());
                    isFirstRow = false;
                    continue;
                }

                const values = (row.values as unknown[]).slice(1);
                const parsedRow: SpreadsheetParser.ParsedRow = {};
                headers.forEach((header, idx) => {
                    parsedRow[header] = values[idx] ?? null;
                });

                yield parsedRow;
            }
        }
    }
}
```

O uso de `AsyncGenerator` permite que o use case processe cada linha conforme chega, sem carregar o arquivo inteiro em memória. O caller itera com `for await (const row of parser.parseStream(path))`.

## Few-shot 2 — `DemandsParserImpl` (JSON com validação Zod)

```typescript
// src/infrastructure/adapters/parsers/demands-parser.ts
import { z } from 'zod';
import * as fs from 'fs';

import { DemandsParser } from '@/domain/adapters/parsers/demands-parser.js';
import { DemandModel } from '@/domain/models/demand.js';

export class DemandsParserImpl implements DemandsParser {
    private readonly schema = z.object({
        materialNumber: z.string(),
        plant: z.string(),
        quantity: z.number().positive()
    });

    public async parse(filePath: string): Promise<DemandModel[]> {
        const raw = fs.readFileSync(filePath, 'utf-8');
        const parsed = JSON.parse(raw);
        const validated = this.schema.array().parse(parsed);
        return validated.map(DemandModel.with);
    }
}
```

`this.schema.array().parse(parsed)` lança `ZodError` se qualquer item falhar na validação — o use case captura e trata via `handleError`. O `schema` Zod fica como `private readonly` da classe (regra global de [tipagem e constantes](../README.md#tipagem-e-constantes)), não em module-level.

## Tipagem e constantes

Aplicar as 3 regras globais da [seção "Tipagem e constantes" do README da infrastructure-layer](../README.md#tipagem-e-constantes). Para parsers especificamente:

- `XxxParser.ParsedRow`, `XxxParser.Params` e tipos auxiliares vivem no namespace da interface em `domain/adapters/parsers/<formato>-parser.ts`.
- Schemas Zod ficam como `private readonly schema = z.object({...})` da classe — nunca como `const` em module-level.
- O resultado validado pelo Zod é mapeado para `DomainModel` via `DomainModel.with(validated)`.

## Regras

1. **Sem DI por padrão.** Caso o parser precise de dependências externas (telemetria, armazenamento temporário), receber via constructor tipado pelas interfaces do domínio.

2. **Parsers propagam erros.** `JSON.parse` lança `SyntaxError`, `ZodError` é lançado pelo Zod, ExcelJS lança em arquivos corrompidos. Não envolver em `try/catch` interno — o use case trata via `handleError`.

3. **Streaming para arquivos grandes.** Usar ExcelJS `WorkbookReader` ou `csv-parse` stream API para arquivos acima de 10 MB. `fs.readFileSync` é aceitável apenas para payloads JSON pequenos e bem delimitados.

4. **Nome: `<Formato>Parser`, sem `Impl`.** A ausência de variação real/fake torna o sufixo `Impl` redundante — alinhado com a mesma convenção de `*UnitOfWork`.

5. **Schemas Zod como `private readonly` da classe.** Nunca `const xxxSchema = z.object(...)` em module-level — viola a regra global de constantes da infrastructure.
