---
name: abrir-pr
description: Abre ou atualiza um Pull Request no GitHub para a branch atual usando o CLI `gh`, sempre como Draft, com título e corpo em português brasileiro vinculados a uma issue. Use sempre que o usuário pedir para "abrir PR", "criar pull request", "subir PR", "fazer PR dessa branch", "atualizar o PR" ou similar — mesmo que ele não diga explicitamente qual issue referenciar, pois a skill tenta detectar automaticamente. Não use para apenas commitar ou dar push (a skill nunca faz isso), nem para revisar/mergear PRs existentes.
argument-hint: "[opcional: ID da issue]"
allowed-tools:
  - Read
  - Bash(git *)
  - Bash(gh *)
---

# Abrir PR

Cria ou atualiza um Pull Request no GitHub para a branch atual, via `gh` CLI. PR sempre em **Draft**, título e corpo em **pt-BR**, referenciando uma issue.

Esta skill NUNCA roda `git commit` ou `git push`. Se a branch não estiver publicada no remoto, pare e peça pro usuário dar push antes.

Input do usuário: $ARGUMENTS

## Fluxo

### 1. Descobrir branch atual e validar

```bash
git rev-parse --abbrev-ref HEAD
```

Se for `main`/`master` (ou a branch default do repo), avise o usuário que não dá pra abrir PR a partir da branch base e pare.

Confira se a branch tem upstream e está publicada:

```bash
git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null
```

Se não tiver upstream, ou se `git status -sb` / `git log @{u}..HEAD` mostrar commits locais não enviados, **pare e avise o usuário** para dar push primeiro — a skill não faz isso por ele.

### 2. Verificar se já existe PR aberto pra essa branch

```bash
gh pr view --json number,title,body,url,baseRefName,isDraft 2>/dev/null
```

Se retornar dado válido, é uma atualização (passo 10b). Se der erro/vazio, é criação nova (passo 10a).

### 3. Descobrir a branch base

```bash
gh repo view --json defaultBranchRef -q .defaultBranchRef.name
```

Use esse valor como `<base>` em todos os passos seguintes (e como `--base` no `gh pr create`), a menos que o usuário já tenha indicado outra base.

### 4. Descobrir a issue relacionada

Ordem de tentativa:

1. **Usuário informou explicitamente** no pedido (ex: "#123", "issue 123", link do GitHub) ou em `$ARGUMENTS` — use esse número direto.
2. **Nome da branch** — procure um número nele (ex: `123-fix-login`, `feature/123-algo`, `fix/GH-123`). Rode `gh issue view <N>` pra confirmar que a issue existe antes de assumir que é ela.
3. **Commits da branch** — rode `git log <base>..HEAD` e procure menções tipo `#123`, `issue 123`, `closes 123` nas mensagens.

Se nenhuma tentativa achar algo, ou se achar mais de um número plausível e diferente entre si, **pergunte ao usuário** qual issue é a correta antes de continuar. Não invente número de issue.

Com o número em mãos, busque os dados dela:

```bash
gh issue view <N> --json title,body,url
```

### 5. Analisar a entrega

```bash
git log <base>..HEAD --format='%h %s%n%b'
```

Leia os commits pra entender o que foi de fato entregue nessa branch, e compare com o que a issue pediu — é essa relação que vai virar o corpo do PR.

### 6. Montar título e corpo (pt-BR)

**Título**: texto livre em português, direto, descrevendo a entrega principal da branch (baseado nos commits, não copiado literalmente da issue). Sem prefixo de tipo (feat/fix/...).

**Corpo**: siga essa estrutura —

```markdown
## Resumo

<1-3 frases resumindo o que a issue #<N> pedia e como os commits desta branch atendem a esse pedido>

Closes #<N>
```

O resumo deve de fato relacionar pedido da issue com entrega — não é só reescrever o título da issue. Se a branch entrega só parte do que a issue pede, diga isso no resumo em vez de fingir que está completo.

### 7. Validar que código novo veio com teste

Antes de escolher revisor ou montar PR, cheque se arquivo novo criado nessa branch exige teste junto. Liste só os **arquivos criados** (não conta alterado nem removido):

```bash
git diff <base>..HEAD --diff-filter=A --name-only
```

Aplique estas regras:

| Arquivo novo em... | Precisa ter teste novo em... |
|---|---|
| algum diretório chamado `Repository` ou `Service` dentro de `backend` (em qualquer nível, ex: `backend/modules/pedidos/Service/PedidoService.cs`) | `backend/tests` |
| `frontend/src/api` | `frontend/tests/api` |

