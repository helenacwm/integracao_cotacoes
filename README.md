# Integração de Cotações de Moedas (AwesomeAPI -> SQLite)

**Grupo 1:** Ana Luiza, Dieli, Helena e Pedro Alves

## 1. O que a aplicação faz
Consome a API pública **AwesomeAPI**, transforma os dados e os grava no **SQLite** local.

## 2. Arquitetura
`AwesomeAPI ──HTTP──> api_client ──> transformer ──> repository ──> SQLite`

## 3. Tecnologias
Python 3, `requests` (API), `sqlite3` (Banco), `pytest` (Testes).

## 4. Como executar
```bash
pip install -r requirements.txt
export AWESOMEAPI_KEY=sua_chave   # opcional (veja a seção 5)
python -m app.main sync      # busca na API e grava no SQLite (cotacoes.db)
python -m app.main listar    # mostra o que foi gravado
python -m pytest -v          # roda os testes
```

## 5. Tratamento de falhas e decisões de implementação
- **API isolada:** só `app/api_client.py` conhece URL, HTTP, timeout e retry; o restante recebe dados já obtidos.
- **Falhas de comunicação:** timeout de 5 s e até 3 tentativas com espera crescente (timeout, conexão, HTTP 5xx). Falha final gera `ErroComunicacaoAPI`.
- **HTTP 429 (limite de requisições):** nova tentativa respeitando `Retry-After` (máx. 30 s) ou o dobro da espera normal.
- **Respostas inválidas:** HTTP 4xx (exceto 429), JSON inválido, resposta vazia ou campos ausentes/não numéricos geram `RespostaInvalidaAPI`. Nada é gravado nesses casos.
- **Chave de API (opcional):** lida da variável de ambiente `AWESOMEAPI_KEY` e enviada no cabeçalho `x-api-key`. A AwesomeAPI oferece chave gratuita; sem ela, a API pode limitar (HTTP 429) em IPs compartilhados, como o do Google Colab.
- **Sem dados falsos:** a aplicação nunca grava dados simulados; se a API falha, o erro é mostrado.
- **SQLite:** não exige servidor. `UNIQUE(par, cotado_em)` + `INSERT OR IGNORE` evita duplicar a mesma cotação.
- **Transformação:** textos viram número, timestamp Unix vira data (UTC) e é calculado o spread (venda - compra).
- **Client injetável:** `CotacaoService` recebe o client, o que permite testar sem internet.

## 6. Testes (9)
O teste de integração exercita service, transformer e SQLite reais, com a resposta HTTP simulada: fluxo completo; sem duplicar ao sincronizar duas vezes; timeout; HTTP 503 com 3 tentativas; HTTP 429 com recuperação; HTTP 404 sem repetir; envio da chave no cabeçalho; JSON inválido; campos faltando.

## 7. Classificação da Integração
| Critério | Classificação | Justificativa |
|---|---|---|
| Contexto de negócio | B2B (consumo de serviço de terceiros) | A aplicação consome dados de um serviço mantido por outra organização. |
| Padrão arquitetural | Ponto a ponto (Point-to-Point) | Conexão direta entre dois sistemas, sem broker, ESB ou hub. |
| Padrão de comunicação | Síncrona, requisição-resposta, REST/JSON via HTTP (pull) | A aplicação pede, espera a resposta e então processa; a iniciativa é nossa, sem eventos. |
