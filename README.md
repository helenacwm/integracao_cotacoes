# Integração de Cotações de Moedas (AwesomeAPI -> SQLite)

Trabalho de Avaliação 3 — Integração com Reuso de Software.

**Grupo:** _1_

## 1. O que a aplicação faz

Consome a API pública **AwesomeAPI** (`https://economia.awesomeapi.com.br`), que fornece cotações de moedas, transforma os dados recebidos e os grava em um banco **SQLite** local.

Fluxo:

```
AwesomeAPI ──HTTP/JSON──> api_client ──> transformer ──> repository ──> SQLite
                         (comunicação)   (regras)       (persistência)
                                    orquestrado por service
```

## 2. Requisitos da atividade × onde estão atendidos

| # | Requisito | Onde |
|---|-----------|------|
| 1 | Consumir dados de API externa | `app/api_client.py` |
| 2 | Processar/transformar dados | `app/transformer.py` |
| 3 | Persistir em armazenamento | `app/repository.py` (SQLite) |
| 4 | Tratar falhas e respostas inválidas | `api_client.py` + `exceptions.py` |
| 5 | Comunicação com API separada do resto | Apenas `api_client.py` conhece HTTP/URL |
| 6 | Teste de integração | `tests/test_integracao.py` |
| 7 | README | este arquivo |
| 8 | Classificação da integração | seção 6 |

## 3. Tecnologias

- Python 3.10+
- `requests` (cliente HTTP)
- `sqlite3` (biblioteca padrão do Python)
- `pytest` (testes)

## 4. Como executar

```bash
# 1. Entrar na pasta e criar ambiente virtual (opcional, recomendado)
cd integracao-cotacoes
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

# 2. Instalar dependências
pip install -r requirements.txt

# 3. Buscar cotações na API e gravar no banco (padrão: USD-BRL, EUR-BRL, BTC-BRL)
python -m app.main sync
python -m app.main sync USD-BRL GBP-BRL     # pares específicos

# 4. Ver o que foi gravado
python -m app.main listar
python -m app.main listar USD-BRL

# 5. Rodar os testes
python -m pytest -v
```

O banco é criado automaticamente no arquivo `cotacoes.db`.

## 5. Arquitetura e decisões de implementação

```
integracao-cotacoes/
├── app/
│   ├── api_client.py    # ÚNICA camada que fala com a API externa
│   ├── transformer.py   # converte dados da API para o modelo interno
│   ├── repository.py    # acesso ao SQLite
│   ├── service.py       # orquestra: buscar -> transformar -> salvar
│   ├── exceptions.py    # erros próprios da integração
│   └── main.py          # interface de linha de comando
├── tests/test_integracao.py
└── requirements.txt
```

Decisões:

- **Isolamento da API externa:** só `api_client.py` conhece URL, HTTP, timeout e formato bruto. Se a API mudar, ou for trocada por outra, só esse arquivo é alterado.
- **Tolerância a falhas:**
  - timeout de 5 s em toda requisição;
  - até 3 tentativas com espera crescente em caso de falha de rede, timeout ou HTTP 5xx;
  - erros 4xx **não** geram nova tentativa (repetir não resolveria);
  - duas exceções distintas: `ErroComunicacaoAPI` (rede/servidor) e `RespostaInvalidaAPI` (JSON inválido, campos ausentes, valores não numéricos ou não positivos);
  - dados inválidos nunca chegam ao banco.
- **Transformação:** textos numéricos viram `float`; o timestamp Unix vira data ISO 8601 em UTC; são calculados o `spread` (venda − compra) e mantida a variação percentual do dia.
- **Persistência idempotente:** a tabela tem `UNIQUE(par, cotado_em)` e a gravação usa `INSERT OR IGNORE`, então sincronizar duas vezes a mesma cotação não duplica registros. A gravação ocorre em uma transação.
- **Testes sem depender da internet:** o teste de integração sobe um servidor HTTP local que imita a AwesomeAPI. O fluxo é real (HTTP real → transformação → SQLite real), mas determinístico.

### Testes (7, todos passando)

1. Fluxo completo grava os dados transformados.
2. Sincronizar duas vezes não duplica.
3. HTTP 503 gera 3 tentativas e nada é gravado.
4. JSON inválido é rejeitado.
5. Campos faltando são rejeitados.
6. Valor não numérico é rejeitado.
7. API fora do ar gera erro de comunicação.

## 6. Classificação da integração

| Critério | Classificação | Justificativa |
|----------|---------------|---------------|
| **Contexto de negócio** | Integração externa entre organizações (B2B / consumo de serviço de terceiros) | Nossa aplicação consome dados de um serviço mantido por outra organização, fora do nosso domínio. |
| **Padrão arquitetural** | Ponto a ponto (point-to-point), baseada em API (cliente–servidor) | Há uma única conexão direta entre dois sistemas, sem barramento (ESB) nem broker intermediário. |
| **Padrão de comunicação** | Síncrona, requisição–resposta, REST/JSON sobre HTTP, com obtenção por *pull* sob demanda | O sistema envia a requisição, espera a resposta e só então processa. Os dados são buscados por iniciativa nossa (não há push/eventos). |

## 7. Limitações e melhorias futuras

- A sincronização é manual (sob demanda); poderia ser agendada (cron).
- Sem cache nem controle de limite de requisições da API.
- Poderia expor os dados por uma API própria ou um dashboard.
