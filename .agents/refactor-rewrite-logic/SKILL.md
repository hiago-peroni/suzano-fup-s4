---
name: refactor-rewrite-logic
description: >
  Skill atomica para reescrever a logica interna de funcoes e classes para
  conformidade com a Clean Architecture. Aplica Either monad nos use cases,
  elimina framework do domain, converte models anemicos para ricos, padroniza
  error taxonomy (6+1 classes), renomeia execute() para handle() em controllers,
  e move validacao de use cases para model.validate(). Generica nos padroes CA,
  especifica nas regras de negocio (lidas do codigo existente). Use quando o
  refactor-plan classificar a acao como rewrite-logic, ou quando o usuario
  disser "reescrever logica", "converter model anemico", "padronizar error
  taxonomy", "execute para handle", "eliminar framework do domain".
  NAO use para mover arquivos entre camadas (use refactor-move-file) nem para
  criar interfaces/factories (use refactor-dependency-inversion).
metadata:
  author: Numen DS
  version: '1.0.0'
  tags: [refatoracao, clean-architecture, either-monad, error-taxonomy, rich-models]
---

# Refactor Rewrite Logic — Conformidade de Codigo

Reescreve logica interna de funcoes e classes para cumprir as 10 regras de ouro
da Clean Architecture. Foco no QUE o codigo faz estruturalmente, nao nas regras
de negocio especificas (que sao lidas do codigo existente).

> **Principio:** o Agent ja conhece clean code, SOLID e TypeScript. Esta skill
> foca nos padroes ESPECIFICOS da empresa que a IA nao deduz sozinha.

## Quando usar

- O `refactor-plan` classificou a acao como `rewrite-logic`
- Tasks tipo: padronizar error taxonomy, eliminar framework do domain, converter
  models anemicos → ricos, controllers execute() → handle()

## Escopo

Atua sobre:
- Use cases em `application/use-cases/`
- Models em `domain/models/`
- Controllers em `presentation/controllers/`
- Errors em `domain/errors/`
- Services em `application/services/` ou `domain/services/`

Nao atua sobre:
- Movimento de arquivos entre pastas (papel do `refactor-move-file`)
- Criacao de interfaces + factories (papel do `refactor-dependency-inversion`)

## Regras estruturais (genericas — mesmas em todo cliente)

### R1 — Either e o contrato de use case

```
// CORRETO — use case retorna Either
async execute(params: XxxParams): Promise<Either<AbstractError, XxxResult>> {
  return ok(result) ou err(new BadRequestError(...))
}

// ERRADO — use case lanca excecao
async execute(params): Promise<XxxResult> {
  if (invalid) throw new Error(...)  // ❌ infrastructure lanca; use case retorna Either
}
```

| Camada | Retorno | Excecao |
|--------|---------|---------|
| Use case (`application/`) | `Either<AbstractError, T>` | NUNCA lanca |
| Infrastructure (`infrastructure/`) | `T` direto | Propaga excecao |
| Controller (`presentation/`) | Traduz Either → resposta HTTP | NUNCA lanca |

### R2 — Sem framework no domain

A UNICA dependencia permitida em `domain/` e `@sweet-monads/either`.

| Proibido em domain/ | Permitido |
|---------------------|-----------|
| `@sap/cds` | `@sweet-monads/either` |
| `@sap/cds-odata-parser` | Tipos puros de TS |
| `express` | |
| Qualquer framework | |

### R3 — Models ricos, nunca anemicos

```typescript
// ANEMICO (ERRADO):
class XxxModel {
  constructor(public readonly field1: string) {}
}

// RICO (CORRETO):
class XxxModel {
  private constructor(private readonly props: XxxProps) {}

  public static with(props: XxxProps): Either<AbstractError, XxxModel> {
    // validacao aqui
    if (!props.field1) return err(new BadRequestError('field1 required'))
    return ok(new XxxModel(props))
  }

  public get field1(): string { return this.props.field1 }
  public validate(): Either<AbstractError, void> { ... }
}
```

| Modelo anemico | Modelo rico |
|----------------|-------------|
| Constructor publico | Constructor `private` |
| Sem validacao | `static with(props)` com validacao |
| Getters diretos | `validate()` metodo |
| Setters mutaveis | Props `readonly` |

### R4 — Error taxonomy (6+1 classes)

| Classe | HTTP | Quando usar |
|--------|------|------------|
| `BadRequestError` | 400 | Input invalido |
| `UnauthorizedError` | 401 | Nao autenticado |
| `ForbiddenError` | 403 | Sem permissao |
| `NotFoundError` | 404 | Recurso nao existe |
| `ConflictError` | 409 | Conflito de estado |
| `ServerError` | 500 | Erro interno |
| `UnavailableServiceError` | 503 | API externa indisponivel (opcional) |

Proibido: `BadGatewayError` (502), `UnprocessableEntity` (422) — dead code.

### R5 — Controllers usam handle(), nao execute()

```typescript
// ERRADO:
class XxxController {
  async execute(eventContext) { ... }
}

// CORRETO:
class XxxController {
  async handle(eventContext): Promise<BaseControllerResponse> { ... }
}
```

