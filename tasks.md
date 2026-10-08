---
# Tarefas 

Leitura: `constitution.md` → `spec.md` → `plan.md` → `tests.md` → `tasks.md`. Este arquivo define a ORDEM de execução. Cada tarefa cita os `REQ` que implementa e os `TEST` que comprovam que está pronta.

## Regras para o executor

1. Execute as tarefas T01 a T11 em ordem, numa única passagem, gerando todos os arquivos listados. Não pare entre tarefas.
2. Não há ambiguidade pendente: toda dúvida tem resposta em `spec.md` seção 12 (Decisões). Não registre `[BLOQUEIO]`.
3. Gere SOMENTE os arquivos da tabela abaixo. Nenhum endpoint, campo ou dependência além do que a `spec.md` exige (PRI-02).
4. Valores de cobrança: use a tabela da `spec.md` seção 10. O exemplo `95 min → 1250` e o exemplo `{"valor": 12.50}` do contrato NÃO valem.
5. Nunca use `float`, `round()` ou `/` em cálculo de dinheiro ou de média.
6. Nunca use `# noqa`. Função grande demais é dividida (PRI-04).

## Arquivos finais

|Arquivo|Criado em|
|---|---|
|`requirements.txt`|T01|
|`requirements-dev.txt`|T01|
|`pyproject.toml`|T01|
|`Dockerfile`|T01|
|`regras.py`|T02|
|`repositorio.py`|T03|
|`app.py`|T04 a T09|
|`tests/test_regras.py`|T10|
|`tests/test_api.py`|T10|

---

## T01 — Esqueleto do projeto

**REQ:** 070, 071, 073 · **Depende de:** —

1. `requirements.txt` com uma única linha: `flask>=3.0,<4`.
2. `requirements-dev.txt` com duas linhas: `-r requirements.txt` e `pytest>=8`.
3. `Dockerfile` idêntico ao bloco da seção Container do `plan.md` (porta 8003, `CMD ["python", "app.py"]`). Não adicionar `ENV`.
4. `pyproject.toml` com o bloco da seção Qualidade do `plan.md`, mais:
    - em `[tool.ruff.lint.per-file-ignores]`: `"tests/*" = ["PLR2004"]` (valores literais esperados nos testes);
    - em `[tool.pytest.ini_options]`: `pythonpath = ["."]` e `testpaths = ["tests"]`.

**Pronto quando:** os quatro arquivos existem; o `Dockerfile` não depende de nenhuma variável de ambiente.

## T02 — Regras puras (`regras.py`)

**REQ:** 013, 014, 015, 016, 017, 031, 033, 072 · **Depende de:** T01

Sem importar Flask, sem estado global mutável, sem ler o relógio.

1. Constantes no topo, com os valores da `spec.md` seção 3: `TARIFA_HORA_CENTAVOS = 500`, `FRACAO_MINUTOS = 30`, `VALOR_FRACAO_CENTAVOS = TARIFA_HORA_CENTAVOS * FRACAO_MINUTOS // 60`, `TETO_DIARIO_CENTAVOS = 5000`, `TOLERANCIA_MINUTOS = 0`, `FUSO = timezone(timedelta(hours=-3))`.
2. `calcular_minutos(entrada, saida) -> int`: passo 1 da seção 10.
3. `calcular_valor(minutos) -> int`: passos 2 a 4 da seção 10.
4. `calcular_tempo_medio(lista_minutos) -> int`: `0` para lista vazia; senão `(2 * soma + n) // (2 * n)`.
5. `normalizar_data_hora(valor) -> datetime`: converte para `FUSO` e zera microssegundos (`astimezone(FUSO).replace(microsecond=0)`).
6. `formatar_data_hora(valor) -> str`: `normalizar_data_hora(valor).isoformat()`, resultado no formato `AAAA-MM-DDTHH:MM:SS-03:00`.

**Pronto quando:** TEST-001 a TEST-034 passam.

## T03 — Repositório em memória (`repositorio.py`)

