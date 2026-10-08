---

# Especificação funcional

Leitura: `constitution.md` → `spec.md` → `plan.md` → `tests.md` → `tasks.md`. Este documento define O QUE o serviço faz. O `plan.md` define COMO.

**Para o executor:** todas as ambiguidades do contrato já foram resolvidas na seção 12 (Decisões). Não existe pergunta em aberto: não registre `[BLOQUEIO]`, implemente a decisão escrita.

---

## 1. Visão geral

Serviço HTTP com corpo JSON que controla bilhetes de estacionamento rotativo (Zona Azul). Um bilhete é aberto para uma placa, depois é encerrado (com cobrança) ou cancelado (sem cobrança). O serviço também lista bilhetes abertos, mostra o histórico de uma placa e gera um relatório diário.

Não há autenticação, usuários, banco de dados nem variáveis de ambiente.

## 2. Glossário

|Termo|Significado|
|---|---|
|Bilhete|Registro de um período de estacionamento de uma placa|
|Placa|Identificador do veículo, 7 caracteres `A-Z` ou `0-9`|
|Entrada|Instante em que o bilhete começa a contar|
|Saída|Instante do encerramento (sempre o relógio do servidor)|
|Fração|Bloco mínimo de cobrança: 30 minutos|
|Tolerância|Minutos iniciais grátis: 0 nesta variante|
|Teto|Valor máximo cobrado por bilhete: 5000 centavos|

## 3. Parâmetros da variante (constantes fixas)

|Constante|Valor|Observação|
|---|---|---|
|`TARIFA_HORA_CENTAVOS`|`500`|Hora cheia = R$ 5,00|
|`FRACAO_MINUTOS`|`30`|Arredonda sempre para cima|
|`VALOR_FRACAO_CENTAVOS`|`250`|`500 / (60 / 30)`, inteiro exato|
|`TETO_DIARIO_CENTAVOS`|`5000`|Limite por bilhete|
|`TOLERANCIA_MINUTOS`|`0`|Sem minutos grátis além de 0|
|`PORTA_SERVICO`|`8003`|Bind em `0.0.0.0`|
|Fuso|`-03:00`|Offset fixo|

As constantes de cobrança ficam como literais no topo de `regras.py`; a porta fica em `app.py`. Não são lidas de variável de ambiente, arquivo ou parâmetro.

## 4. Modelo de dados

### 4.1 Bilhete

|Campo|Tipo|Regra|
|---|---|---|
|`id`|`int`|Sequencial global iniciando em 1; nunca reutilizado|
|`placa`|`str`|Formato da seção 6.2|
|`entrada`|`datetime` com fuso|Convertido para `-03:00`, microssegundos zerados|
|`status`|`str`|`aberto`, `encerrado` ou `cancelado`|
|`saida`|`datetime` com fuso|Só existe se `encerrado`|
|`minutos`|`int`|Só existe se `encerrado`|
|`valor_centavos`|`int`|Só existe se `encerrado`|

O contador de `id` só avança quando um bilhete é efetivamente criado (201). Requisição rejeitada (422 ou 409) não consome `id`.

### 4.2 Máquina de estados

|Estado atual|Ação|Novo estado|Resposta|
|---|---|---|---|
|—|abrir|`aberto`|201|
|`aberto`|encerrar|`encerrado`|200|
|`aberto`|cancelar|`cancelado`|200|
|`encerrado`|encerrar|—|409 `bilhete_ja_encerrado`|
|`cancelado`|encerrar|—|409 `bilhete_ja_encerrado`|
|`encerrado`|cancelar|—|409 `bilhete_nao_aberto`|
|`cancelado`|cancelar|—|409 `bilhete_nao_aberto`|

`encerrado` e `cancelado` são estados finais.

## 5. Representação JSON do bilhete

Toda resposta que devolve bilhete usa exatamente estes campos, conforme o status. Nenhum campo a mais, nenhum a menos. A ordem das chaves é livre.

|Status|Campos|
|---|---|
|`aberto`|`id`, `placa`, `entrada`, `status`|
|`encerrado`|`id`, `placa`, `entrada`, `status`, `saida`, `minutos`, `valor_centavos`|
|`cancelado`|`id`, `placa`, `entrada`, `status`|

Exemplo de bilhete encerrado nesta variante:

