# factories/adapters/

Instanciam implementações de infraestrutura que satisfazem interfaces definidas em `domain/adapters/*`.

## Regras

- O tipo de retorno **sempre** é a interface do domínio, nunca a implementação concreta.
- Adapters com estado compartilhado usam singleton lazy (`let instance = null`).
- Adapters stateless retornam `new Impl()` diretamente, sem singleton.
- A decisão de qual implementação usar (prod vs dev) fica aqui, nunca em outro lugar.

---

## Shape 1 — Singleton simples

Use quando o adapter é stateful e deve ser compartilhado (ex.: conexão de banco, cliente HTTP).

```typescript
// src/main/factories/adapters/hdb-adapter.ts
import type { HdbAdapter } from '@/domain/adapters/hdb-adapter';
import { SapHdbAdapter } from '@/infrastructure/adapters/hdb-adapter';

let instance: HdbAdapter | null = null;

export const makeHdbAdapter = (): HdbAdapter => {
    if (!instance) {
        instance = new SapHdbAdapter();
    }
    return instance;
};
```

---

## Shape 2 — Singleton com switch de ambiente

Use quando o adapter tem uma implementação real (produção) e uma local (dev).

```typescript
// src/main/factories/adapters/object-store.ts
import type { ObjectStoreAdapter } from '@/domain/adapters/object-store';
import { BtpObjectStore } from '@/infrastructure/adapters/object-store/btp-object-store';
import { LocalObjectStore } from '@/infrastructure/adapters/object-store/local-object-store';

let instance: ObjectStoreAdapter | null = null;

export const makeObjectStoreAdapter = (): ObjectStoreAdapter => {
    if (!instance) {
        instance = process.env.NODE_ENV === 'dev'
            ? new LocalObjectStore()
            : new BtpObjectStore();
    }
    return instance;
};
```

---

## Shape 3 — Singleton com lifecycle (shutdown hooks)

Use quando o adapter gerencia recursos que precisam ser liberados ao encerrar o processo (ex.: telemetria com buffer, pool de conexões).

```typescript
// src/main/factories/adapters/telemetry.ts
import type { Telemetry } from '@/domain/adapters/telemetry';
import { CompositeTelemetry } from '@/infrastructure/adapters/telemetry/composite-telemetry';
import { ConsoleSink } from '@/infrastructure/adapters/telemetry/console-sink';
import { DbSink } from '@/infrastructure/adapters/telemetry/db-sink';
import { ObservabilityChannelsResolver } from '@/infrastructure/adapters/telemetry/observability-channels-resolver';

let instance: CompositeTelemetry | null = null;
let shutdownHooked: boolean = false;

export const makeTelemetry = (): Telemetry => {
    if (!instance) {
        const resolver = new ObservabilityChannelsResolver();
        const sinks = [new ConsoleSink(), new DbSink(resolver)];
        instance = new CompositeTelemetry(sinks);
        if (!shutdownHooked) {
            registerShutdownHooks();
            shutdownHooked = true;
        }
    }
    return instance;
};

const registerShutdownHooks = (): void => {
    const shutdown = async (): Promise<void> => {
        if (instance) {
            await instance.shutdown();
        }
    };
    process.on('SIGTERM', shutdown);
    process.on('SIGINT', shutdown);
};
```

---

## Shape 4 — Stateless (nova instância por chamada)

Use quando o adapter não guarda estado entre chamadas (ex.: parser de arquivo, validador, serializador).

```typescript
// src/main/factories/adapters/spreadsheet-parser.ts
import type { SpreadsheetParser } from '@/domain/adapters/parsers';
import { ExcelSpreadsheetParser } from '@/infrastructure/adapters/parsers';

export const makeSpreadsheetParser = (): SpreadsheetParser => new ExcelSpreadsheetParser();
```

---

## Quando usar cada shape

| Situação | Shape |
|---|---|
| Adapter simples, sem impl alternativa | 1 — Singleton simples |
| Adapter com impl local para dev | 2 — Singleton com switch de ambiente |
| Adapter com recursos a liberar no shutdown | 3 — Singleton com lifecycle |
| Adapter sem estado, leve, seguro para múltiplas instâncias | 4 — Stateless |
