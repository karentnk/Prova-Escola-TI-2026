# Spec — Zona Azul Digital

A operadora de estacionamento rotativo precisa de uma API para controlar os
bilhetes por placa: abrir, encerrar com cobrança, cancelar, ver quem está
estacionado agora, ver o histórico de uma placa e tirar um relatório do dia.
Só a API faz parte do escopo; não existe back-office.

## Contrato resumido

Base URL: `http://localhost:8001`. Corpo de erro sempre `{"erro": "<codigo>"}`.
Campos e códigos em português, minúsculos, exatamente como abaixo.

| UC | Método | Rota | Sucesso | Erros |
| --- | --- | --- | --- | --- |
| UC1 | `POST` | `/bilhetes` | `201` | `422 placa_invalida`, `422 entrada_invalida`, `409 bilhete_em_aberto` |
| UC2 | `POST` | `/bilhetes/{id}/encerramento` | `200` | `404 bilhete_nao_encontrado`, `409 bilhete_ja_encerrado` |
| UC3 | `GET` | `/bilhetes/ativos` | `200` | — |
| UC4 | `GET` | `/relatorios/diario?data=AAAA-MM-DD` | `200` | `422 data_invalida` |
| UC5 | `POST` | `/bilhetes/{id}/cancelamento` | `200` | `404 bilhete_nao_encontrado`, `409 bilhete_nao_aberto` |
| UC6 | `GET` | `/bilhetes?placa=ABC1D23` | `200` | `422 placa_invalida` |

Parâmetros: `TARIFA_HORA_CENTAVOS = 400`, `FRACAO_MINUTOS = 15` (fração vale
`100` centavos), `TETO_DIARIO_CENTAVOS = 6000`, `TOLERANCIA_MINUTOS = 15`,
`PORTA_SERVICO = 8001`.

## Representação do bilhete

| Campo | Tipo | Presente quando |
| --- | --- | --- |
| `id` | inteiro, sequencial a partir de 1, nunca reutilizado | sempre |
| `placa` | string de 7 caracteres | sempre |
| `entrada` | ISO-8601 `AAAA-MM-DDTHH:MM:SS-03:00` | sempre |
| `status` | `aberto` \| `encerrado` \| `cancelado` | sempre |
| `saida` | ISO-8601 `-03:00` | somente `encerrado` |
| `minutos` | inteiro | somente `encerrado` |
| `valor_centavos` | inteiro (nunca decimal) | somente `encerrado` |

## Casos de uso

**UC1 — Abrir bilhete** (`POST /bilhetes`)
- Entrada: `placa` (string, obrigatória, exatamente 7 caracteres `A-Z` ou `0-9`, maiúsculos); `entrada` (opcional, ISO-8601).
- Critérios de aceite:
  - Quando válido, o sistema deve retornar `201` com `{"id", "placa", "entrada", "status": "aberto"}`.
  - Quando `entrada` não for enviada, o sistema deve usar o horário atual em `-03:00`.
  - Quando `entrada` for enviada, o bilhete deve abrir exatamente naquele instante, devolvido convertido para `-03:00`.
  - Quando `placa` estiver ausente, não for string, tiver tamanho diferente de 7, contiver minúscula ou caractere especial, o sistema deve retornar `422 {"erro": "placa_invalida"}`.
  - Quando `entrada` estiver presente mas não for ISO-8601 válido, o sistema deve retornar `422 {"erro": "entrada_invalida"}`.
  - Quando o corpo não for JSON válido ou não for um objeto, o sistema deve retornar `422 {"erro": "placa_invalida"}`.

**UC2 — Encerrar bilhete** (`POST /bilhetes/{id}/encerramento`)
- Entrada: `id` na rota; corpo opcional `{"saida": "<ISO-8601>"}` (ver decisão D5).
- Critérios de aceite:
  - Quando o bilhete estiver aberto, o sistema deve retornar `200` com `{"id", "placa", "entrada", "saida", "minutos", "valor_centavos"}` (e `"status": "encerrado"`).
  - `minutos` deve ser os minutos inteiros decorridos entre `entrada` e `saida`, **descartando os segundos** (95 min e 59 s → `95`).
  - Valor: até 15 minutos → `0`; acima de 15 → frações de 15 min arredondadas **para cima**, × `100` centavos, limitado a `6000`.
  - Fração exata cobra a própria fração (30 min → `200`); 1 minuto a mais cobra a seguinte (31 min → `300`).
  - `valor_centavos` deve ser sempre inteiro, nunca decimal.
  - Quando o `id` não existir (ou não for inteiro), o sistema deve retornar `404 {"erro": "bilhete_nao_encontrado"}`.
  - Quando o bilhete já estiver encerrado ou cancelado, o sistema deve retornar `409 {"erro": "bilhete_ja_encerrado"}`.

**UC3 — Listar ativos** (`GET /bilhetes/ativos`)
- Critérios de aceite:
  - O sistema deve retornar `200` com array contendo **somente** bilhetes `aberto`.
  - A ordem deve ser do mais recente para o mais antigo por `entrada`; empate, maior `id` primeiro.
  - Sem bilhetes abertos, deve retornar `[]`.

