# Spec-01 — Ingestão de transações por endereço

Depende de: nenhuma (primeira spec do pipeline)
Contexto do projeto: ver `00-contexto-projeto.md`

## Objetivo

Dado um endereço Bitcoin, buscar **todas** as transações que o tocam via API pública mempool.space e persistir o dado cru (sem transformação) em uma tabela Delta Lake.

## Requisitos funcionais

1. **Input**: endereço Bitcoin como string. Formatos aceitos: legacy (`1...`), P2SH (`3...`), bech32/segwit (`bc1q...`), taproot (`bc1p...`).
2. **Validação prévia**: validar o formato do endereço (checksum incluído) antes de qualquer chamada de rede. Se inválido, retornar erro claro e não chamar a API.
3. **Busca paginada**: a API mempool.space retorna no máximo 25 transações confirmadas por chamada (`GET /api/address/:address/txs`, mais recentes primeiro). Para obter o histórico completo, paginar com `GET /api/address/:address/txs/chain/:last_seen_txid` até a resposta vir vazia.
4. **Transações não confirmadas**: incluir também `GET /api/address/:address/txs/mempool` (transações no mempool, ainda sem bloco) — persistir com `block_height = null`.
5. **Rate limiting**: tratar HTTP 429 com backoff exponencial (mínimo 3 tentativas). Não disparar requisições em paralelo descontrolado — uma cadeia de paginação por vez, por endereço.
6. **Persistência**: gravar o JSON cru de cada transação (sem parse de campos internos) em uma tabela Delta `bronze_address_txs`.
7. **Idempotência**: reexecutar a ingestão para o mesmo endereço não deve duplicar transações já persistidas. Usar MERGE (upsert) por chave `(address, txid)`.

## Requisitos não funcionais

- Deve funcionar sem erro para um endereço válido com zero transações (tabela resultante vazia).
- Deve logar, ao final, quantas transações novas foram inseridas e quantas já existiam.
- Timeout de rede: tratar com retry (máximo 3 tentativas, backoff exponencial começando em 1s).
- Não deve haver nenhuma lógica de preço, conversão de moeda ou métricas de mercado neste módulo.

## Contrato de dados — tabela `bronze_address_txs`

| Coluna | Tipo | Nulo? | Descrição |
|---|---|---|---|
| `address` | string | não | endereço consultado |
| `txid` | string | não | id da transação |
| `raw_json` | string | não | corpo JSON completo retornado pela API para essa tx |
| `confirmed` | boolean | não | `status.confirmed` da API |
| `fetched_at` | timestamp | não | momento da ingestão |

Chave lógica: `(address, txid)`.

## Critérios de aceite

1. Rodar a ingestão para um endereço público conhecido com histórico de transações (ex.: um endereço de doação de projeto open source) e confirmar que `bronze_address_txs` contém, no mínimo, o número de transações confirmadas visível em um block explorer de referência.
2. Rodar a ingestão duas vezes seguidas para o mesmo endereço → segunda execução não altera a contagem de linhas (idempotência).
3. Rodar para um endereço com formato inválido (checksum quebrado) → erro claro antes de qualquer chamada de rede, nenhuma linha escrita.
4. Rodar para um endereço válido sem nenhuma transação → execução sem exceção, tabela resultante vazia (ou sem novas linhas).
5. Simular resposta HTTP 429 (ou testar contra endereço de altíssimo volume) → execução se recupera via backoff, sem falhar imediatamente.

## Fora de escopo (v1)

- Parse do conteúdo interno da transação (`vin`/`vout` estruturados) — isso é `spec-02`.
- Download de blocos via full node (`bitcoind`) — fica para uma fase 2 do projeto, quando a dependência da API de terceiro precisar ser eliminada.
- Qualquer enriquecimento com dado de preço/mercado.

## Interface de execução sugerida

```
python -m btc_tracker.ingest --address <endereco> --output-path <caminho_delta>
```

Saída no terminal: contagem de transações novas inseridas e total acumulado para o endereço.