### R6 — console.error apenas

`stdout` e reservado para CAP/JSON-RPC. Logs de erro em `console.error`.

### R7 — Imports NodeNext exigem .js

```typescript
import { XxxRepository } from './xxx-repository.js'  // .js obrigatorio
```

## Regras de negocio (especificas — lidas do codigo)

| O que ler | Como |
|-----------|------|
| Logica atual da funcao | `Read` do arquivo alvo |
| Como a funcao e chamada | `Grep` pelo nome da funcao |
| Tipos/entidades do dominio | `Read` dos models/interfaces existentes |
| Regras de validacao | `Read` do use case que sera refatorado |

## Workflow

### Passo 1 — Carregar contexto

```
Read("{project-root}/AGENTS.md")
Read("{project-root}/.refactor-plan.md")
Read("{arquivo-alvo}")  # o arquivo a reescrever
```

### Passo 2 — Diagnosticar nao-conformidades

Ler o arquivo e classificar cada problema:

| Problema | Regra violada | Acao |
|----------|---------------|------|
| Use case lanca excecao | R1 | Converter para `Either` |
| Domain importa `@sap/cds` | R2 | Extrair para `infrastructure/` |
| Model sem `static with()` | R3 | Adicionar factory + validacao |
| Erro usa `Error` generico | R4 | Trocar pela classe canonica |
| Controller tem `execute()` | R5 | Renomear para `handle()` |
| `console.log` em producao | R6 | Trocar para `console.error` ou remover |
| Import sem `.js` | R7 | Adicionar extensao |

### Passo 3 — Reescrever

Para cada nao-conformidade, aplicar a correcao:

1. **Converter para Either (R1):**
   - Identificar todos os pontos de `throw` no use case
   - Substituir `throw new XxxError(msg)` por `return err(new XxxError(msg))`
   - Envolver retorno de sucesso em `return ok(result)`
   - Mudar tipo de retorno para `Promise<Either<AbstractError, T>>`

2. **Eliminar framework do domain (R2):**
   - Identificar imports de framework em `domain/`
   - Mover a logica que depende do framework para `infrastructure/`
   - Criar interface no `domain/` se necessario (delega para `refactor-dependency-inversion`)

3. **Converter model anemico → rico (R3):**
   - Tornar constructor `private`
   - Adicionar `static with(props): Either<AbstractError, Model>`
   - Mover validacao do use case para `with()` ou `validate()`
   - Adicionar getters para propriedades

4. **Padronizar error taxonomy (R4):**
   - `Grep` por cada classe nao-canonica (`Error`, `BadGatewayError`, etc.)
   - Substituir pela classe canonica mais proxima
   - Verificar que todas as 6+1 classes existem em `domain/errors/`

5. **Renomear execute() → handle() (R5):**
   - `Grep` por `execute(` em `presentation/controllers/`
   - Renomear para `handle(`
   - Atualizar callers (grep por `.execute(` no mesmo service)

### Passo 4 — Verificar

```
cd "{service-dir}" && yarn lint && yarn tsc --noEmit
```

Se falhar:
- Classificar erro (tipo incompatible, import quebrado, naming)
- Corrigir
- Re-rodar

### Passo 5 — Retornar

Retornar para o `refactor-orchestrator`:
- Status: `done` | `partial`
- Nao-conformidades corrigidas: [{regra} → {arquivo}:{linha}]
- Lint/typecheck: pass | fail

## Gotchas

| # | Armadilha | Como evitar |
|---|-----------|-------------|
| 1 | `Either` nao importado | Verificar `import { Either } from '@sweet-monads/either'` apos conversao |
| 2 | `static with()` esquece retornar `Either` | O factory DEVE retornar `Either<AbstractError, Model>`, nao `Model` |
| 3 | `execute()` renomeado mas caller nao atualizado | Grep por `.execute(` apos renomear — atualizar TODOS os callers |
| 4 | Error generico `new Error()` em use case | Substituir por classe canonica da taxonomy |
| 5 | Domain ainda importa tipo de framework via type-only | Mesmo `import type` de framework e violacao — extrair para interface |
| 6 | Validacao movida para model mas use case nao atualizado | Apos mover validacao, o use case deve chamar `model.validate()` |

## Tratamento de Erros (LangGraph)

| Estado | Acao |
|--------|------|
| Type error apos conversao para Either | Verificar que todos os `return` agora retornam `Either` |
| Model quebrado apos `static with()` | Verificar que todos os callers usam `XxxModel.with(props)` |
| `handle()` nao reconhecido pelo CAP | Verificar que o binding em `routes/index.cds` referencia `handle` |
| Lint falha por `any` | Substituir por tipo concreto do dominio |

## Referencias

- `docs/obsidian/Obsidian/IAtizacao/Clean Architecture.md` — 10 regras de ouro, error taxonomy
- `docs/obsidian/Obsidian/IAtizacao/Either Monad.md` — tratamento de erros com `@sweet-monads/either`
- `docs/obsidian/Obsidian/IAtizacao/Code Style.md` — estilo TypeScript canonico
- `{project-root}/AGENTS.md` — convens especificas do projeto
