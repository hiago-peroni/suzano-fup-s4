---
name: refactor-move-file
description: >
  Skill atomica para mover arquivos entre camadas da Clean Architecture ou eliminar
  pastas proibidas. Aplica a dependency rule, atualiza todos os imports que
  referenciam o path antigo, corrige sufixos por camada e elimina barrel exports.
  Generica nas regras de negocio (lidas do AGENTS.md do projeto), precisa nas
  regras estruturais (qual pasta, naming, imports). Use quando o refactor-plan
  classificar a acao como move-file, ou quando o usuario disser "mover arquivo",
  "eliminar pasta proibida", "eliminar barrel exports", "rename persistence para
  repositories". NAO use para reescrever logica interna (use refactor-rewrite-logic)
  nem para inverter dependencias (use refactor-dependency-inversion).
metadata:
  author: Numen DS
  version: '1.0.0'
  tags: [refatoracao, clean-architecture, move-file, barrel-exports]
---

# Refactor Move File — Realocacao entre Camadas

Move arquivos para a camada correta da Clean Architecture, elimina pastas proibidas
e remove barrel exports. Atualiza todos os imports referenciando o path antigo.

> **Regra absoluta:** a dependency rule nunca pode ser violada ao mover. Se mover
> um arquivo cria uma dependencia que aponta para fora, o arquivo NAO deve ser
> movido — ele pertence a camada externa.

## Quando usar

- O `refactor-plan` classificou a acao como `move-file`
- O usuario pede "mover X para Y", "eliminar pasta Z", "remover barrel exports"
- Tasks tipo: rename persistence→repositories, eliminar pastas proibidas, eliminar
  barrel exports (53 index.ts)

## Escopo

Atua sobre:
- Arquivos TypeScript/CDS dentro de `packages/backend/<service>/src/` ou equivalente
- Estrutura de camadas: `domain/`, `application/`, `infrastructure/`, `presentation/`, `main/`

Nao atua sobre:
- Logica interna de funcoes (papel do `refactor-rewrite-logic`)
- Criacao de interfaces/factories (papel do `refactor-dependency-inversion`)

## Regras estruturais (genericas — mesmas em todo cliente)

### Camadas e sufixos

| Camada | O que vive aqui | Sufixo obrigatorio |
|--------|----------------|-------------------|
| `domain/` | Contratos, models ricos, errors | `Model`, `Error` |
| `application/` | Use cases, services auxiliares | `UseCaseImpl`, `ServiceImpl` |
| `infrastructure/` | Repositories, adapters, hydrators, UoW | `Impl`, `ApiImpl` |
| `presentation/` | Controllers | `Controller` |
| `main/` | Factories, routes, config, scripts | `makeXxx()` |

### Dependency rule

```
main → presentation → domain
main → application   → domain
main → infrastructure → domain
```

Setas SO apontam para `domain/`. Inverter = violacao. Se ao mover um arquivo
ele passa a importar de uma camada mais externa, o movimento e INVALIDO.

### Pastas proibidas (devem ser eliminadas)

| Pasta | Para onde mover o conteudo |
|------|---------------------------|
| `extractors/` | `infrastructure/parsers/` |
| `formatters/` | `infrastructure/parsers/` ou `domain/utils/` |
| `hidrators/` | `infrastructure/hydrators/` (com "y") |
| `i18n/` | `main/config/` ou `shared/` |
| `main/adapters/` | `domain/adapters/` (interfaces) ou `infrastructure/adapters/` (impl) |
| `persistence/` | `infrastructure/repositories/` |

### Barrel exports

- Proibido `index.ts` que so faz re-export (exceto `domain/errors/index.ts`)
- Ao eliminar barrel: atualizar TODOS os imports do `index.ts` para import direto
  do arquivo de origem

### Imports

- NodeNext exige `.js` na extensao do import
- Sem alias relativo cruzando camadas (ex: `domain/` nao importa de `infrastructure/`)
- Alias `@/` ou module-alias: seguir o que esta no `AGENTS.md` do projeto

## Regras de negocio (especificas — lidas do projeto)

| O que ler | Onde | Por que |
|-----------|------|---------|
| Stack exata e versoes | `AGENTS.md` do projeto | TS version, CAP version |
| Naming conventions | `AGENTS.md` do projeto | PascalCase, snake_case, etc |
| Module aliases | `tsconfig.json` ou `AGENTS.md` | `@domain/`, `@infra/`, etc |
| Estrutura de servicos | `AGENTS.md` + `ls packages/` | Quantos servicos, nomes |
| Convens de import | `AGENTS.md` + `eslint` | Ordem, estilo |

