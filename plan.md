# Plano técnico

Leitura: `constitution.md` → `spec.md` → `plan.md` → `tests.md` → `tasks.md`.
Este plano define COMO implementar o que o `spec.md` define.

## Stack e decisões

| Decisão      | Escolha                                                                        | Justificativa                                                                                           | Descartado                                                 |
| ------------ | ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| Linguagem    | Python 3.12                                                                    | `int` de precisão arbitrária para centavos; `datetime.fromisoformat` aceita offsets como `-03:00` e `Z` | Node (números são float; fuso exige biblioteca)            |
| Framework    | Flask 3.x                                                                      | Sem validação automática: todo erro segue o formato `{"erro": ...}` do contrato                         | FastAPI (devolve 422 no formato próprio `{"detail": ...}`) |
| Persistência | Memória: `dict` por `id` + contador sequencial iniciando em 1                  | Contrato não exige durabilidade; menos camadas, menos falhas                                            | SQLite (camada sem benefício)                              |
| Concorrência | Um `threading.Lock` em toda escrita                                            | Evita dois bilhetes abertos para a mesma placa (UC8) sob requisições paralelas                          | —                                                          |
| Fuso         | Offset fixo `timezone(timedelta(hours=-3))`                                    | Brasil sem horário de verão desde 2019; dispensa `tzdata`, ausente na imagem slim                       | `zoneinfo("America/Sao_Paulo")`                            |
| Relógio      | Uma única função `agora()` que devolve datetime com fuso, truncado em segundos | Ponto único de tempo; container roda em UTC e não pode haver datetime sem fuso                          | `datetime.now()` sem fuso                                  |
| Dinheiro     | Centavos `int` em todo o cálculo; nunca `float`                                | `0.1 + 0.2 != 0.3`: inteiros eliminam erro de arredondamento                                            | Decimal (desnecessário)                                    |
| Porta        | Escutar em `0.0.0.0:8003` por padrão                                           | A suíte chama `localhost:8003`; bind em `127.0.0.1` dentro do container fica inacessível                | Porta interna 8080 do JSON                                 |

## Estrutura de módulos

| Arquivo            | Responsabilidade                                                                          |
| ------------------ | ----------------------------------------------------------------------------------------- |
| `app.py`           | Rotas Flask, validação de entrada, montagem das respostas e erros                         |
| `regras.py`        | Funções puras: `calcular_minutos`, `calcular_valor`, `tempo_medio`; sem Flask, sem estado |
| `repositorio.py`   | Armazenamento em memória, contador de `id`, lock                                          |
| `requirements.txt` | `flask>=3.0,<4`                                                                           |
| `Dockerfile`       | Imagem `python:3.12-slim`                                                                 |
| `pyproject.toml`   | Configuração do Ruff (seção Qualidade)                                                    |

## Detalhes de implementação obrigatórios

- Ler o body com `request.get_json(silent=True)`: JSON inválido vira `None` e gera 422 `placa_invalida`.
- Declarar o `id` das rotas como texto e validar com `isdigit()`. Não numérico gera 404 `bilhete_nao_encontrado`, e não o 404 em HTML do `<int:id>`.
- Tempo médio com conta inteira `(2 * soma + n) // (2 * n)`, nunca `round()`, que arredonda 0,5 para o par.
- Datas de saída no formato `AAAA-MM-DDTHH:MM:SS-03:00`, sem microssegundos.

## Container

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8003
CMD ["python", "app.py"]
```

Execução local: `pip install -r requirements.txt && python app.py`, servindo em `http://localhost:8003`.

## Qualidade

Limites definidos no `constitution.md`, verificados com esta configuração:

```toml
[tool.ruff]
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "C90", "PLR"]

[tool.ruff.lint.mccabe]
max-complexity = 8

[tool.ruff.lint.pylint]
max-statements = 25
max-branches = 10
max-args = 5
max-returns = 6
```
