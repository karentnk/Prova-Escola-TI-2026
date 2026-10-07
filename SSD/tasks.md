# Tasks — Decomposição

## Contrato de referência

Base URL `http://localhost:8001`. Erro sempre `{"erro": "<codigo>"}`.

| UC | Rota | Sucesso | Erros |
| --- | --- | --- | --- |
| UC1 | `POST /bilhetes` body `{"placa", "entrada"?}` | `201 {id, placa, entrada, status: "aberto"}` | `422 placa_invalida`, `422 entrada_invalida`, `409 bilhete_em_aberto` |
| UC2 | `POST /bilhetes/{id}/encerramento` body `{"saida"?}` | `200 {id, placa, entrada, saida, minutos, valor_centavos}` | `404 bilhete_nao_encontrado`, `409 bilhete_ja_encerrado` |
| UC3 | `GET /bilhetes/ativos` | `200` array de abertos, mais recentes primeiro | — |
| UC4 | `GET /relatorios/diario?data=AAAA-MM-DD` | `200 {data, total_bilhetes, faturamento_centavos, tempo_medio_minutos}` | `422 data_invalida` |
| UC5 | `POST /bilhetes/{id}/cancelamento` | `200 {id, placa, entrada, status: "cancelado"}` | `404 bilhete_nao_encontrado`, `409 bilhete_nao_aberto` |
| UC6 | `GET /bilhetes?placa=` | `200` array da placa, mais recentes primeiro | `422 placa_invalida` |

Variante: tarifa `400`/h, fração `15` min (`100` centavos), teto `6000`,
tolerância `15` min (até 15 = grátis; 16+ cobra desde o minuto zero), porta
`8001` (e também `8080`). Python 3.12 + FastAPI + Pytest, persistência em memória.

## Tarefas

- [ ] TK1 — Scaffolding: pastas `app/` e `tests/`, `app/__init__.py`, `requirements.txt` com versões fixadas (fastapi, uvicorn, pytest, httpx, ruff), `pyproject.toml` com config do ruff e do pytest, `.gitignore` e `.dockerignore`
- [ ] TK2 — Configuração e relógio: `app/config.py` com as constantes da variante (env opcional com default) e `app/clock.py` com `agora()` substituível nos testes
- [ ] TK3 — Modelos e repositório em memória: `app/models.py` (bilhete e status `aberto`/`encerrado`/`cancelado`) e `app/store.py` (dicionário por `id`, lock, id sequencial a partir de 1)
- [ ] TK4 — Regras puras em `app/rules.py`: validação de placa, de ISO-8601 (sem fuso = `-03:00`) e de `AAAA-MM-DD`; minutos truncados; valor com tolerância, fração para cima e teto; média half-up com inteiros, sem `round()` (T19–T33, T35, T41–T42)
- [ ] TK5 — Formato de erro: handlers para que toda resposta de erro, inclusive 404/405 do framework, seja `{"erro": ...}` e nunca `{"detail": ...}` (T60)
- [ ] TK6 — UC1 e UC8: abrir bilhete, validando corpo → placa → entrada → conflito, nessa ordem (T1–T18)
- [ ] TK7 — UC2 e UC7: encerrar bilhete com `saida` opcional, minutos e valor (T19–T40)
- [ ] TK8 — UC5: cancelar bilhete aberto, sem `saida` nem `valor_centavos` (T52–T55)
- [ ] TK9 — UC3 e UC6: listar ativos e histórico por placa, ordenados por `entrada` decrescente e depois `id` decrescente (T50–T51, T56–T59)
- [ ] TK10 — UC4: relatório diário com `total_bilhetes`, `faturamento_centavos` e `tempo_medio_minutos` (T41–T49)
- [ ] TK11 — Servidor: `GET /healthz` e inicialização de dois Uvicorn (portas 8001 e 8080, host `0.0.0.0`) no mesmo processo via `python -m app.main` (T61)
- [ ] TK12 — Testes próprios: `tests/test_rules.py` e `tests/test_api.py` com uma função `def test_` para cada cenário T1–T61 do `tests.md`
- [ ] TK13 — Container: `Dockerfile` e `Containerfile` idênticos (`python:3.12-slim`, usuário não-root, `EXPOSE 8001 8080`, `CMD ["python", "-m", "app.main"]`)
- [ ] TK14 — README: como instalar, rodar local, rodar testes (`pytest`), rodar linter (`ruff check .`) e rodar com Docker (`docker build -t zona-azul .` e `docker run -p 8001:8001 zona-azul`)
- [ ] TK15 — Verificação final: suíte completa passando, `ruff check .` sem erros, nenhum segredo no repositório e cada critério de aceite do `spec.md` conferido