```json
{"id": 1, "placa": "ABC1D23", "status": "encerrado",
 "entrada": "2026-10-12T08:30:00-03:00", "saida": "2026-10-12T10:05:00-03:00",
 "minutos": 95, "valor_centavos": 1000}
```

**Formato de data/hora em toda resposta:** `AAAA-MM-DDTHH:MM:SS-03:00`. Sempre convertido para `-03:00`, sem microssegundos, sem `Z`, sem `+00:00`.

**Proibido:** campo `valor`, valor em reais, qualquer número com ponto decimal. O "exemplo de relatório antigo" `{"id": 7, "valor": 12.50}` está ERRADO e deve ser ignorado.

## 6. Validações de entrada

Toda validação acontece em `app.py`, antes de chamar `repositorio.py` ou `regras.py`.

### 6.1 Corpo de `POST /bilhetes`

- Ler com `request.get_json(silent=True, force=True)` (aceita corpo JSON mesmo sem `Content-Type`).
- Se o resultado não for `dict` (JSON inválido, corpo vazio, lista, número), tratar como `{}` → a placa estará ausente → 422 `placa_invalida`.
- Campos não previstos no corpo são ignorados (decisão D-08).

### 6.2 Placa

Válida somente se for `str` e casar inteira com `^[A-Z0-9]{7}$`. Não aplicar `strip()`, não converter para maiúsculas.

|Valor recebido|Resultado|
|---|---|
|`"ABC1D23"`, `"ABC1234"`, `"1234567"`|válida|
|ausente, `null`, `""`|422 `placa_invalida`|
|`"abc1d23"`, `"Abc1D23"`|422 `placa_invalida`|
|`"ABC1D2"` (6), `"ABC1D234"` (8)|422 `placa_invalida`|
|`"ABC-1D23"`, `" ABC1D23"`, `"ABÇ1D23"`|422 `placa_invalida`|
|`1234567` (número), `["ABC1D23"]`|422 `placa_invalida`|

### 6.3 Entrada (opcional)

|Valor recebido|Resultado|
|---|---|
|ausente ou `null`|`entrada = agora()`|
|`str` aceita por `datetime.fromisoformat` e COM fuso|convertida para `-03:00`, microssegundos zerados|
|`str` sem fuso (ex.: `"2026-10-12T08:30:00"`, `"2026-10-12"`)|422 `entrada_invalida`|
|`str` fora de ISO-8601 (ex.: `"ontem"`, `"12/10/2026 08:30"`, `""`)|422 `entrada_invalida`|
|qualquer não-`str` (número, `true`, objeto)|422 `entrada_invalida`|

"Com fuso" significa `dt.utcoffset() is not None`. Aceitar `Z` e qualquer offset (`+00:00`, `-03:00`, `+05:30`). Exemplo: `"2026-10-12T11:30:00Z"` vira `"2026-10-12T08:30:00-03:00"`.

### 6.4 `id` na rota

O `id` é recebido como texto. Válido somente se casar com `^[0-9]+$` (apenas dígitos ASCII). Inválido (`"abc"`, `"-1"`, `"1.5"`) ou inexistente → 404 `bilhete_nao_encontrado`, sempre em JSON.

### 6.5 Query `data` (relatório)

Válida somente se casar com `^[0-9]{4}-[0-9]{2}-[0-9]{2}$` E for data real (`date.fromisoformat` não falha). Ausente, vazia, `"05/10/2026"`, `"2026-1-5"`, `"2026-13-01"`, `"2026-02-30"` → 422 `data_invalida`.

### 6.6 Query `placa` (histórico)

Mesma regra da seção 6.2. Ausente → 422 `placa_invalida`.

## 7. Precedência de erros

Validação de formato (422) vem SEMPRE antes de regra de negócio (409). Ordem de avaliação, parando no primeiro erro:

|Endpoint|Ordem|
|---|---|
|`POST /bilhetes`|1. placa → 2. entrada → 3. placa já tem bilhete aberto (409)|
|`POST /bilhetes/{id}/encerramento`|1. `id` válido e existente (404) → 2. estado (409)|
|`POST /bilhetes/{id}/cancelamento`|1. `id` válido e existente (404) → 2. estado (409)|
|`GET /bilhetes`|1. placa|
|`GET /relatorios/diario`|1. data|

Exemplo: placa inválida E entrada inválida → `placa_invalida`. Placa válida que já tem bilhete aberto E entrada inválida → 422 `entrada_invalida` (nunca 409).

## 8. Catálogo de erros

