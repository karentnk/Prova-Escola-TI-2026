# Tests — Cenários de teste (TDD)

Cada item abaixo vira um teste automatizado (`def test_...`) em
`tests/test_api.py` (rotas via `TestClient`) ou `tests/test_rules.py`
(funções puras). Os casos de borda são obrigatórios. Cada teste começa com o
repositório em memória vazio.

## Premissas

Variante: tarifa `400`/h, fração `15` min (`100` centavos), teto `6000`,
tolerância `15` min, porta `8001`.

**Como medir duração:** abrir com `entrada = 2026-10-12T08:00:00-03:00` e
encerrar com `saida` = entrada + N minutos no corpo do encerramento (ou
substituindo `clock.agora()`). Nos cenários abaixo, "N min" significa isso.

## UC1 — Abrir bilhete

| # | Cenário | Esperado | Tipo |
| --- | --- | --- | --- |
| T1 | Abrir `{"placa": "ABC1D23"}` | `201`, `id` inteiro, `status: "aberto"`, `entrada` termina em `-03:00` | feliz |
| T2 | Abrir com `entrada` `2026-10-12T08:30:00-03:00` | `201`, `entrada` igual à enviada | feliz |
| T3 | Dois bilhetes de placas diferentes | `id` 1 e 2, sequenciais | feliz |
| T4 | Placa com 6 caracteres `ABC1D2` | `422 placa_invalida` | borda |
| T5 | Placa com 8 caracteres `ABC1D234` | `422 placa_invalida` | borda |
| T6 | Placa minúscula `abc1d23` | `422 placa_invalida` | borda |
| T7 | Placa com caractere especial `ABC-123` | `422 placa_invalida` | borda |
| T8 | Corpo sem `placa` / `placa` numérica `1234567` (sem aspas) | `422 placa_invalida` | borda |
| T9 | Corpo que não é JSON | `422 placa_invalida` | borda |
| T10 | `entrada` `"ontem"` | `422 entrada_invalida` | borda |
| T11 | `entrada` `2026-13-45T08:00:00-03:00` (data impossível) | `422 entrada_invalida` | borda |
| T12 | `entrada` sem fuso `2026-10-12T08:30:00` | `201`, devolvida como `2026-10-12T08:30:00-03:00` | borda |
| T13 | `entrada` em UTC `2026-10-12T11:30:00Z` | `201`, devolvida como `2026-10-12T08:30:00-03:00` | borda |

## UC8 — Uma vaga por placa

| # | Cenário | Esperado | Tipo |
| --- | --- | --- | --- |
| T14 | Abrir `ABC1D23` duas vezes | 2ª chamada `409 bilhete_em_aberto` | borda |
| T15 | Abrir, encerrar e abrir de novo a mesma placa | `201` | borda |
| T16 | Abrir, cancelar e abrir de novo a mesma placa | `201` | borda |
| T17 | Placa ocupada e reenviada com `entrada` inválida | `422 entrada_invalida` (formato antes de conflito) | borda |
| T18 | Outra placa enquanto `ABC1D23` está aberta | `201` (não conflita) | borda |

## UC2 + UC7 — Valor (fração, tolerância e teto)

| # | Duração | Esperado `valor_centavos` | Tipo |
| --- | --- | --- | --- |
| T19 | 0 min | `0` | borda |
| T20 | 14 min (abaixo da tolerância) | `0` | borda |
| T21 | 15 min (tolerância exata) | `0` | borda |
| T22 | 16 min (1 acima da tolerância, cobra desde o minuto zero) | `200` | borda |
| T23 | 30 min (fração exata) | `200` | borda |
| T24 | 31 min (adjacência +1) | `300` | borda |
| T25 | 45 min | `300` | borda |
| T26 | 60 min (hora cheia) | `400` | feliz |
| T27 | 61 min | `500` | borda |
| T28 | 95 min | `700` | feliz |
| T29 | 899 min (abaixo do teto) | `6000` | borda |
| T30 | 900 min (teto exato) | `6000` | borda |
| T31 | 901 min (acima do teto) | `6000` | borda |
| T32 | 1440 min (24 h) | `6000` | borda |
| T33 | Tipo de `valor_centavos` em qualquer encerramento | inteiro, nunca decimal; campo `valor` não existe | borda |