## Workflow

### Passo 1 — Carregar contexto

```
Read("{project-root}/AGENTS.md")
Read("{project-root}/.refactor-plan.md")  # se existir (do refactor-plan)
```

### Passo 2 — Identificar arquivos afetados

```
Glob("{path-alvo}/**/*.ts")  # arquivos a mover
Grep("{path-antigo}", include="*.ts")  # arquivos que importam do path antigo
```

### Passo 3 — Mover cada arquivo

Para cada arquivo da lista:

1. **Verificar dependency rule:** o arquivo importa de algo mais externo?
   - Se SIM: o arquivo pertence a camada externa — nao mover para mais interna
   - Se NAO: seguro mover

2. **Mover o arquivo:**
   ```
   Read("{path-antivo}/{arquivo}")
   Write("{path-novo}/{arquivo}", conteudo)
   # deletar o antigo via bash:
   rm "{path-antigo}/{arquivo}"
   ```

3. **Corrigir sufixo da classe/export** se a camada exigir:
   - Ex: `class UserRepository` em `infrastructure/` → `class UserRepositoryImpl`

4. **Atualizar imports que referenciam o path antigo:**
   ```
   Grep("{path-antigo}", include="*.ts")
   # para cada match: Edit substituindo path antigo por novo
   ```

5. **Eliminar barrel export** se o arquivo era re-exportado por `index.ts`:
   - Atualizar imports de `from "./index"` para `from "./arquivo"`

### Passo 4 — Eliminar pastas proibidas (se aplicavel)

Se a task envolver eliminar pasta proibida:

| Pasta | Acao |
|------|------|
| `extractors/` | Mover conteudo para `infrastructure/parsers/`, atualizar imports |
| `formatters/` | Mover para `infrastructure/parsers/` ou `domain/utils/` |
| `hidrators/` | Mover para `infrastructure/hydrators/` (com "y") |
| `i18n/` | Mover para `main/config/` ou `shared/` |
| `persistence/` | Mover para `infrastructure/repositories/` |

Apos mover todo conteudo, deletar a pasta vazia:
```
rmdir "{pasta-proibida}"
```

### Passo 5 — Verificar

```
# Verificar que nao sobrou import do path antigo
Grep("{path-antigo}", include="*.ts")  # deve retornar vazio

# Lint + typecheck
cd "{service-dir}" && yarn lint && yarn tsc --noEmit
```

Se lint OU typecheck falhar:
- Classificar o erro (import quebrado, tipo incompatible, naming)
- Corrigir
- Re-rodar

### Passo 6 — Retornar

Retornar para o `refactor-orchestrator`:
- Status: `done` | `partial` (com lista do que falta)
- Arquivos movidos: [{path-antigo} → {path-novo}]
- Imports atualizados: {quantidade}
- Pastas eliminadas: {lista}
- Lint/typecheck: pass | fail

## Gotchas

| # | Armadilha | Como evitar |
|---|-----------|-------------|
| 1 | Import esquecido apos mover | Rodar `Grep` pelo path antigo apos TODOS os movimentos |
| 2 | Barrel export escondido em subpasta | Verificar `index.ts` em cada nivel da pasta movida |
| 3 | `.cds` files tambem precisam atualizar | Se mover arquivos CDS, atualizar `@cds` imports tambem |
| 4 | Module alias nao atualizado | Se `tsconfig.json` tem `paths` apontando para o path antigo, atualizar |
| 5 | Sufixo da classe nao corrigido |apos mover, verificar se a classe precisa de sufixo da nova camada |
| 6 | Pasta nao fica vazia apos mover | Verificar arquivos ocultos (`.gitkeep`, `index.ts`) antes de deletar |

## Tratamento de Erros (LangGraph)

| Estado | Acao |
|--------|------|
| Import quebrado apos mover | Grep pelo path antigo + Edit para corrigir |
| Type error por tipo incompatible | A move nao deve mudar tipos — se mudou, e bug de logica, escalar |
| Dependency rule violada ao mover | O arquivo NAO deve ser movido para mais interna — mover para a externa correta |
| Lint falha por naming | Corrigir sufixo da classe conforme tabela de camadas |

## Referencias

- `docs/obsidian/Obsidian/IAtizacao/Clean Architecture.md` — 5 camadas, 10 regras de ouro, pastas proibidas
- `docs/obsidian/Obsidian/IAtizacao/Code Style.md` — estilo TypeScript canonico
- `docs/obsidian/Obsidian/IAtizacao/Padrao Monorepo.md` — estrutura packages/
- `{project-root}/AGENTS.md` — convens especificas do projeto
