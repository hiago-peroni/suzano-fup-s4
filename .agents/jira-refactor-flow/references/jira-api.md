# Jira Refactor Shared Rules

Regras compartilhadas para operacoes no Jira via REST API. Lida uma vez no inicio
do `jira-refactor-flow` e mantida em memoria ate o fim.

## Credenciais

As credenciais do Jira estao em `~/.config/kilo/kilo.json` (nao `.jsonc`):

```bash
JIRA_USER=$(grep '"JIRA_USERNAME"' ~/.config/kilo/kilo.json | head -1 | sed 's/.*: *"//;s/".*//')
JIRA_TOKEN=$(grep '"JIRA_API_TOKEN"' ~/.config/kilo/kilo.json | head -1 | sed 's/.*: *"//;s/".*//')
JIRA_URL="https://b2rise.atlassian.net"
```

## Operacoes REST API

### Ler issue (GET)

```bash
curl -s -u "$JIRA_USER:$JIRA_TOKEN" \
  "$JIRA_URL/rest/api/2/issue/{KEY}?fields=summary,status,assignee,description,priority,issuetype,labels,subtasks,issuelinks" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); ..."
```

### Obter meu usuario (GET /myself)

```bash
curl -s -u "$JIRA_USER:$JIRA_TOKEN" "$JIRA_URL/rest/api/2/myself" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['accountId']); print(d['displayName'])"
```

### Obter transicoes (GET)

```bash
curl -s -u "$JIRA_USER:$JIRA_TOKEN" \
  "$JIRA_URL/rest/api/2/issue/{KEY}/transitions" \
  | python3 -c "import sys,json; [print(f'{t[\"id\"]}: {t[\"name\"]}') for t in json.load(sys.stdin)['transitions']]"
```

### Atribuir issue (PUT)

```bash
curl -s -o /dev/null -w "%{http_code}" -u "$JIRA_USER:$JIRA_TOKEN" \
  -X PUT -H "Content-Type: application/json" \
  "$JIRA_URL/rest/api/2/issue/{KEY}" \
  -d '{"fields":{"assignee":{"accountId":"{ACCOUNT_ID}"}}}'
# 204 = sucesso
```

### Mover issue (POST transition)

```bash
curl -s -o /dev/null -w "%{http_code}" -u "$JIRA_USER:$JIRA_TOKEN" \
  -X POST -H "Content-Type: application/json" \
  "$JIRA_URL/rest/api/2/issue/{KEY}/transitions" \
  -d '{"transition":{"id":"{TRANSITION_ID}"}}'
# 204 = sucesso
```

### Adicionar comentario (POST)

```bash
curl -s -u "$JIRA_USER:$JIRA_TOKEN" \
  -X POST -H "Content-Type: application/json" \
  "$JIRA_URL/rest/api/2/issue/{KEY}/comment" \
  -d "{\"body\":\"{COMENTARIO_MARKDOWN}\"}" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'Comment ID: {d[\"id\"]}')"
```

## Board RFIA

| Parametro | Valor |
|-----------|-------|
| Board ID | 2011 |
| Board URL | https://b2rise.atlassian.net/jira/software/projects/RFIA/boards/2011 |
| Projeto Jira | RFIA |

## Transicoes do board

| Coluna | ID | Quando no fluxo |
|--------|-----|-----------------|
| Backlog | 11 | Task criada |
| Em analise | 21 | Atribuida + plano sendo criado |
| Fazendo | 31 | Execucao |
| Para Testar | 41 | Review + card atualizado |
| Encerramento | 51 | Decision log finalizado |
| Feito | 61 | Tudo verde (skill separada) |

## Regras criticas

| # | Regra |
|---|-------|
| 1 | **Use curl para TODAS as operacoes de escrita.** O MCP atlassian so tem tools de leitura. |
| 2 | **Sempre extrair as credenciais de ~/.config/kilo/kilo.json** no inicio da execucao. |
| 3 | **Verificar HTTP 204** em PUT/POST de transition/assign. Qualquer outra coisa = erro. |
| 4 | **Comentarios em Markdown** — Jira aceita formato Markdown nos comentarios. |
| 5 | **NUNCA expor o token** em output de texto — so usar como variavel no bash. |
