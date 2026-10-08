# Constituição

Este documento contém regras que valem para todo o projeto, independentemente
de funcionalidade. Nenhum termo do domínio aparece aqui.

**Precedência:** CONSTITUTION > SPEC > PLAN > TESTS > TASKS > código.
Em conflito, o de maior precedência vence e o de menor é corrigido.

**Como ler:** cada princípio tem ID `PRI-nn`, uma regra objetiva, o mecanismo
que a verifica e o racional. Regra sem mecanismo não é regra.

---

## Seção A — Processo

### PRI-01 — A especificação é a fonte da verdade
**Regra:** nenhum comportamento observável existe no código sem um `REQ-nnn`
correspondente. Nenhum `REQ-nnn` existe sem ao menos um `TEST-nnn`.
**Racional:** o que não está escrito não pode ser revisado nem regenerado.

### PRI-02 — Escopo fechado
**Regra:** é proibido implementar funcionalidade não exigida por um `REQ`.
Isso inclui campos extras, endpoints "úteis" e opções de configuração.
**Racional:** funcionalidade não especificada é defeito, não bônus.

### PRI-03 — Ambiguidade é bloqueio
**Regra:** ao encontrar ambiguidade, o executor PARA, registra
`[BLOQUEIO: pergunta]` na saída e não adivinha.
**Racional:** adivinhação silenciosa é o modo de falha mais caro deste fluxo.

---

## Seção B — Forma do código

### PRI-04 — Tamanho de unidade
**Regra:** | Unidade | Limite | Verificação | | --- | --- | --- | | Complexidade ciclomática por função | ≤ 8 | `ruff check` (C901) | | Instruções por função | ≤ 25 | `ruff check` (PLR0915) | | Ramos por função | ≤ 10 | `ruff check` (PLR0912) | | Parâmetros por função | ≤ 5 | `ruff check` (PLR0913) | | Comprimento de linha | ≤ 100 caracteres | `ruff check` (E501) | | Linhas por arquivo | ≤ 300 | `wc -l` (o Ruff não cobre) |
**Racional:** limites objetivos substituem a discussão sobre "função pequena".
O agente deve criar o `pyproject.toml` com a configuração do Ruff descrita no `plan.md`. Função que ultrapassar um limite deve ser dividida, nunca ter a regra desativada com `# noqa`.

### PRI-05 — Nomes
**Regra:** sem abreviação fora da lista aprovada (`id`, `url`, `db`, `api`);
booleano começa com `is`, `has`, `can` ou `should`; função começa com verbo.
**Verificação:** revisão na tarefa de polimento de cada fase.

---

## Seção C — Segurança (OWASP Top 10 / ASVS L1)

### PRI-11 — Validação na borda  [A03]
**Regra:** todo dado vindo de fora (body, query, params, header, webhook)
passa por schema Zod no adaptador HTTP, com `.strict()`, antes de chegar ao
serviço. Campo não declarado faz a requisição falhar com 422.


### PRI-12 — Autorização no serviço  [A01]
**Regra:** a decisão de autorização ocorre na camada de serviço, com base no
ator e no recurso. O controller nunca é a única barreira. Toda consulta a
recurso de propriedade de usuário filtra por dono na query, não após ela.

### PRI-13 — Sem SQL concatenado  [A03]
**Regra:** toda consulta usa ORM ou query parametrizada. `$queryRawUnsafe`
e template string com interpolação em SQL são proibidos.

### PRI-14 — Segredos fora do código  [A02, A05]
**Regra:** nenhuma credencial, chave ou token literal no repositório. Acesso
exclusivamente pelo módulo `config`, que lê de variável de ambiente e valida
na inicialização, falhando rápido se faltar.

### PRI-15 — Dados pessoais  [LGPD]
**Regra:** campo que contenha dado pessoal é marcado no modelo e nunca
aparece em log, em mensagem de erro ou em resposta de listagem pública.
Toda entidade com dado pessoal tem rota de exportação e de eliminação.

### PRI-16 — Erro não vaza  [A05, A09]
**Regra:** resposta de erro contém apenas código, título, status e mensagem
redigida. Nunca stack, SQL, nome de classe, caminho de arquivo ou versão.

### PRI-17 — Log de segurança  [A09]
**Regra:** log estruturado em JSON. Eventos de autenticação, autorização
negada e alteração de permissão são sempre registrados, com ator e resultado,
e nunca com credencial ou dado pessoal.

