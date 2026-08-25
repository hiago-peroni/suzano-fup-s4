# Decision Log — suzano-fup-s4

## Decisoes

| ID | Data | Card | Arquivo | Acao | Porque | Padrao seguido | Status |
|----|------|------|---------|------|--------|----------------|--------|
| D1 | 2026-08-25 | RFIA-17 | AGENTS.md | Criar (scaffold) | T-AGENTS-ROOT do scaffold prompt; ≤150 linhas | [[Padrão Monorepo]] | done |
| D2 | 2026-08-25 | RFIA-17 | package.json | Criar (scaffold) | T-PACKAGE-JSON com HAS_MTA=true; MTA_ID=suzano-fup-s4 v2.0.0 (D1 das Decisões Arquiteturais) | [[Padrão Monorepo]] | done |
| D3 | 2026-08-25 | RFIA-17 | .nvmrc | Criar (scaffold) | Node 22 conforme stack alvo do Senso | [[Padrão Monorepo]] | done |
| D4 | 2026-08-25 | RFIA-17 | .gitignore | Criar (scaffold) | T-GITIGNORE + gen/ + @cds-models/ (exigidos pelo Done when do card) | [[Padrão Monorepo]] | done |
| D5 | 2026-08-25 | RFIA-17 | renovate.json | Criar (scaffold) | T-RENOVATE com baseBranches=[quality] patch-only | [[Git Flow Padrão]] | done |
| D6 | 2026-08-25 | RFIA-17 | packages/backend/ | Criar (scaffold) | Estrutura monorepo: packages/backend/ para 3 serviços CAP | [[Padrão Monorepo]] | done |
| D7 | 2026-08-25 | RFIA-17 | packages/backend/AGENTS.md | Criar (scaffold) | Regras específicas do backend (3 serviços, Clean Architecture 5 camadas, error taxonomy) | [[Clean Architecture]] | done |