**REQ:** 001, 006, 010, 012, 040, 042, 062 · **Depende de:** T02

1. Exceções: `BilheteNaoEncontrado`, `BilheteEmAberto`, `BilheteNaoAberto`.
2. Classe `Repositorio` com `self._bilhetes: dict[int, dict]`, `self._proximo_id = 1` e `self._trava = threading.Lock()`. Cada bilhete é um `dict` com `id`, `placa`, `entrada` (datetime), `status` e, se encerrado, `saida`, `minutos`, `valor_centavos`.
3. Métodos, todos devolvendo CÓPIAS (`dict(bilhete)`), nunca o objeto interno:

|Método|Comportamento|
|---|---|
|`abrir_bilhete(placa, entrada)`|Sob a trava: se a placa tem bilhete `aberto`, levanta `BilheteEmAberto`; senão cria com `id = _proximo_id`, incrementa e devolve|
|`encerrar_bilhete(id_bilhete, saida)`|Sob a trava: inexistente → `BilheteNaoEncontrado`; status ≠ `aberto` → `BilheteNaoAberto`; senão grava `saida`, `minutos` e `valor_centavos` (via `regras`) e muda para `encerrado`|
|`cancelar_bilhete(id_bilhete)`|Sob a trava: mesmas exceções; muda para `cancelado`|
|`listar_ativos()`|Status `aberto`, ordenados por `(entrada, id)` decrescente|
|`listar_por_placa(placa)`|Todos da placa, mesma ordenação|
|`listar_encerrados_no_dia(dia)`|Status `encerrado` com `saida.astimezone(FUSO).date() == dia`|

**Pronto quando:** nenhum método altera estado fora do `with self._trava`; o contador só avança em criação bem-sucedida (D-12).

## T04 — Base HTTP (`app.py`)

**REQ:** 060, 061, 070, 072, 074 · **Depende de:** T03

1. `agora() -> datetime`: `datetime.now(FUSO).replace(microsecond=0)`. É a única leitura de relógio do projeto. As rotas chamam `agora()` pelo nome global do módulo (necessário para o `monkeypatch` do `tests.md`).
2. `responder_erro(codigo, status_http)`: devolve `jsonify({"erro": codigo})` com o status.
3. `representar_bilhete(bilhete) -> dict`: monta a representação da `spec.md` seção 5 conforme o status, com datas via `regras.formatar_data_hora`.
4. Validadores (spec seção 6): `validar_placa(valor) -> bool`, `converter_entrada(valor) -> datetime` (levanta `ValueError` se inválida), `converter_id(texto) -> int | None`, `converter_data(texto) -> date | None`.
5. `criar_app() -> Flask`: cria um `Repositorio` novo, registra as rotas das tarefas T05 a T09 e os `errorhandler` de 404 (`rota_nao_encontrada`), 405 (`metodo_nao_permitido`) e 500 (`erro_interno`).
6. Silenciar o log de acesso: `logging.getLogger("werkzeug").setLevel(logging.ERROR)`.
7. Bloco `if __name__ == "__main__":` com `criar_app().run(host="0.0.0.0", port=8003, debug=False)`.

**Pronto quando:** TEST-190 a TEST-193 passam.

## T05 — UC1 e UC8: abrir bilhete

**REQ:** 001 a 007 · **Depende de:** T04

Rota `POST /bilhetes`, na ordem exata da `spec.md` seção 7:

1. Corpo via `request.get_json(silent=True, force=True)`; não-`dict` vira `{}`.
2. Placa inválida → 422 `placa_invalida`.
3. `entrada` ausente ou `null` → `agora()`; senão `converter_entrada`; `ValueError` → 422 `entrada_invalida`.
4. `abrir_bilhete`; `BilheteEmAberto` → 409 `bilhete_em_aberto`.
5. Sucesso → 201 com `representar_bilhete`.

**Pronto quando:** TEST-101 a TEST-124 passam (exceto os que dependem de T06/T07, que passam ao fim delas).

## T06 — UC2 e UC7: encerrar bilhete