Pra cada arquivo novo que bater numa regra, procure na lista de arquivos criados um teste correspondente na pasta esperada. Não basta *existir algum arquivo* na pasta de teste — confira que o teste realmente parece cobrir aquele arquivo (nome parecido, ex. `PedidoService.cs` → algo como `PedidoServiceTests.cs`/`pedido_service.test.ts`; se o nome não bater, abra o teste e confira se ele importa/exercita o arquivo em questão antes de aceitar).

Se algum arquivo novo bater numa regra e não tiver teste correspondente criado na branch: **pare o fluxo aqui**. Não continue pros próximos passos (revisores, montar PR, criar/atualizar). Avise o usuário, em texto, listando exatamente quais arquivos novos ficaram sem teste e em qual pasta o teste era esperado. Só retome depois que o usuário disser que os testes foram adicionados (rode o diff de novo pra confirmar) ou pedir explicitamente pra ignorar essa checagem.

### 8. Escolher revisores pelo código modificado

"Código tocado" = tudo que essa branch cria, altera ou remove em relação à base. Veja os arquivos:

```bash
git diff <base>..HEAD --name-only
```

E o conteúdo de fato alterado (não só nomes de arquivo, já que a regra de segurança é sobre o que o código faz):

```bash
git diff <base>..HEAD
```

Aplique estas regras (não são exclusivas entre si — podem se acumular vários revisores):

| Condição | Time Revisor |
|---|---|
| Algum arquivo tocado está em `backend/migrations` e/ou `backend/seeds` | time `staff-db` |
| Algum arquivo tocado está em `frontend` | time `frontenders` |
| O código tocado envolve fluxos de autenticação, autorização, permissionamento, criptografia ou hash | time `securityeng` |

Para as duas primeiras regras, checar o caminho do arquivo já basta. Para a de segurança, não dá pra confiar só em grep de palavra-chave — leia de fato o diff (ou os arquivos, se o diff não for claro sozinho) e julgue se a mudança mexe em: login/logout, sessões, tokens, JWT, OAuth, controle de acesso (roles/permissions/ACL), criptografia/decriptação, ou geração/verificação de hash (senha, assinatura, etc). Coisas que só *mencionam* essas palavras em comentário ou nome de variável sem implementar o fluxo não contam.

Se nenhuma regra bater, não adicione revisor nenhum — não force um revisor "por garantia".

> Os revisores são nomes de **times** do GitHub. Execute `gh repo view --json owner --jq '.owner.login'` pra descobrir a organização, e monte o reviewer como `<org>/<slug-do-time>`.

### 9. Confirmar com o usuário — SEMPRE

Antes de rodar qualquer `gh pr create` ou `gh pr edit`, mostre pro usuário, em texto:

- Issue referenciada (número + título)
- Branch base
- Título proposto do PR
- Corpo proposto do PR (completo)
- Revisores escolhidos (e por qual regra cada um foi escolhido) — ou que nenhum bateu
- Se vai criar ou atualizar (e nesse caso, quais campos serão sobrescritos)

Só prossiga depois que o usuário confirmar ou pedir ajustes. Se pedir ajuste, refaça e confirme de novo.

### 10a. Criar PR novo

```bash
gh pr create --draft --base <base> --title "<título>" --body "<corpo>" --reviewer <revisor1> --reviewer <revisor2>
```

Omita `--reviewer` se nenhuma regra do passo 8 bateu.

### 10b. Atualizar PR existente

Por padrão, atualize **título, corpo e revisores**:

```bash
gh pr edit <número> --title "<título>" --body "<corpo>" --add-reviewer <revisor1> --add-reviewer <revisor2>
```

`--add-reviewer` só adiciona — não remove revisor que já estava no PR, mesmo que a regra que o adicionou não bata mais (ex: se um commit novo deixou de tocar `frontend`). Não remova revisor manualmente a menos que o usuário peça.

Não mexa em outros campos (labels, base, draft status etc.) a menos que o usuário peça explicitamente.

## Coisas a não fazer

- Não commitar, não dar push, não fazer merge.
- Não criar PR sem branch estar publicada no remoto.
- Não inventar número de issue quando a detecção automática falhar — pergunte.
- Não pular a confirmação do passo 9, mesmo em atualização de PR já existente.
- Não tirar o PR do modo Draft.
- Não decidir revisor de segurança só por grep de palavra-chave — leia o diff.
- Não seguir pra revisores/PR se faltar teste de arquivo novo (passo 7) sem o usuário liberar explicitamente.