Corpo de erro é SEMPRE exatamente `{"erro": "<codigo>"}`: uma única chave.

|HTTP|`erro`|Quando|
|---|---|---|
|422|`placa_invalida`|Seções 6.1, 6.2, 6.6|
|422|`entrada_invalida`|Seção 6.3|
|422|`data_invalida`|Seção 6.5|
|404|`bilhete_nao_encontrado`|Seção 6.4|
|409|`bilhete_em_aberto`|Placa já tem bilhete `aberto` (UC8)|
|409|`bilhete_ja_encerrado`|Encerrar bilhete não `aberto`|
|409|`bilhete_nao_aberto`|Cancelar bilhete não `aberto`|
|404|`rota_nao_encontrada`|Caminho não mapeado|
|405|`metodo_nao_permitido`|Caminho existe, método não|
|500|`erro_interno`|Exceção não tratada|

Nunca devolver HTML, stack trace, nome de classe ou caminho de arquivo.

---

## 9. Requisitos funcionais

Formato: `REQ-nnn` → regra → critérios de aceite (CA) mensuráveis.

### UC1 — Abrir bilhete · `POST /bilhetes`

**REQ-001** — Requisição válida cria bilhete `aberto`.

- CA1: status HTTP 201.
- CA2: corpo = representação `aberto` (seção 5), `status == "aberto"`.
- CA3: o primeiro bilhete criado tem `id == 1`; o seguinte, `id == 2`.

**REQ-002** — Sem `entrada`, usa o relógio.

- CA1: `entrada` da resposta difere de `agora()` em no máximo 2 segundos.
- CA2: `entrada` segue o formato `AAAA-MM-DDTHH:MM:SS-03:00`.

**REQ-003** — Com `entrada` válida, usa o valor informado convertido.

- CA1: `{"placa":"ABC1D23","entrada":"2026-10-12T11:30:00Z"}` → `entrada` `"2026-10-12T08:30:00-03:00"`.
- CA2: `"2026-10-12T08:30:00.987-03:00"` → `"2026-10-12T08:30:00-03:00"`.

**REQ-004** — Placa inválida → 422 `placa_invalida`; nada é criado.

**REQ-005** — Entrada inválida → 422 `entrada_invalida`; nada é criado.

### UC8 — Placa duplicada

**REQ-006** — Uma placa tem no máximo UM bilhete `aberto`.

- CA1: segundo `POST /bilhetes` com a mesma placa → 409 `bilhete_em_aberto`.
- CA2: o bilhete existente não muda; o contador de `id` não avança.
- CA3: verificação e criação ocorrem sob o mesmo lock (atômicas).

**REQ-007** — Depois de encerrar ou cancelar, a placa pode abrir de novo.

- CA1: abrir → encerrar → abrir a mesma placa → 201 com `id` novo.
- CA2: abrir → cancelar → abrir a mesma placa → 201 com `id` novo.

### UC2 — Encerrar bilhete · `POST /bilhetes/{id}/encerramento`

**REQ-010** — Encerrar bilhete `aberto`.

- CA1: status HTTP 200.
- CA2: corpo = representação `encerrado` (seção 5).
- CA3: `saida = agora()`; o corpo da requisição é ignorado.
- CA4: `minutos` e `valor_centavos` seguem a seção 10.

**REQ-011** — `id` inválido ou inexistente → 404 `bilhete_nao_encontrado`.

**REQ-012** — Bilhete `encerrado` ou `cancelado` → 409 `bilhete_ja_encerrado`; o bilhete não muda.

**REQ-013** — Cálculo de minutos (seção 10, passo 1).

- CA1: entrada 08:30:00, saída 10:05:00 → `minutos == 95`.
- CA2: entrada 08:30:00, saída 08:30:59 → `minutos == 0`.

**REQ-014** — Cobrança por fração de 30 min arredondada para cima.

- CA1: 30 min → 250; 31 min → 500; 95 min → 1000.

**REQ-015** — Teto: `valor_centavos` nunca supera 5000.

- CA1: 600 min → 5000; 601 min → 5000; 3 dias (4320 min) → 5000.

**REQ-017** — Dinheiro sempre `int` em centavos.

- CA1: `type(valor_centavos) is int` em toda resposta; não existe campo `valor`.

### UC7 — Tolerância

**REQ-016** — Duração `<= TOLERANCIA_MINUTOS` (0) → `valor_centavos == 0`. Passou da tolerância → cobra desde o minuto zero (a tolerância NÃO é descontada).

