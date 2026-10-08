# Constituição

Este documento contém regras que valem para todo o projeto, independentemente
de funcionalidade. Nenhum termo do domínio aparece aqui.

**Precedência:**  **Contrato da prova > CONSTITUTION > SPEC > PLAN > TESTS > TASKS**.
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
**Regra:** ao encontrar ambiguidade, aplicar a decisão da seção 12 do `spec.md`; se não houver, escolher a opção mais restritiva compatível com o contrato e seguir, nunca parar
**Racional:** adivinhação silenciosa é o modo de falha mais caro deste fluxo.

---

## Seção B — Forma do código

### PRI-04 — Tamanho de unidade
**Regra:** | Unidade | Limite | Verificação |
| --- | --- | --- | | Complexidade ciclomática por função | ≤ 8 | `ruff check` (C901) | | Instruções por função | ≤ 25 | `ruff check` (PLR0915) | | Ramos por função | ≤ 10 | `ruff check` (PLR0912) | | Parâmetros por função | ≤ 5 | `ruff check` (PLR0913) | | Comprimento de linha | ≤ 100 caracteres | `ruff check` (E501) | | Linhas por arquivo | ≤ 300 | `wc -l` (o Ruff não cobre) |
**Racional:** limites objetivos substituem a discussão sobre "função pequena".
O agente deve criar o `pyproject.toml` com a configuração do Ruff descrita no `plan.md`. Função que ultrapassar um limite deve ser dividida, nunca ter a regra desativada com `# noqa`.

### PRI-05 — Nomes
**Regra:** sem abreviação fora da lista aprovada (`id`, `url`, `db`, `api`);
booleano começa com `is`, `has`, `can` ou `should`; função começa com verbo `agora` e os handlers `rota_nao_encontrada`, `metodo_nao_permitido` e `erro_interno`.
**Verificação:** revisão na tarefa de polimento de cada fase.

---

## Seção C — Segurança (OWASP Top 10 / ASVS L1)

### PRI-06 — Validação na borda  [A03]
**Regra:** "validação na borda por funções do adaptador HTTP; campos não declarados são ignorados"

### PRI-10 — Dados pessoais  [LGPD]
**Regra:** campo que contenha dado pessoal é marcado no modelo e nunca
aparece em log.

### PRI-11 — Erro não vaza  [A05, A09]
**Regra:** rcorpo de erro tem exatamente uma chave: `erro`

