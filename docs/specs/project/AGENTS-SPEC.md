# AGENTS.md

O AGENTS.md é um arquivo Markdown que descreve para a I.A. o que é o projeto e como ela deve agir durante a leitura e desenvolvimento do código.

### Estrutura do AGENTS.md

1. Overview:
    
    Um resumo mais focado no que o projeto faz de fato, quem consome e o que gera.
    
2. Comandos de teste e build:
    
    Quais comandos rodam o build e os testes do projeto, alem de outros comandos importantes
    
3. Estrutura do projeto:
    
    É como será a estrutura, arquitetura e organização interna do projeto, dos arquivos, extensões, barrels e etc.
    
4. Convenções globais e boas praticas:
    
    Versão de bibliotecas, padrões de organização, lint, nomenclaturas, imports, idioma e etc
    
5. Instruções de teste:
    
    Como gerar, rodar e o que esperar dos testes
    
6. Segurança:
7. Convenções e regras de segurança

O consenso é que o arquivo deve ter de 100 a 150 linhas. Um AGENTS.md **NUNCA** deve passar de 300 linhas. Prefira multiplos AGENTS.md aninhados.

A leitura do AGENTS.md pela I.A. são feitas por hierarquia, então ele lê o arquivo **mais próximo** do projeto que ele está lendo (*proximity-based*). O arquivo AGENTS.md da raiz dá uma indexação e regra sglobais, mas os AGENTS.md dentro dos repositórios que devem específicar seus prórpios projetos

O que é **OBRIGATÓRIO** ter em um AGENTS.md:

- Stack exata do sistema: Linguagens, frameworks e suas versões usadas, pois isso impede a IA de usar sintaxe de outra versão durante o desenvolvimento
- Convenção de Nomenclatura: qual tipo de “case” usar (PascalCase, snake_case e etc), sufixos
- Gestão de Tipagem/Erros: Usar o either para tratamento de erros, proibido uso de “any”

A I.A. lida melhor com um fluxo baseado em **AÇÕES** ao invés de explicações, então use palavras como “DEVE”, “SEMPRE”, “FAÇA”, “NUNCA”. Trabalhe sempre ações específicas, como “as funções DEVEM ter, no máximo, 30 linhas”, “SEMPRE siga a Clean Architecture durante o desenvolvimento”, “SEMPRE use o `yarn` ao invés do `npm`". 

Especificações de REGRAS DE NEGÓCIO do projeto não entram nesse arquivo, pois ele é um guia para a I.A. saber como deve fazer o DESENVOLVIMENTO do projeto. 

NUNCA insira credencias e segredos da aplicação

### Regra para o usuário:

Não adicione as regras do AGENTS às rules do agent usado pois pode causar um *code drift* e o agent se perder nas regras de desenvolvimento

### Pontos extras a se considerar na criação do AGENTS.md:

- Guardrails:
Usar em arquivos que a I.A. não pode alterar

```tsx
## Zonas de Proteção (Read-Only)
- A pasta `/src/docs/deep-research` contém as pesquisas feitas antes de implementar o projeto. NUNCA sugira refatorações, deleções ou alterações nos arquivos desta pasta.
- Trate esses arquivos como *Read-Only*. Se houver um bug nesses fluxos, instrua o usuário a consertar em conjunto.
```

- Contexto Dinâmico:
Para lidar com grandes projetos, usamos o AGENTS.md da raiz como índice para ler os outros AGENTS.md dos projetos quando necessário

```tsx
## Documentações de Referência Dinâmica
- Se a tarefa envolver o `Painel FUP`, comece pelo arquivo packages/frontend/painel-fup e packages/backend/painel-fup
```

- Protocolo de Handoff:
Podemos criar um arquivo auxiliar de “estado do projeto após alterações”, como um `.agent-memory.md` que vai ser atualizado após novas implementações e correções, para manter um histórico de checagem e uma contextualização para a próxima sessão

```tsx
## Protocolo de Handoff
- Ao finalizar a implementação ou correção de bug, atualize o arquivo `.agent-memory.md` na raiz do projeto.
- Registre ali um resumo de no máximo 5 linhas do que foi feito e quais os próximos passos pendentes. 
- Sempre que você (o agent) iniciar uma mudança no código, leia silenciosamente o `.agent-memory.md` para entender o estado atual do projeto.
```