- CA1: 0 min → 0 centavos.
- CA2: 1 min → 250 centavos.

### UC3 — Listar ativos · `GET /bilhetes/ativos`

**REQ-020** — Lista bilhetes `aberto`.

- CA1: status 200; corpo é array de representações `aberto`.
- CA2: `encerrado` e `cancelado` nunca aparecem.
- CA3: ordem: `entrada` decrescente; empate por `id` decrescente.
- CA4: nenhum aberto → `[]`.

### UC4 — Relatório diário · `GET /relatorios/diario?data=AAAA-MM-DD`

**REQ-030** — Resumo do dia.

- CA1: status 200; corpo exatamente `{"data", "total_bilhetes", "faturamento_centavos", "tempo_medio_minutos"}`.
- CA2: `data` repete o valor da query.
- CA3: universo = bilhetes `encerrado` cuja `saida` (em `-03:00`) cai na data. `aberto` e `cancelado` não entram (decisão D-04).
- CA4: `total_bilhetes` = quantidade do universo; `faturamento_centavos` = soma de `valor_centavos` do universo (inteiro).

**REQ-031** — `tempo_medio_minutos` = média de `minutos` do universo, arredondando 0,5 para cima, só com inteiros: `(2 * soma + n) // (2 * n)`. Nunca `round()`, nunca divisão com `/`.

- CA1: minutos `[30, 61]` → 46. CA2: `[10, 11]` → 11. CA3: `[10, 10, 11]` → 10.

**REQ-032** — Data inválida ou ausente → 422 `data_invalida`.

**REQ-033** — Dia sem bilhetes → `total_bilhetes`, `faturamento_centavos` e `tempo_medio_minutos` iguais a `0` (sem divisão por zero).

### UC5 — Cancelar bilhete · `POST /bilhetes/{id}/cancelamento`

**REQ-040** — Cancelar bilhete `aberto`, sem cobrança.

- CA1: status 200; corpo = representação `cancelado` (`id`, `placa`, `entrada`, `status: "cancelado"`), sem `saida` nem `valor_centavos`.
- CA2: o bilhete sai de `/bilhetes/ativos` e continua no histórico da placa.
- CA3: o corpo da requisição é ignorado.

**REQ-041** — `id` inválido ou inexistente → 404 `bilhete_nao_encontrado`.

**REQ-042** — Bilhete `encerrado` ou `cancelado` → 409 `bilhete_nao_aberto`.

### UC6 — Histórico da placa · `GET /bilhetes?placa=XXX`

**REQ-050** — Todos os bilhetes da placa, qualquer status.

- CA1: status 200; array com a representação de cada bilhete conforme o status.
- CA2: só bilhetes daquela placa.
- CA3: ordem: `entrada` decrescente; empate por `id` decrescente.
- CA4: placa válida que nunca estacionou → `[]`.

**REQ-051** — Placa ausente ou inválida → 422 `placa_invalida`.

### Transversais

**REQ-060** — Erros no formato e com os códigos da seção 8, inclusive 404/405/500 genéricos do framework (registrar `errorhandler` em JSON).

**REQ-061** — Toda resposta tem `Content-Type: application/json`.

**REQ-062** — Escritas no repositório (criar, encerrar, cancelar) ocorrem sob um único `threading.Lock`; leitura do estado e mudança do estado ficam dentro do mesmo bloco `with`.

## 10. Regra de cálculo (UC2 + UC7)

Executar nesta ordem, só com `int`:

|Passo|Regra|
|---|---|
|1|`segundos = int((saida - entrada).total_seconds())`; `minutos = max(0, segundos // 60)`|
|2|Se `minutos <= TOLERANCIA_MINUTOS` (0) → `valor = 0` e fim|
|3|`fracoes = (minutos + FRACAO_MINUTOS - 1) // FRACAO_MINUTOS`|
|4|`valor = min(fracoes * VALOR_FRACAO_CENTAVOS, TETO_DIARIO_CENTAVOS)`|

`entrada` e `saida` já estão sem microssegundos (seção 4.1). Minutos são TRUNCADOS (segundos incompletos não contam); as frações são arredondadas para CIMA.

