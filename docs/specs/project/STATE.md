# State — Suzano FUP S4

> Memória persistente do projeto: decisões, blockers, lessons, todos, deferred ideas.

## Decisions

### 2026-08-21 — Research global (fup-s4-migration)

- **D1**: 1 MTA unificado (`suzano-fup-s4` v2.0.0)
- **D2**: Estrutura `packages/frontend/<app>/` + `packages/backend/<service>/` + `packages/backend/db/`. Sem yarn workspaces.
- **D3**: Stack padronizada — TS 5.9.x + UI5 1.120.39 + @ui5/cli v3
- **D4**: 2 app-routers + 2 xs-security.json (interno XSUAA + público sem auth)
- **D5**: role-management será refatorado para Clean Architecture
- **D6**: printWidth 180 + Prettier wired em todos
- **D7**: CI com 1 workflow + paths-filter + NUMENDS/git-flow-action
- **D8**: Testes como P2 (P1 = smoke test de build apenas)

### 2026-08-21 — Research backend (fup-backend-refactor)

- **G1**: Models anêmicos têm ZERO validação implícita — validação vive nos use cases
- **G2**: Bug recursão infinita no getter `message()` é ATIVO (acessado em PROC save-carrier-invoice-data.ts:53,121)
- **G3**: Error taxonomy = 7 classes (6 canônicas + UnavailableServiceError 503). Remover BadGateway (dead code) + UnprocessableEntity (dead code). Mover SAPGatewayError/DellavolpeError para domain/adapters/external-api/.
- **G4**: Rename `db/legacy-eva-models/` → `db/replication/` é seguro (`@cds.persistence.exists` é per-entity)
- **G5**: Sem yarn workspaces (herdado de D2)
- **G6**: 56% mecânico (216 arquivos), 44% manual (260 arquivos)

### 2026-08-21 — Jira board RFIA criado

- 14 épicos criados (RFIA-3 a RFIA-16)
- Board: https://b2rise.atlassian.net/jira/software/projects/RFIA/boards/2011
- Ordem de execução: EP-01 → EP-04 → EP-07 → EP-02 → EP-12 → EP-09 → EP-05 → EP-06 → EP-10 → EP-08

## Blockers

- Nenhum no momento.

## Lessons Learned

- O getter `get message() { return this.message }` é recursão infinita — provavelmente não estoura em produção por V8 prototype shadowing do `Error.message` nativo, mas é bomba relógio.
- `BadGatewayError` e `UnprocessableEntity` foram criados mas nunca instanciados — sempre verificar uso real antes de manter classes de erro.
- `SAPGatewayError` e `DellavolpeError` são namespaces de TYPES, não classes de erro — confusão comum ao nomear arquivos de type definitions em `domain/errors/`.
- Validação de domínio vivendo em use cases ao invés de models é o padrão atual — converter para rich models exige reposicionar a lógica, não apenas mudar `type` para `class`.

## Todos

- [ ] Criar ADR justificando `UnavailableServiceError` (503) como 7ª classe de erro
- [ ] Criar ADR para Clean Architecture 5 camadas como padrão (retroativo)
- [ ] Criar ADR para `@sweet-monads/either` como Either monad canônico (retroativo)
- [ ] Criar ADR para Factory pattern (DI manual) sem container (retroativo)
- [ ] Deep research individual por app frontend (M3)

## Deferred Ideas

- Extrair `packages/backend/shared/` com código compartilhado (M2, EP-11)
- @cds-models gerado automaticamente do backend para frontends (M4)
- Playwright E2E (pós-M4)
- Anexação ao `suzano-eva-frontend-s4` (futuro)

## Preferences

- Baby steps: especificar → implementar → validar → próxima.
- Sempre passar prompt completo para agentes externo.
- Tarefas leves (state updates, validation) funcionam bem com modelos mais rápidos/baratos.
- Tarefas pesadas (brownfield mapping, complex design) exigem reasoning profundo.
