# Plan — Arquitetura e decisões

## Contrato que o plano precisa respeitar

Base URL `http://localhost:8001`. Todo erro: `{"erro": "<codigo>"}`. Campos em
português, minúsculos, com `_`.

| UC | Método | Rota | Sucesso | Erros |
| --- | --- | --- | --- | --- |
| UC1 | `POST` | `/bilhetes` body `{"placa", "entrada"?}` | `201 {id, placa, entrada, status: "aberto"}` | `422 placa_invalida`, `422 entrada_invalida`, `409 bilhete_em_aberto` |
| UC2 | `POST` | `/bilhetes/{id}/encerramento` body `{"saida"?}` | `200 {id, placa, entrada, saida, minutos, valor_centavos, status: "encerrado"}` | `404 bilhete_nao_encontrado`, `409 bilhete_ja_encerrado` |
| UC3 | `GET` | `/bilhetes/ativos` | `200` array de abertos, mais recentes primeiro | — |
| UC4 | `GET` | `/relatorios/diario?data=AAAA-MM-DD` | `200 {data, total_bilhetes, faturamento_centavos, tempo_medio_minutos}` | `422 data_invalida` |
| UC5 | `POST` | `/bilhetes/{id}/cancelamento` | `200 {id, placa, entrada, status: "cancelado"}` | `404 bilhete_nao_encontrado`, `409 bilhete_nao_aberto` |
| UC6 | `GET` | `/bilhetes?placa=` | `200` array da placa, qualquer status, mais recentes primeiro | `422 placa_invalida` |

Variante: `TARIFA_HORA_CENTAVOS = 400`, `FRACAO_MINUTOS = 15`,
`TETO_DIARIO_CENTAVOS = 6000`, `TOLERANCIA_MINUTOS = 15`, `PORTA_SERVICO = 8001`.

## Stack

| Item | Escolha | Justificativa |
| --- | --- | --- |
| Linguagem | Python 3.12 | Tipagem por *hints*, `datetime.fromisoformat` aceita ISO-8601 com fuso nativamente |
| Framework | FastAPI + Uvicorn | Rotas REST declarativas, `TestClient` para testes sem subir servidor |
| Persistência | Em memória (dicionário por `id`) com `threading.Lock` | O enunciado não exige banco; evita dependência externa e variável de ambiente obrigatória |
| Testes | Pytest + `httpx` (`TestClient`) | Padrão da comunidade; `httpx` é exigido pelo `TestClient` |
| Linter | Ruff | Linter genérico do ecossistema Python, rápido e com config simples no `pyproject.toml` |
| Manifesto | `requirements.txt` com versões fixadas | Build reprodutível no container |

## Estrutura de arquivos a gerar

```
app/__init__.py    # marca app/ como pacote Python (vazio)
app/config.py      # constantes da variante (lidas de env com default)
app/clock.py       # função agora() substituível nos testes
app/models.py      # dataclass Bilhete e enum de status
app/rules.py       # funções puras: validar placa/data, minutos, valor, média
app/store.py       # repositório em memória com lock e id sequencial
app/main.py        # rotas FastAPI e inicialização nas portas 8001 e 8080
tests/test_rules.py  # testes unitários das funções de rules.py
tests/test_api.py    # testes das rotas via TestClient (cenários do tests.md)
requirements.txt   # fastapi, uvicorn, pytest, httpx e ruff com versões fixas
pyproject.toml     # configuração do ruff e do pytest
Dockerfile         # imagem da API: EXPOSE 8001 8080 e CMD (decisão 13)
Containerfile      # cópia idêntica do Dockerfile (Podman)
README.md          # como rodar local, rodar testes e rodar com Docker
.gitignore         # ignora __pycache__/, .venv/, .pytest_cache/, .env
.dockerignore      # mesmo conteúdo, para não copiar lixo para a imagem
```

## Decisões

1. **Dinheiro em centavos inteiros.** Justificativa: ponto flutuante acumula
   erro (0.1 + 0.2 ≠ 0.3); inteiros eliminam a classe de bug e o contrato exige
   `valor_centavos` inteiro.
2. **Cálculo do valor** em `rules.py`, só com inteiros:
   - se `minutos <= 15` → `0` (tolerância, sem desconto depois);
   - senão `fracoes = (minutos + 14) // 15` (teto da divisão sem `float`);
   - `valor = fracoes * 400 * 15 // 60` (= `fracoes * 100`);
   - resultado final `min(valor, 6000)`.
   Justificativa: divisão inteira com teto evita `math.ceil` sobre `float`.
3. **Minutos truncados:** `minutos = int((saida - entrada).total_seconds()) // 60`,
   mínimo `0`. Justificativa: a suíte envia `entrada` no passado e encerra
   "agora"; milissegundos extras não podem virar uma fração a mais.
4. **Tempo médio half-up com inteiros:** `(2 * soma + n) // (2 * n)`, e `0`
   quando `n = 0`. Justificativa: o `round()` do Python arredonda 0,5 para o par
   (10,5 → 10), o que viola o contrato (10,5 → 11).
5. **Validação manual do corpo**, sem modelos Pydantic nas rotas: ler o corpo
   bruto e validar em `rules.py`. Justificativa: a validação automática do
   FastAPI responde `422 {"detail": ...}`, formato que o contrato não aceita.
   Também registrar *handlers* para que 404/405 do framework respondam no
   formato `{"erro": ...}`.
6. **Ordem de validação no UC1:** corpo JSON → `placa` → `entrada` → conflito
   de placa aberta. Justificativa: o contrato manda `422` antes de `409`. Se
   placa e entrada forem inválidas juntas, responde `placa_invalida`.
7. **Placa:** expressão `^[A-Z0-9]{7}$` sobre valor do tipo string, sem
   converter para maiúsculas. Justificativa: o contrato exige maiúsculos.
8. **Datas:** `entrada` e `saida` via `datetime.fromisoformat`; sem fuso,
   assume `-03:00`; com outro fuso, converte para `-03:00`. Saída serializada
   como `AAAA-MM-DDTHH:MM:SS-03:00`, sem microssegundos. A `data` do relatório
   exige a expressão `^\d{4}-\d{2}-\d{2}$` e data de calendário válida.
9. **Relógio injetável** (`clock.agora()`): todo "agora" passa por essa função,
   que os testes substituem. Justificativa: testar fração, teto e tolerância
   sem esperar tempo real.
10. **IDs sequenciais** a partir de 1, gerados no repositório sob o lock,
    nunca reutilizados.
11. **Ordenação "mais recentes primeiro"** por `entrada` decrescente, empate
    por `id` decrescente.
12. **Duas portas, um processo:** `app/main.py` sobe dois servidores Uvicorn
    (portas `8001` e `8080`, host `0.0.0.0`) no mesmo laço `asyncio`, usando a
    mesma instância da aplicação. Justificativa: o contrato cita a porta
    interna `8080` e também `PORTA_SERVICO = 8001`; escutar nas duas atende
    qualquer mapeamento da suíte, e o mesmo processo garante estado único.
13. **Container:** `python:3.12-slim`, `pip install --no-cache-dir -r requirements.txt`,
    usuário não-root, `EXPOSE 8001 8080` e `CMD ["python", "-m", "app.main"]`.
    Justificativa: imagem pequena, sem privilégios desnecessários.
14. **Relatório:** `total_bilhetes` conta bilhetes com `entrada` na data
    (qualquer status); `faturamento_centavos` e `tempo_medio_minutos` usam só
    bilhetes `encerrado` com `saida` na data, sempre no fuso `-03:00`.