**UC4 — Relatório diário** (`GET /relatorios/diario?data=AAAA-MM-DD`)
- Entrada: `data` (obrigatória, formato `AAAA-MM-DD`, data de calendário válida).
- Critérios de aceite:
  - O sistema deve retornar `200` com `{"data", "total_bilhetes", "faturamento_centavos", "tempo_medio_minutos"}`, onde `data` repete o valor consultado.
  - `total_bilhetes`: quantidade de bilhetes com `entrada` na data (fuso `-03:00`), em qualquer status.
  - `faturamento_centavos`: soma dos `valor_centavos` dos bilhetes **encerrados** com `saida` na data.
  - `tempo_medio_minutos`: média dos `minutos` **apenas dos bilhetes encerrados** com `saida` na data, arredondando **0,5 para cima** (10,5 → `11`; 10,4 → `10`). Abertos e cancelados não entram.
  - Sem bilhetes no dia, deve retornar zeros em todos os campos numéricos.
  - Quando `data` estiver ausente, fora do formato (`2026-1-5`, `05/10/2026`) ou for data inexistente (`2026-02-30`), o sistema deve retornar `422 {"erro": "data_invalida"}`.

**UC5 — Cancelar bilhete** (`POST /bilhetes/{id}/cancelamento`)
- Critérios de aceite:
  - Quando o bilhete estiver aberto, o sistema deve retornar `200` com `{"id", "placa", "entrada", "status": "cancelado"}`, **sem** `saida` e **sem** `valor_centavos`.
  - Quando o `id` não existir, deve retornar `404 {"erro": "bilhete_nao_encontrado"}`.
  - Quando o bilhete já estiver encerrado ou cancelado, deve retornar `409 {"erro": "bilhete_nao_aberto"}`.
  - O bilhete cancelado deixa de aparecer em `/bilhetes/ativos`, mas continua no histórico da placa.

**UC6 — Histórico por placa** (`GET /bilhetes?placa=ABC1D23`)
- Critérios de aceite:
  - O sistema deve retornar `200` com array de **todos** os bilhetes da placa, em qualquer status, do mais recente para o mais antigo.
  - Placa válida que nunca estacionou → `200` com `[]`.
  - Quando `placa` estiver ausente ou fora do formato, deve retornar `422 {"erro": "placa_invalida"}`.

**UC7 — Tolerância gratuita** (regra de valor do UC2)
- Critérios de aceite:
  - Duração ≤ 15 minutos → `valor_centavos: 0` (0, 1, 14 e 15 minutos).
  - Duração de 16 minutos ou mais → cobra **integral desde o minuto zero**, sem descontar a tolerância (16 min → 2 frações → `200`).

**UC8 — Uma vaga por placa** (regra do UC1)
- Critérios de aceite:
  - Quando a placa já tiver bilhete `aberto`, `POST /bilhetes` deve retornar `409 {"erro": "bilhete_em_aberto"}` e não criar bilhete.
  - Depois de encerrar ou cancelar, a mesma placa deve conseguir abrir novo bilhete (`201`).
  - Placa inválida com bilhete aberto retorna `422`, nunca `409` (formato antes de conflito).

## Tabela de valores da variante (UC2 + UC7)

| Minutos | Frações | `valor_centavos` |
| --- | --- | --- |
| 0 a 15 | tolerância | `0` |
| 16 | 2 | `200` |
| 30 | 2 | `200` |
| 31 | 3 | `300` |
| 60 | 4 | `400` |
| 61 | 5 | `500` |
| 95 | 7 | `700` |
| 900 | 60 | `6000` (teto exato) |
| 901 ou mais | 61+ | `6000` (limitado pelo teto) |

## Decisões sobre ambiguidades

| # | Ambiguidade | Decisão |
| --- | --- | --- |
| D1 | O enunciado traz um exemplo `{"id": 7, "valor": 12.50}` | Ignorado: contradiz o contrato. O campo é `valor_centavos`, inteiro |
| D2 | Porta interna `8080` vs `PORTA_SERVICO` `8001` | Escuta nas duas, no mesmo processo e estado |
| D3 | `entrada` sem fuso (ex.: `2026-10-12T08:30:00`) | Aceita e interpreta como `-03:00` |
| D4 | Contagem de minutos com segundos | Trunca segundos (evita cobrar fração extra por milissegundos) |
| D5 | Como testar duração no encerramento | Corpo opcional `saida` (ISO-8601); ausente = agora; inválido = `422 {"erro": "entrada_invalida"}`; anterior à entrada = `0` minutos |
| D6 | Encerrar bilhete cancelado | `409 bilhete_ja_encerrado` (único 409 do UC2) |
| D7 | O que conta em `total_bilhetes` | Todos os bilhetes com entrada no dia, qualquer status |
| D8 | Ordem "mais recentes primeiro" | Por `entrada` decrescente; empate por `id` decrescente |
| D9 | Placa minúscula (`abc1d23`) | Rejeitada com `422 placa_invalida` (contrato exige maiúsculas) |

## Requisitos não funcionais

- Persistência em memória; a API sobe sem banco e sem variável de ambiente obrigatória.
- Rota inexistente → `404`.
- `GET /healthz` → `200 {"status": "ok"}` (apoio para subir o container).