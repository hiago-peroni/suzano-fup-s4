# AGENTS.md — suzano-fup-s4

Instruções canônicas para agentes de IA. **Leia tudo antes de escrever qualquer código.**
Para o humano, a porta de entrada é o [`README.md`](README.md).

---

## 1. Overview

Monorepo do FUP (Follow-Up de Pedidos) — sistema SAP que gerencia o acompanhamento de pedidos de compra com fornecedores. Consolida 3 repositórios (2 frontends UI5 + 1 backend CAP com 3 serviços) em um único monorepo no padrão IAtização, refatorando o backend para conformidade total com o standard `cap-clean-architecture`.

O código está em **produção ativa** — toda refatoração deve preservar comportamento.

## 2. Estrutura do repositório

```
suzano-fup-s4/
├── packages/
│   ├── backend/
│   │   ├── application-service/     ← PanelService (/fup/panel)
│   │   ├── processing-service/      ← ProcessingService (/fup-panel-processing)
│   │   ├── public-service/          ← PublicService (/fup/public)
│   │   ├── db/                      ← 62 entidades CDS + HANA HDI
│   │   ├── app-router/              ← interno (XSUAA)
│   │   └── public-app-router/       ← público (sem auth)
│   └── frontend/
│       ├── fup-panel/               ← app principal
│       ├── manual-fups/             ← app de manuais
│       ├── role-management/         ← gestão de roles
│       └── dynamic-forms/           ← formulários públicos
├── docs/                            ← pesquisas, specs, standards (READ-ONLY)
│   ├── standards/                   ← padrões canônicos da empresa
│   ├── researches/                  ← pesquisas (deep-research)
│   ├── specs/                       ← specs do projeto (tlc-spec-driven)
│   └── obsidian/                    ← vault Obsidian (symlink)
├── scripts/                         ← scripts utilitários
├── .github/                         ← CI/CD + PR template
├── mta.yaml                         ← deploy MTA (Cloud Foundry)
├── xs-security.json                ← XSUAA config (interno)
├── package.json                     ← orquestrador (bd, build, deploy, undeploy, clean)
├── .nvmrc                           ← Node 22
├── renovate.json                    ← updates automáticos (patch-only)
├── AGENTS.md                        ← ESTE ARQUIVO (índice geral)
└── README.md                        ← porta de entrada humana
```

## 3. Convenções globais

- **Package manager:** SEMPRE use `yarn` — NUNCA use `npm` ou `bun`.
- **Node:** versão 22 (`.nvmrc` na raiz).
- **Git flow:** `feature/*` ou `bugfix/*` → PR para `quality` → merge automático para `main` via `NUMENDS/git-flow-action`.
- **Commits:** git flow — `<ação>: <descrição em até 80 caracteres>`. Ações: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `ci`, `build`, `perf`. NUNCA commitar sem pedido explícito do usuário. NUNCA commitar secrets.
- **Renovate:** `baseBranches: [quality]`, patch-only (major/minor desabilitados).
- **Idioma:** código em inglês; docs e conteúdo gerado em PT-BR; termos técnicos consagrados em inglês.
- **NÃO mate portas** após testes — o usuário roda múltiplos dev servers simultaneamente.
- **Arquivos grandes:** escreva em blocos (write + edit append) para evitar timeout.

## 4. Comandos da raiz (orquestrador)

| Script | Comando | Descrição |
|--------|---------|-----------|
| `bd` | `npm run build && npm run deploy` | Build + deploy MTA |
| `build` | `rimraf resources mta_archives && mbt build --mtar suzano-fup-s4` | Build MTA |
| `deploy` | `cf deploy mta_archives/suzano-fup-s4.mtar --retries 0 -f` | Deploy Cloud Foundry |
| `undeploy` | `cf undeploy suzano-fup-s4 --delete-services --delete-service-keys --delete-service-brokers` | Undeploy |
| `clean` | `rimraf resources mta_archives mta-op*` | Limpar artefatos |

Para comandos dos apps (dev, lint, ts-typecheck, test:unit, build:cf), veja os `AGENTS.md` em `packages/backend/AGENTS.md` e `packages/frontend/AGENTS.md`.

## 5. Zonas de Proteção (Read-Only)

- A pasta `docs/researches/` contém pesquisas feitas antes de implementar o projeto. NUNCA sugira refatorações, deleções ou alterações nos arquivos desta pasta.
- A pasta `docs/specs/` contém especificações e tasks. Trate como Read-Only — se houver um bug, instrua o usuário a corrigir em conjunto.
- A pasta `docs/standards/` contém padrões canônicos da empresa (Clean Architecture + code style). NUNCA altere estes arquivos — eles são referência transversal.
- O arquivo `docs/specs/project/STATE.md` é a memória persistente do projeto. Atualize-o apenas ao registrar decisões, blockers ou lessons learned.

## 6. Protocolo de Handoff

- Ao finalizar a implementação ou correção de bug, atualize o arquivo `docs/specs/project/STATE.md` na seção apropriada.
- Registre um resumo de no máximo 5 linhas do que foi feito e quais os próximos passos pendentes.
- Sempre que iniciar uma mudança no código, leia silenciosamente o `STATE.md` para entender o estado atual do projeto.

## 7. Documentações de Referência Dinâmica

- Se a tarefa envolver o **backend**, comece pelo arquivo `packages/backend/AGENTS.md` — ele contém as regras específicas de stack, convenções, arquitetura e testes do backend.
- Se a tarefa envolver o **frontend**, comece pelo arquivo `packages/frontend/AGENTS.md`.
- Se a tarefa envolver **deploy/MTA**, leia `mta.yaml` e `xs-security.json` na raiz.
- Se a tarefa envolver **CI/CD**, leia `.github/workflows/`.
- Se a tarefa envolver **padrões arquiteturais**, leia `docs/standards/cap-clean-architecture/README.md` (Clean Architecture canônica) e `docs/standards/code-style/typescript.md` (code style).
- Se a tarefa envolver **pesquisa/specs**, leia `docs/researches/fup-backend-refactor/research.md` e `docs/specs/features/fup-backend-refactor/`.
- Se a tarefa envolver **vault Obsidian**, leia `docs/obsidian/Obsidian/suzano-s4/` (senso, decisões, padrões do projeto).
