# Application layer

A application layer **implementa os casos de uso** declarados no domain. Ela recebe as interfaces de repositório injetadas, orquestra chamadas e devolve `Either<AbstractError, T>`. Nenhum detalhe de infraestrutura (fetch, HTTP, localStorage) vive aqui — a camada só conhece interfaces de domínio.

> **Equivalência com o CAP:** a `application/` corresponde à `application/` do backend CAP — nomenclatura idêntica.

> **Regra de isolamento:** a application layer **não importa nada** de `infra/`, `presentation/`, `main/` ou `shared/`. Importa apenas de `domain/`.

## Estrutura

```
src/application/
└── use-cases/
    └── <feature>/
        └── <kebab-name>-use-case.ts    → class XxxUseCase implements Xxx
```

Os arquivos espelham exatamente a estrutura de `domain/use-cases/`: mesma pasta `<feature>/`, mesmo nome de arquivo com sufixo `-use-case`.

## Responsabilidades

| Subpasta | Responsabilidade |
|---|---|
| `use-cases/<feature>/` | Implementações concretas dos contratos de `domain/use-cases/` |

## Regras de ouro

1. **Nenhuma chamada de infraestrutura diretamente.** HTTP, banco, localStorage — tudo via interfaces de repositório injetadas no constructor.
2. **`execute` é o único método público.** A assinatura espelha exatamente a da interface de domínio correspondente.
3. **`Result` é sempre `Either<AbstractError, T>`.** Caminho feliz retorna `right(valor)`; erros retornam `left(new XxxError(...))`. Nunca lança `throw` para fora do `execute`.
4. **DI exclusivamente via constructor** com `private readonly` tipado pela interface de domínio — nunca pela implementação concreta.
5. **Sem tipos ou interfaces soltos no arquivo.** Definições de `type`, `interface` ou `enum` não vivem em arquivos da `application/` — pertencem a `domain/`.
6. **Contratos (`Result`) vivem em `domain/`**, não em `application/`. Use `XxxUseCase.Result` importado de `@/domain/use-cases/...`.

## Naming

| Elemento | Convenção | Exemplo |
|---|---|---|
| Arquivo | `kebab-case` + `-use-case.ts` | `load-customers-use-case.ts` |
| Classe | `PascalCase` + `UseCase` | `LoadCustomersUseCase`, `CloneSalesOrderUseCase` |
| Constructor param | `private readonly` + nome da interface | `private readonly repository: CustomerRepository` |

## Tipagem e constantes

1. **Nenhuma declaração de tipo na application layer.** Todos os tipos (`type`, `interface`, `enum`) vêm de `@/domain/`. Qualquer tipo auxiliar necessário pertence ao namespace do contrato correspondente em `domain/use-cases/`.
2. **Use cases retornam `Either<AbstractError, T>` — nunca `Promise<T>` diretamente.** O tipo de retorno do `execute` é sempre `Promise<XxxUseCase.Result>`, onde `Result` é `Either<AbstractError, T>` definido no domain.
3. **Sem constantes de módulo nesta camada.** Valores fixos de configuração local não pertencem a arquivos de `application/` — pertencem ao domain ou à camada que os consome.

## Anti-padrões

❌ **Fazer HTTP diretamente no use case** — toda chamada de infraestrutura deve passar pela interface de repositório injetada no constructor:
```typescript
// ❌ Errado
async execute(): Promise<LoadCustomers.Result> {
    const res = await fetch('/api/customers'); // ← viola isolamento da camada
    return right(await res.json());
}
```

❌ **Lançar `throw` em vez de retornar `left(error)`** — o `execute` nunca lança exceções para fora; erros são encapsulados em `left()`:
```typescript
// ❌ Errado
async execute(id: string): Promise<...> {
    if (!id) {
        throw new BadRequestError('id obrigatório'); // ← retorne left()
    }
}
```

❌ **Usar `error as string` no catch** — o valor capturado é `unknown`; use `const err = error as Error; err.message`:
```typescript
// ❌ Errado
} catch (error) {
    return left(new ServerError(error as string)); // ← cast incorreto
}

// ✅ Correto
} catch (error) {
    const err = error as Error;
    return left(new ServerError(err.message));
}
```

❌ **Converter um `BadRequestError` em `ServerError` no catch** — ao capturar um erro sem verificar `instanceof AbstractError`, erros de domínio lançados internamente perdem seu tipo original:
```typescript
// ❌ Errado — destrói o tipo do erro de domínio
} catch (error) {
    const err = error as Error;
    return left(new ServerError(err.message)); // ← BadRequestError vira ServerError
}

// ✅ Correto — preserva o tipo original
} catch (error) {
    if (error instanceof AbstractError) {
        return left(error);
    }
    const err = error as Error;
    return left(new ServerError(err.message));
}
```

> **Nota:** a application layer do React não possui `BaseUseCaseImpl`. Por isso, o catch manual deve sempre verificar `instanceof AbstractError` antes de criar um `ServerError` — caso contrário, um `BadRequestError` ou `NotFoundError` lançado internamente seria convertido em `ServerError`, perdendo o código HTTP e a mensagem de negócio originais.

## Quem importa a application layer

| Camada | Pode importar de `application/`? |
|---|---|
| `main/factories/` | ✅ Instancia as classes de use case |
| `presentation/` | ❌ Nunca — presentation só conhece a interface do domain |
| `infra/` | ❌ Nunca |
| `domain/` | ❌ Nunca |

## Documentos desta seção

- [use-cases.md](./use-cases.md) — padrão de implementação, `left()`/`right()`, tratamento de erros
