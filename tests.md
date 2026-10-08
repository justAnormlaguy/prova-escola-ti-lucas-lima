
Leitura: `constitution.md` → `spec.md` → `plan.md` → `tests.md` → `tasks.md`. Este arquivo define os TESTES. 

## 1. Como automatizar

|Item|Definição|
|---|---|
|Ferramenta|`pytest`, declarado em `requirements-dev.txt` (não entra na imagem de produção)|
|Arquivos|`tests/test_regras.py` (seções 2–3) e `tests/test_api.py` (seções 4–11)|
|Estado|Cada teste usa `criar_app()` novo: repositório vazio, contador em 1|
|Relógio|`agora` do módulo `app` substituído via `monkeypatch` por instante fixo|
|Instante padrão|`AGORA = 2026-10-12T15:00:00-03:00`|
|Notação|`AGORA−N` = string ISO de `AGORA` menos N minutos, ex.: `AGORA−95` = `"2026-10-12T13:25:00-03:00"`|

Fixture de referência (as rotas chamam `agora()` pelo nome global do módulo `app`, por isso o `monkeypatch` funciona):

```python
FUSO = timezone(timedelta(hours=-3))
AGORA = datetime(2026, 10, 12, 15, 0, 0, tzinfo=FUSO)

@pytest.fixture
def relogio(monkeypatch):
    estado = {"instante": AGORA}
    monkeypatch.setattr(app_modulo, "agora", lambda: estado["instante"])
    return estado

@pytest.fixture
def cliente(relogio):
    return app_modulo.criar_app().test_client()
```

Para mudar o relógio no meio do teste: `relogio["instante"] = novo_datetime`.

---

## 2. Regra de valor — `regras.calcular_valor(minutos) -> int`

|ID|REQ|Entrada (`minutos`)|Esperado|Nota|
|---|---|---|---|---|
|TEST-001|016|0|0|**Borda** tolerância 0: duração ≤ tolerância|
|TEST-002|016|1|250|**Borda** passou da tolerância: cobra desde o minuto zero|
|TEST-003|014|29|250||
|TEST-004|014|30|250|**Borda** fração exata cobra 1 fração|
|TEST-005|014|31|500|**Borda** 1 minuto a mais cobra a fração seguinte|
|TEST-006|014|60|500|Hora cheia = tarifa 500|
|TEST-007|014|61|750||
|TEST-008|014|95|1000|Não é 1250 (valor de outra variante)|
|TEST-009|014|120|1000||
|TEST-010|014|121|1250||
|TEST-011|015|570|4750|Última fração abaixo do teto|
|TEST-012|015|571|5000|**Borda** atinge o teto exatamente|
|TEST-013|015|601|5000|**Borda** passaria do teto (5250)|
|TEST-014|015, 017|4320|5000|3 dias; resultado é `int` (`type(...) is int`)|

## 3. Regras puras de tempo

### 3.1 `regras.calcular_minutos(entrada, saida) -> int`

Datas abaixo em `-03:00` salvo indicação.

|ID|REQ|Entrada|Saída|Esperado|Nota|
|---|---|---|---|---|---|
|TEST-015|013|08:30:00|08:30:00|0||
|TEST-016|013|08:30:00|08:30:59|0|**Borda** segundos incompletos são truncados (D-01)|
|TEST-017|013|08:30:00|08:31:00|1||
|TEST-018|013|08:30:00|10:05:00|95||
|TEST-019|013|`11:30:00Z`|08:31:00|1|**Borda** offsets diferentes|
|TEST-020|013|10:00:00|09:00:00|0|**Borda** saída antes da entrada (D-02)|
|TEST-021|013|12/10 23:50:00|13/10 00:20:00|30|Vira o dia|

### 3.2 `regras.calcular_tempo_medio(lista_minutos) -> int`