**REQ:** 010 a 017 · **Depende de:** T05

Rota `POST /bilhetes/<id_texto>/encerramento` (parâmetro como texto, nunca `<int:...>`):

1. `converter_id`; `None` → 404 `bilhete_nao_encontrado`.
2. `encerrar_bilhete(id, agora())`; `BilheteNaoEncontrado` → 404 `bilhete_nao_encontrado`; `BilheteNaoAberto` → 409 `bilhete_ja_encerrado`.
3. Sucesso → 200 com `representar_bilhete` (status `encerrado`, 7 chaves).

**Pronto quando:** TEST-130 a TEST-139 e TEST-145 passam.

## T07 — UC5: cancelar bilhete

**REQ:** 040 a 042 · **Depende de:** T05

Rota `POST /bilhetes/<id_texto>/cancelamento`: igual a T06, mas chama `cancelar_bilhete` e `BilheteNaoAberto` → 409 `bilhete_nao_aberto`. Sucesso → 200 com representação `cancelado` (4 chaves).

**Pronto quando:** TEST-140 a TEST-144 passam.

## T08 — UC3 e UC6: listagens

**REQ:** 020, 050, 051 · **Depende de:** T06, T07

1. `GET /bilhetes/ativos` → 200 com `[representar_bilhete(b) for b in listar_ativos()]`.
2. `GET /bilhetes`: placa da query `request.args.get("placa")`; inválida ou ausente → 422 `placa_invalida`; senão 200 com a lista de `listar_por_placa`.

**Pronto quando:** TEST-141, TEST-150 a TEST-165 passam.

## T09 — UC4: relatório diário

**REQ:** 030 a 033 · **Depende de:** T06

Rota `GET /relatorios/diario`:

1. `converter_data(request.args.get("data"))`; `None` → 422 `data_invalida`.
2. `encerrados = listar_encerrados_no_dia(dia)`.
3. 200 com `data` (texto recebido), `total_bilhetes = len(encerrados)`, `faturamento_centavos = sum(valor_centavos)`, `tempo_medio_minutos = regras.calcular_tempo_medio([minutos ...])`.

**Pronto quando:** TEST-170 a TEST-179 passam.

## T10 — Testes automatizados

**REQ:** todos · **Depende de:** T09

1. `tests/test_regras.py`: seções 2 e 3 do `tests.md`, usando `pytest.mark.parametrize` com as tabelas.
2. `tests/test_api.py`: seções 4 a 12 do `tests.md`, com as fixtures `relogio` e `cliente` da seção 1 (TEST-195 é manual, não automatizar).
3. Cada função de teste tem o `TEST-nnn` no nome, ex.: `test_131_fracao_exata`.

**Pronto quando:** `pytest` passa sem falhas.

## T11 — Polimento e verificação final

**REQ:** todos · **Depende de:** T10

1. PRI-04: `ruff check .` sem erros; `wc -l` de cada `.py` ≤ 300.
2. PRI-05: nomes sem abreviação fora de `id`, `url`, `db`, `api`; funções começam com verbo; booleanos com `is`, `has`, `can` ou `should`.
3. Conferir o checklist abaixo item a item.

| Item                  | Verificação                                            |
| --------------------- | ------------------------------------------------------ |
| Porta                 | Servidor em `0.0.0.0:8003`                             |
| Nomes de campos       | Exatamente os da `spec.md` seção 5 e REQ-030           |
| Status HTTP           | 201 abrir; 200 demais sucessos; 404/409/422 da seção 8 |
| Erros                 | Sempre `{"erro": codigo}`, nunca HTML                  |
| Dinheiro              | Só `int`; nenhuma chave `valor`; nenhum `float`        |
| Datas                 | `AAAA-MM-DDTHH:MM:SS-03:00` em toda resposta           |
| Precedência           | 422 sempre antes de 409                                |
| Variáveis de ambiente | Nenhuma lida pelo código                               |
|                       |                                                        |

**Pronto quando:** checklist completo, `ruff check .` e `pytest` limpos.
