---
name: refactor-dependency-inversion
description: >
  Skill atomica para inverter dependencias criando interface no domain,
  implementacao no infrastructure e factory com DI manual no main. Aplica
  o padrao Factory (makeXxx(): Interface) com constructor tipado por
  private readonly, sem container/IOC. Generica no padrao DI, especifica
  nas interfaces do dominio (lidas do codigo). Use quando o refactor-plan
  classificar a acao como dependency-inversion, ou quando o usuario disser
  "inverter dependencia", "criar interface", "criar factory", "aplicar DI",
  "extrair adapter". NAO use para mover arquivos (use refactor-move-file) nem
  para reescrever logica interna (use refactor-rewrite-logic).
metadata:
  author: Numen DS
  version: '1.0.0'
  tags: [refatoracao, clean-architecture, dependency-inversion, factory, di]
---

# Refactor Dependency Inversion — DI + Factory

Cria a triade interface → implementacao → factory para inverter dependencias.
A camada externa (`infrastructure/`) depende da interna (`domain/`), nunca o
inverso. O `main/` faz o wiring manual via factory.

> **Dependency rule absoluta:** se uma classe em `domain/` precisa acessar
> algo externo (DB, API, filesystem), cria-se uma INTERFACE em `domain/` e a
> IMPLEMENTACAO em `infrastructure/`. O domain so conhece a interface.

## Quando usar

- O `refactor-plan` classificou a acao como `dependency-inversion`
- Tasks tipo: extrair adapter, criar interface de repositorio, aplicar DI
- Quando um use case em `application/` acessa diretamente `infrastructure/`

## Escopo

Atua sobre:
- `domain/repositories/` — criar interface
- `domain/adapters/` — criar interface de adapter
- `domain/services/` — criar interface de service
- `infrastructure/repositories/` — criar implementacao
- `infrastructure/adapters/` — criar implementacao
- `main/factories/` — criar factory com DI

Nao atua sobre:
- Logica interna de use cases (papel do `refactor-rewrite-logic`)
- Movimento de arquivos entre pastas (papel do `refactor-move-file`)

## Regras estruturais (genericas — mesmas em todo cliente)

### Padrao Factory (DI manual)

```typescript
// 1. INTERFACE no domain (contrato)
// domain/repositories/xxx-repository.ts
export interface XxxRepository {
  findById(id: string): Promise<Either<AbstractError, XxxModel>>
  save(model: XxxModel): Promise<Either<AbstractError, void>>
}

export namespace XxxRepository {
  export interface DbRow { ... }  // tipos auxiliares no namespace
}

// 2. IMPLEMENTACAO no infrastructure
// infrastructure/repositories/xxx-repository-impl.ts
export class XxxRepositoryImpl implements XxxRepository {
  constructor(private readonly db: DbConnection) {}  // DI via constructor

  async findById(id: string): Promise<Either<AbstractError, XxxModel>> { ... }
}

// 3. FACTORY no main (composition root)
// main/factories/xxx-repository-factory.ts
export function makeXxxRepository(): XxxRepository {
  const db = getDbConnection()
  return new XxxRepositoryImpl(db)
}
```

### Regras do DI

| Regra | Detalhe |
|-------|--------|
| Interface em `domain/` | O contrato que use cases conhecem |
| Impl em `infrastructure/` | A classe concreta que fala com DB/API |
| Factory em `main/factories/` | Funcao `makeXxx(): Interface` que instancia impl |
| Constructor com `private readonly` | Tipado pela INTERFACE, nao pela impl |
| Sem container/IOC | DI manual — sem Inversify, tsyringe, etc |
| Tipos auxiliares no namespace | `XxxRepository.DbRow` vive no namespace da interface |

### Tipos de inversao

| Tipo | Interface em | Impl em | Quando |
|------|---------------|---------|--------|
| Repository | `domain/repositories/` | `infrastructure/repositories/` | Acesso a DB (CDS/HANA) |
| Adapter (API) | `domain/adapters/external-api/` | `infrastructure/adapters/external-api/` | Chamada a API externa |
| Adapter (parser) | `domain/adapters/parsers/` | `infrastructure/adapters/parsers/` | Parse de formato |
| Service | `domain/services/` | `application/services/` | Logica de dominio compartilhada |

## Regras de negocio (especificas — lidas do codigo)

| O que ler | Como |
|-----------|------|
| Metodos que o use case chama | `Read` do use case que precisa da dependencia |
| Tipos de retorno esperados | `Read` dos models e types existentes |
| Como o DB e acessado hoje | `Grep` por `@sap/cds` ou `SELECT` no service |
| Factory existente como referencia | `Glob` por `main/factories/**/*.ts` |