|ID|REQ|Entrada|Esperado|Nota|
|---|---|---|---|---|
|TEST-030|033|`[]`|0|**Borda** sem divisão por zero|
|TEST-031|031|`[30, 61]`|46|**Borda** 45,5 arredonda para cima|
|TEST-032|031|`[10, 11]`|11|**Borda** 10,5 → 11 (`round()` daria 10)|
|TEST-033|031|`[10, 10, 11]`|10|10,33 → 10|
|TEST-034|031|`[1, 2, 2]`|2|1,67 → 2|

---

## 4. UC1 — Abrir bilhete · `POST /bilhetes`

|ID|REQ|Corpo enviado|Status|Corpo esperado / verificação|
|---|---|---|---|---|
|TEST-101|001, 002|`{"placa":"ABC1D23"}`|201|`{"id":1,"placa":"ABC1D23","entrada":"2026-10-12T15:00:00-03:00","status":"aberto"}`, exatamente 4 chaves|
|TEST-102|001|após TEST-101, `{"placa":"XYZ9A87"}`|201|`id == 2`|
|TEST-103|003|`{"placa":"ABC1D23","entrada":"2026-10-12T11:30:00Z"}`|201|`entrada == "2026-10-12T08:30:00-03:00"`|
|TEST-104|003|entrada `"2026-10-12T08:30:00.987-03:00"`|201|`entrada == "2026-10-12T08:30:00-03:00"` (**Borda** microssegundos)|
|TEST-105|002|`{"placa":"ABC1D23","entrada":null}`|201|`entrada == "2026-10-12T15:00:00-03:00"` (D-10)|
|TEST-106|001|`{"placa":"ABC1D23","cor":"azul"}`|201|resposta sem chave `cor` (D-08)|
|TEST-107|004|`{}`|422|`{"erro":"placa_invalida"}`|
|TEST-108|004|`{"placa":"abc1d23"}`|422|`placa_invalida` (**Borda** minúscula, D-07)|
|TEST-109|004|`{"placa":"ABC1D2"}`|422|`placa_invalida` (**Borda** 6 caracteres)|
|TEST-110|004|`{"placa":"ABC1D234"}`|422|`placa_invalida` (**Borda** 8 caracteres)|
|TEST-111|004|`{"placa":"ABC-1D2"}` e `{"placa":" ABC1D2"}`|422|`placa_invalida`|
|TEST-112|004|`{"placa":1234567}`|422|`placa_invalida` (não é texto)|
|TEST-113|004|corpo bruto `abc` (JSON inválido)|422|`placa_invalida`|
|TEST-114|004|`["ABC1D23"]`|422|`placa_invalida`|
|TEST-115|005|entrada `"2026-10-12T08:30:00"`|422|`entrada_invalida` (**Borda** sem fuso, D-06)|
|TEST-116|005|entrada `"ontem"`|422|`entrada_invalida`|
|TEST-117|005|entrada `123`|422|`entrada_invalida`|
|TEST-118|004, 005|`{"placa":"abc","entrada":"ontem"}`|422|`placa_invalida` (precedência: placa antes de entrada)|
|TEST-119|001|três requisições 422 e depois `{"placa":"ABC1D23"}`|201|`id == 1` (D-12: rejeição não consome `id`)|

## 5. UC8 — Placa duplicada

|ID|REQ|Passos|Esperado|
|---|---|---|---|
|TEST-120|006|abrir `ABC1D23`; abrir `ABC1D23` de novo|2ª: 409 `{"erro":"bilhete_em_aberto"}`|
|TEST-121|006|TEST-120 e depois abrir `XYZ9A87`|201 com `id == 2` (o 409 não consumiu `id`)|
|TEST-122|006|abrir `ABC1D23`; abrir `ABC1D23` com entrada `"ontem"`|422 `entrada_invalida`, não 409 (**Borda** precedência 422 > 409)|
|TEST-123|007|abrir → encerrar → abrir `ABC1D23`|201 com `id == 2`|
|TEST-124|007|abrir → cancelar → abrir `ABC1D23`|201 com `id == 2`|

## 6. UC2 — Encerrar · `POST /bilhetes/{id}/encerramento`