|`minutos`|Frações|`valor_centavos`|
|---|---|---|
|0|0|0|
|1|1|250|
|29|1|250|
|30|1|250|
|31|2|500|
|60|2|500|
|61|3|750|
|95|4|1000|
|120|4|1000|
|121|5|1250|
|570|19|4750|
|571|20|5000|
|601|21|5000 (teto)|
|4320|144|5000 (teto)|

**Atenção:** o exemplo do contrato (`95 min → 1250`) é de outra variante. Nesta variante, 95 min = 1000 centavos.

## 11. Requisitos não funcionais

**REQ-070** — `python app.py` sobe o servidor em `0.0.0.0:8003`, sem debug.

**REQ-071** — Nenhuma variável de ambiente é obrigatória; o container sobe só com `docker build` + `docker run -p 8003:8003`.

**REQ-072** — Fuso único `timezone(timedelta(hours=-3))`; nenhum `datetime` sem fuso existe no código. `agora()` é a única fonte de tempo.

**REQ-073** — Única dependência de produção: `flask>=3.0,<4`.

**REQ-074** — O log de acesso do Werkzeug fica em nível `ERROR` (`logging.getLogger("werkzeug").setLevel(logging.ERROR)`), para que a placa da query `GET /bilhetes?placa=` não seja registrada (PRI-10).

## 12. Decisões de interpretação

|ID|Ponto ambíguo no contrato|Decisão|
|---|---|---|
|D-01|Como contar minutos com segundos sobrando|Truncar (`// 60`); 30 min e 40 s = 30 min = 1 fração|
|D-02|Entrada no futuro|`minutos = 0`, `valor_centavos = 0`|
|D-03|Encerrar bilhete cancelado|409 `bilhete_ja_encerrado` (único 409 do UC2)|
|D-04|Quais bilhetes entram no relatório|Só `encerrado` com `saida` na data (fuso -03:00)|
|D-05|"Mais recentes primeiro"|`entrada` decrescente, empate por `id` decrescente|
|D-06|Entrada sem fuso|422 `entrada_invalida`|
|D-07|Placa minúscula ou com espaço|Inválida; sem normalização|
|D-08|Campo extra no corpo|Ignorado; não há código de erro no contrato para isso|
|D-09|Resposta do encerramento tem `status`?|Sim, `"encerrado"` (mesma representação do histórico)|
|D-10|`entrada: null`|Tratado como ausente|
|D-11|Teto em bilhete de vários dias|Teto por bilhete: máximo 5000 no total|
|D-12|Requisição rejeitada consome `id`?|Não|

## 13. Fora de escopo (PRI-02)

Autenticação, persistência em disco, paginação, edição de bilhete, `GET /bilhetes/{id}`, health check, CORS, OpenAPI, rotas de exportação ou eliminação de dados, configuração por variável de ambiente.

## 14. Aplicação da constituição nesta stack

|Princípio|Como se aplica aqui|
|---|---|
|PRI-06|Validação na borda feita por funções Python em `app.py` (a stack é Python, ver `plan.md`); nada chega a `regras.py`/`repositorio.py` sem validar|
|PRI-07|Não há atores nem recurso com dono; sem regra de autorização|
|PRI-08|Não há SQL|
|PRI-09|Não há segredos; módulo `config` não é criado; parâmetros da seção 3 são regras de domínio, não configuração|
|PRI-10|Placa nunca vai para log nem para mensagem de erro (REQ-074, seção 8)|
|PRI-11|Corpo de erro `{"erro": codigo}` do contrato; nunca stack|
|PRI-12|Não há autenticação nem permissão; sem eventos de segurança a registrar|

## 15. Rastreabilidade

| UC  | Endpoint                           | REQ              | TEST (ver `tests.md`) |
| --- | ---------------------------------- | ---------------- | --------------------- |
| UC1 | `POST /bilhetes`                   | 001–005          | 101–118               |
| UC8 | `POST /bilhetes`                   | 006–007          | 120–124               |
| UC2 | `POST /bilhetes/{id}/encerramento` | 010–015, 017     | 001–021, 130–137      |
| UC7 | (regra de valor)                   | 016              | 001–002, 138–139      |
| UC3 | `GET /bilhetes/ativos`             | 020              | 150–153               |
| UC4 | `GET /relatorios/diario`           | 030–033          | 030–034, 170–179      |
| UC5 | `POST /bilhetes/{id}/cancelamento` | 040–042          | 140–145               |
| UC6 | `GET /bilhetes`                    | 050–051          | 160–165               |
| —   | transversais                       | 060–062, 070–074 | 190–195               |
