# Constitution — Zona Azul Digital

Leia todos os `.md` antes de gerar código (`constitution.md` → `spec.md` →
`plan.md` → `tests.md` → `tasks.md`). Rotas, campos (minúsculos, com `_`),
status codes e regras numéricas abaixo são exatos e obrigatórios. Em conflito
entre arquivos, este prevalece.

## 1. Contrato resumido (vale para todo o projeto)

| UC | Método | Rota | Sucesso | Erros possíveis |
| --- | --- | --- | --- | --- |
| UC1 | `POST` | `/bilhetes` | `201` bilhete com `status: "aberto"` | `422 placa_invalida`, `422 entrada_invalida`, `409 bilhete_em_aberto` |
| UC2 | `POST` | `/bilhetes/{id}/encerramento` | `200` `{id, placa, entrada, saida, minutos, valor_centavos}` | `404 bilhete_nao_encontrado`, `409 bilhete_ja_encerrado` |
| UC3 | `GET` | `/bilhetes/ativos` | `200` array dos bilhetes abertos, mais recentes primeiro | — |
| UC4 | `GET` | `/relatorios/diario?data=AAAA-MM-DD` | `200` `{data, total_bilhetes, faturamento_centavos, tempo_medio_minutos}` | `422 data_invalida` |
| UC5 | `POST` | `/bilhetes/{id}/cancelamento` | `200` bilhete com `status: "cancelado"` | `404 bilhete_nao_encontrado`, `409 bilhete_nao_aberto` |
| UC6 | `GET` | `/bilhetes?placa=ABC1D23` | `200` array de todos os bilhetes da placa, mais recentes primeiro (vazio se nenhum) | `422 placa_invalida` |
| UC7 | regra | tolerância gratuita | duração ≤ 15 min → `valor_centavos: 0`; 16 min ou mais → cobra integral desde o minuto 0 (16 min = `200`) | — |
| UC8 | regra | uma vaga por placa | — | `409 bilhete_em_aberto` |

Todo erro tem corpo JSON exatamente `{"erro": "<codigo>"}`.

## 2. Parâmetros da variante (fixos neste projeto)

| Parâmetro | Valor |
| --- | --- |
| `TARIFA_HORA_CENTAVOS` | `400` |
| `FRACAO_MINUTOS` | `15` (valor de cada fração = 400 ÷ 4 = `100` centavos) |
| `TETO_DIARIO_CENTAVOS` | `6000` |
| `TOLERANCIA_MINUTOS` | `15` |
| `PORTA_SERVICO` | `8001` |

Esses valores ficam em um único módulo de configuração, como constantes com
esses nomes exatos. Podem ser sobrescritos por variável de ambiente de mesmo
nome, mas **nenhuma variável de ambiente é obrigatória**.

## 3. Regras operacionais

1. **Dinheiro é sempre inteiro em centavos.** Nenhum campo monetário pode ser
   `float` em nenhuma camada (cálculo, armazenamento ou JSON). O campo se chama
   `valor_centavos`, nunca `valor`.
2. **Datas e horas em ISO-8601 com fuso `-03:00`** em todas as respostas, no
   formato `AAAA-MM-DDTHH:MM:SS-03:00`, sem microssegundos.
3. **Minutos são inteiros e truncados:** `minutos` = segundos decorridos ÷ 60,
   descartando a sobra. Nunca arredondar minutos para cima.
4. **Arredondamentos definidos pelo contrato:**
   - frações de cobrança: sempre **para cima**;
   - `tempo_medio_minutos`: **0,5 para cima** (half-up), com aritmética
     inteira. Não usar a função `round()` do Python, que arredonda 0,5 para o
     par mais próximo e daria resultado errado (ex.: 10,5 viraria 10).
5. **Teto:** `valor_centavos` nunca supera `6000`.
6. **Precedência de erros:** validação de formato (`422`) vem antes de
   existência (`404`), que vem antes de conflito de estado (`409`). Payload
   malformado nunca dispara conflito.
7. **Nomes do contrato são intocáveis:** rotas, campos e códigos de erro em
   português, minúsculos, exatamente como na tabela da seção 1. Proibido
   traduzir para inglês ou mudar para camelCase.
8. **Status de bilhete:** somente `aberto`, `encerrado` ou `cancelado`.
9. **Erros do framework não vazam:** nenhuma resposta pode sair no formato
   padrão do framework (ex.: `{"detail": ...}`). Todo erro segue `{"erro": ...}`.

## 4. Stack e padrões de código

- Python 3.12, FastAPI, Uvicorn e Pytest. Código-fonte em `app/`, testes em `tests/`.
- Funções com *type hints*; código limpo segundo o linter `ruff`, sem imports
  não usados, sem `print` de depuração e sem código comentado.
- Regras de negócio (cálculo de valor, validações, relatório) em funções puras,
  separadas das rotas HTTP, para serem testáveis sem servidor.
- Nenhum segredo, token ou senha no repositório.

## 5. Entregáveis obrigatórios do código gerado (SDLC)

| Arquivo | Exigência |
| --- | --- |
| `Dockerfile` | Base `python:3.12-slim`, instala dependências, `EXPOSE 8001 8080` e `CMD` que sobe a API |
| `Containerfile` | Conteúdo idêntico ao `Dockerfile` |
| `requirements.txt` | Todas as dependências com versão fixada (inclui `pytest`, `httpx` e `ruff`) |
| `README.md` | Como rodar localmente, como rodar os testes e como rodar com Docker (`docker build` e `docker run -p 8001:8001`) |
| `tests/` | Testes automatizados próprios cobrindo todos os cenários do `tests.md` (mínimo de 25 funções `def test_`) |
| `.gitignore` e `.dockerignore` | Ignoram `__pycache__/`, `.venv/`, `.pytest_cache/` e arquivos `.env` |

## 6. Restrições

- Somente a API é escopo; não há front-end nem back-office.
- A aplicação sobe sem banco externo e sem variável de ambiente obrigatória.
- Escuta em `0.0.0.0` nas portas `8001` (`PORTA_SERVICO`) e `8080` (porta
  interna do contrato) ao mesmo tempo, no mesmo processo e com o mesmo estado.