|ID|REQ|Preparação|Status|Esperado|
|---|---|---|---|---|
|TEST-130|010, 013, 014, 017|abrir com entrada `AGORA−95`|200|`minutos == 95`, `valor_centavos == 1000`, `saida == "2026-10-12T15:00:00-03:00"`, `status == "encerrado"`, exatamente 7 chaves, sem chave `valor`|
|TEST-131|014|entrada `AGORA−30`|200|`valor_centavos == 250` (**Borda** fração exata)|
|TEST-132|014|entrada `AGORA−31`|200|`valor_centavos == 500` (**Borda** 1 min a mais)|
|TEST-133|015|entrada `"2026-10-09T15:00:00-03:00"`|200|`minutos == 4320`, `valor_centavos == 5000` (**Borda** teto, D-11)|
|TEST-134|013|entrada `"2026-10-12T16:00:00-03:00"` (futuro)|200|`minutos == 0`, `valor_centavos == 0` (D-02)|
|TEST-135|011|nenhum bilhete; `POST /bilhetes/999/encerramento`|404|`{"erro":"bilhete_nao_encontrado"}`|
|TEST-136|011|`POST /bilhetes/abc/encerramento`|404|`{"erro":"bilhete_nao_encontrado"}` em JSON, não HTML|
|TEST-137|012|encerrar o mesmo bilhete duas vezes|409|`{"erro":"bilhete_ja_encerrado"}`; histórico mostra valores do 1º encerramento|

## 7. UC7 — Tolerância (via API)

|ID|REQ|Preparação|Esperado|
|---|---|---|---|
|TEST-138|016|abrir sem entrada e encerrar no mesmo instante|`minutos == 0`, `valor_centavos == 0` (**Borda** igual à tolerância)|
|TEST-139|016|entrada `AGORA−1`|`minutos == 1`, `valor_centavos == 250` (**Borda** tolerância não é descontada)|

## 8. UC5 — Cancelar · `POST /bilhetes/{id}/cancelamento`

|ID|REQ|Preparação|Status|Esperado|
|---|---|---|---|---|
|TEST-140|040|abrir `ABC1D23`; cancelar|200|`{"id":1,"placa":"ABC1D23","entrada":"2026-10-12T15:00:00-03:00","status":"cancelado"}`, sem `saida` nem `valor_centavos`|
|TEST-141|040|após TEST-140|—|`/bilhetes/ativos` → `[]`; `/bilhetes?placa=ABC1D23` contém o bilhete com `status == "cancelado"`|
|TEST-142|041|`POST /bilhetes/999/cancelamento` e `/bilhetes/abc/cancelamento`|404|`bilhete_nao_encontrado`|
|TEST-143|042|cancelar duas vezes|409|`{"erro":"bilhete_nao_aberto"}` (**Borda**)|
|TEST-144|042|encerrar e depois cancelar|409|`bilhete_nao_aberto`; bilhete continua `encerrado`|
|TEST-145|012|cancelar e depois encerrar|409|`bilhete_ja_encerrado` (D-03); bilhete continua `cancelado`|

## 9. UC3 — Ativos · `GET /bilhetes/ativos`

|ID|REQ|Preparação|Esperado|
|---|---|---|---|
|TEST-150|020|nenhum bilhete|200, `[]` (**Borda**)|
|TEST-151|020|A entrada 08:00, B 10:00, C 09:00 (placas distintas)|ids na ordem `[B, C, A]`|
|TEST-152|020|dois bilhetes com a mesma entrada, ids 1 e 2|ordem `[2, 1]` (**Borda** empate, D-05)|
|TEST-153|020|3 abertos; encerrar 1; cancelar 1|só 1 item, `status == "aberto"`, 4 chaves|

## 10. UC6 — Histórico · `GET /bilhetes?placa=`