## UC2 — Encerramento e minutos

| # | Cenário | Esperado | Tipo |
| --- | --- | --- | --- |
| T34 | Encerrar bilhete aberto | `200` com `id, placa, entrada, saida, minutos, valor_centavos` | feliz |
| T35 | Duração de 30 min e 59 s | `minutos: 30`, `valor_centavos: 200` (segundos truncados) | borda |
| T36 | Encerrar `id` inexistente `999` | `404 bilhete_nao_encontrado` | borda |
| T37 | Encerrar `id` não numérico `abc` | `404 bilhete_nao_encontrado` | borda |
| T38 | Encerrar duas vezes | 2ª chamada `409 bilhete_ja_encerrado` | borda |
| T39 | Encerrar bilhete cancelado | `409 bilhete_ja_encerrado` | borda |
| T40 | `saida` inválida no corpo | `422 entrada_invalida` | borda |

## UC4 — Relatório diário

| # | Cenário | Esperado | Tipo |
| --- | --- | --- | --- |
| T41 | 2 encerrados no dia com 10 e 11 min (média 10,5) | `tempo_medio_minutos: 11` (0,5 para cima) | borda |
| T42 | 2 encerrados com 10 e 10 min / 3 com 10, 10 e 11 (média 10,33) | `10` / `10` (abaixo de 0,5 desce) | borda |
| T43 | Dia com 2 encerrados (`200` e `300`), 1 aberto e 1 cancelado | `total_bilhetes: 4`, `faturamento_centavos: 500`, média só dos 2 encerrados | feliz |
| T44 | Dia sem bilhetes | `200`, `total_bilhetes: 0`, `faturamento_centavos: 0`, `tempo_medio_minutos: 0` | borda |
| T45 | Bilhete encerrado em outro dia | não entra no relatório do dia consultado | borda |
| T46 | `data` ausente | `422 data_invalida` | borda |
| T47 | `data` `2026-1-5` / `05/10/2026` | `422 data_invalida` | borda |
| T48 | `data` `2026-02-30` (inexistente) | `422 data_invalida` | borda |
| T49 | Campo `data` da resposta | igual ao valor consultado | feliz |

## UC3, UC5 e UC6 — Listagens e cancelamento

| # | Cenário | Esperado | Tipo |
| --- | --- | --- | --- |
| T50 | Ativos com 2 abertos (entradas 08:00 e 09:00) e 1 encerrado | `200`, só os 2 abertos, o de 09:00 primeiro | feliz |
| T51 | Ativos sem bilhetes abertos | `200`, `[]` | borda |
| T52 | Cancelar bilhete aberto | `200`, `status: "cancelado"`, sem `saida` e sem `valor_centavos` | feliz |
| T53 | Cancelado some de `/bilhetes/ativos` | não aparece | borda |
| T54 | Cancelar duas vezes / cancelar encerrado | `409 bilhete_nao_aberto` | borda |
| T55 | Cancelar `id` inexistente | `404 bilhete_nao_encontrado` | borda |
| T56 | Histórico com 1 encerrado, 1 cancelado e 1 aberto da mesma placa | `200`, os 3, do mais recente para o mais antigo | feliz |
| T57 | Histórico de placa válida que nunca estacionou | `200`, `[]` | borda |
| T58 | Histórico sem `placa` / com `placa=abc` | `422 placa_invalida` | borda |
| T59 | Histórico não traz bilhetes de outras placas | só a placa consultada | borda |

## Gerais

| # | Cenário | Esperado | Tipo |
| --- | --- | --- | --- |
| T60 | Qualquer erro da API | corpo com chave `erro`, nunca `detail` | borda |
| T61 | `GET /healthz` | `200 {"status": "ok"}` | feliz |