## Workflow

### Passo 1 — Carregar contexto

```
Read("{project-root}/AGENTS.md")
Read("{project-root}/.refactor-plan.md")
Read("{arquivo que precisa da dependencia}")  # ex: use case que acessa DB direto
```

### Passo 2 — Identificar a dependencia a inverter

Grep no arquivo alvo por:
- `@sap/cds` (acesso direto ao CDS)
- `SELECT` ou `INSERT` em strings (SQL direto)
- `import` de `infrastructure/` em `application/` ou `domain/`
- Chamadas a APIs externas (axios, fetch, http)

Para cada dependencia encontrada, classificar:

| Dependencia | Tipo | Interface a criar |
|-------------|------|-------------------|
| Acesso a DB via CDS | Repository | `XxxRepository` |
| Chamada a API externa | Adapter (API) | `XxxApi` |
| Parse de dados | Adapter (parser) | `XxxParser` |
| Logica de dominio compartilhada | Service | `XxxService` |

### Passo 3 — Criar a interface no domain

```
Write("{domain-path}/xxx-repository.ts", interfaceContent)
```

A interface deve:
- Declarar todos os metodos que o use case precisa
- Retornar `Either<AbstractError, T>` (se for use case chamando)
- Ter namespace com tipos auxiliares se necessario

### Passo 4 — Criar a implementacao no infrastructure

```
Write("{infrastructure-path}/xxx-repository-impl.ts", implContent)
```

A impl deve:
- `implements XxxRepository`
- Receber dependencias via `constructor(private readonly ...)`
- Conter a logica de DB/API que antes estava no use case

### Passo 5 — Criar a factory no main

```
Write("{main-path}/factories/xxx-repository-factory.ts", factoryContent)
```

A factory deve:
- `export function makeXxxRepository(): XxxRepository`
- Instanciar a impl com suas dependencias
- Retornar tipado pela interface

### Passo 6 — Atualizar o use case para usar a interface

```
Edit("{use-case-path}", substituir acesso direto por chamada via interface)
```

O use case agora:
- Recebe `XxxRepository` via `constructor(private readonly xxxRepository: XxxRepository)`
- Chama `this.xxxRepository.findById(id)` em vez de `cds.db.xxx.select(...)`
- Nao importa nada de `infrastructure/`

### Passo 7 — Atualizar o wiring no main/routes

```
Edit("{main-path}/routes/index.ts", usar a factory para instanciar)
```

### Passo 8 — Verificar

```
cd "{service-dir}" && yarn lint && yarn tsc --noEmit
```

Se falhar:
- Verificar que a interface e a impl tem a mesma assinatura
- Verificar que o use case so importa da interface
- Verificar que a factory retorna o tipo da interface

### Passo 9 — Retornar

Retornar para o `refactor-orchestrator`:
- Status: `done` | `partial`
- Interface criada: `{path}`
- Impl criada: `{path}`
- Factory criada: `{path}`
- Use case atualizado: `{path}`
- Lint/typecheck: pass | fail

## Gotchas

| # | Armadilha | Como evitar |
|---|-----------|-------------|
| 1 | Interface nao retorna Either | Repository chamado por use case DEVE retornar `Either<AbstractError, T>` |
| 2 | Impl importa de domain indevidamente | Impl pode importar de domain (dependencia valida) mas nao de application |
| 3 | Factory esquece injetar dependencia | Verificar que todos os params do constructor da impl estao na factory |
| 4 | Use case ainda importa infrastructure | Apos inversao, o use case so pode importar de `domain/` |
| 5 | Namespace esquecido | Tipos auxiliares como `DbRow` devem estar no namespace da interface |
| 6 | Factory nao tipada pela interface | `return new XxxImpl(db)` retorna a impl; o tipo de retorno deve ser a interface |

## Tratamento de Erros (LangGraph)

| Estado | Acao |
|--------|------|
| Use case nao compila apos inversao | Verificar que a interface tem todos os metodos que o use case chama |
| Impl quebra por falta de contexto CDS | Passar `cds` ou `DbConnection` via constructor da impl |
| Factory retorna tipo errado | Anotar retorno da factory com o tipo da interface |
| Dependency circular detectada | A interface NAO pode importar da impl — verificar imports |

## Referencias

- `docs/obsidian/Obsidian/IAtizacao/Clean Architecture.md` — DI manual, factory pattern, dependency rule
- `docs/obsidian/Obsidian/IAtizacao/Either Monad.md` — Either como contrato de use case
- `{project-root}/AGENTS.md` — convens especificas do projeto