|ID|REQ|Preparação / requisição|Esperado|
|---|---|---|---|
|TEST-160|050|`?placa=ABC1D23` sem bilhetes|200, `[]` (**Borda**)|
|TEST-161|050|`ABC1D23`: um encerrado, um cancelado, um aberto|3 itens; cada um com as chaves do seu status (spec seção 5)|
|TEST-162|050|entradas 08:00, 10:00, 09:00 (abrir/encerrar em sequência)|ordem por `entrada` decrescente|
|TEST-163|050|bilhetes de `ABC1D23` e `XYZ9A87`; consultar `ABC1D23`|nenhum item de `XYZ9A87`|
|TEST-164|051|`GET /bilhetes` sem query|422 `placa_invalida`|
|TEST-165|051|`?placa=abc1d23`|422 `placa_invalida`|

## 11. UC4 — Relatório · `GET /relatorios/diario?data=`

|ID|REQ|Preparação / requisição|Esperado|
|---|---|---|---|
|TEST-170|030, 033|nenhum bilhete; `?data=2026-10-12`|`{"data":"2026-10-12","total_bilhetes":0,"faturamento_centavos":0,"tempo_medio_minutos":0}` (**Borda**)|
|TEST-171|030, 031|encerrar bilhetes com entrada `AGORA−30` e `AGORA−61`|`total_bilhetes 2`, `faturamento_centavos 1000`, `tempo_medio_minutos 46`|
|TEST-172|030|TEST-171 + um aberto + um cancelado|mesmos valores de TEST-171 (**Borda** D-04)|
|TEST-173|030|entrada `2026-10-11T23:00:00-03:00`, relógio `2026-10-12T01:00:00-03:00`, encerrar|conta em `2026-10-12` (120 min, 1000); `2026-10-11` devolve zeros|
|TEST-174|030|relógio `2026-10-12T23:30:00-03:00` (já 13/10 em UTC), encerrar|conta em `2026-10-12`, não em `2026-10-13` (**Borda** fuso)|
|TEST-175|032|sem `data`|422 `data_invalida`|
|TEST-176|032|`?data=05/10/2026`|422 `data_invalida`|
|TEST-177|032|`?data=2026-13-01`|422 `data_invalida`|
|TEST-178|032|`?data=2026-02-30`|422 `data_invalida` (**Borda** formato certo, data inexistente)|
|TEST-179|032|`?data=2026-1-5`|422 `data_invalida`|

## 12. Transversais

|ID|REQ|Verificação|Esperado|
|---|---|---|---|
|TEST-190|060|`GET /nao-existe`|404 `{"erro":"rota_nao_encontrada"}` em JSON|
|TEST-191|060|`DELETE /bilhetes`|405 `{"erro":"metodo_nao_permitido"}`|
|TEST-192|061|respostas de TEST-101, 107, 135, 170, 190|`Content-Type` começa com `application/json`|
|TEST-193|060|todos os corpos de erro acima|exatamente uma chave, `erro`; sem stack trace|
|TEST-194|006, 062|20 threads abrem `ABC1D23` ao mesmo tempo (um `test_client` por thread, mesmo app)|exatamente 1 resposta 201 e 19 respostas 409|
|TEST-195|070, 071|manual: `docker build -t zona .` e `docker run -p 8003:8003 zona`; `curl -X POST localhost:8003/bilhetes -d '{"placa":"ABC1D23"}'`|201, sem nenhuma variável de ambiente|

---

## 13. Cobertura mínima por regra de negócio

| Regra                            | Casos de borda          |
| -------------------------------- | ----------------------- |
| Fração arredondada para cima     | TEST-004, 005, 131, 132 |
| Teto por bilhete                 | TEST-012, 013, 133      |
| Tolerância (0)                   | TEST-001, 002, 138, 139 |
| Minutos truncados                | TEST-016, 019, 020      |
| Centavos inteiros                | TEST-014, 130           |
| Uma placa, um aberto             | TEST-120, 122, 194      |
| Reabertura após fim              | TEST-123, 124           |
| Precedência 422 > 409            | TEST-118, 122           |
| Estados finais                   | TEST-137, 143, 144, 145 |
| Ordenação                        | TEST-151, 152, 162      |
| Média arredondando 0,5 para cima | TEST-031, 032           |
| Relatório por dia de saída       | TEST-172, 173, 174      |
| Validação de data                | TEST-175 a 179          |
