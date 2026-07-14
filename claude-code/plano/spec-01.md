# Plano — Spec-01: Ingestão de transações por endereço

Spec de referência: `specs/spec-01-ingestao.md` · Depende de: nada (primeira fatia) · Índice: [../PLANO.md](../PLANO.md)

## Entregáveis concretos

1. `btc_tracker/address.py` — validação de endereço Bitcoin **antes de qualquer rede**:
   - base58check com verificação de checksum (legacy `1...`, P2SH `3...`);
   - bech32 (BIP-173) para `bc1q...` e bech32m (BIP-350) para `bc1p...`.
   - Retorna erro claro (`InvalidAddressError` com mensagem legível) para formato/checksum inválido.
2. `btc_tracker/api_client.py` — cliente REST mempool.space:
   - `GET /api/address/:address/txs` (confirmadas, 25 por página, mais recentes primeiro);
   - paginação via `GET /api/address/:address/txs/chain/:last_seen_txid` até resposta vazia;
   - `GET /api/address/:address/txs/mempool` (não confirmadas);
   - tratamento de HTTP 429 e timeout com backoff exponencial: 3 tentativas, começando em 1s (1s → 2s → 4s);
   - estritamente **uma cadeia de paginação por vez por endereço** (loop sequencial, sem concorrência).
3. `btc_tracker/session.py` — factory de `SparkSession` local com extensão Delta configurada.
4. `btc_tracker/schemas.py` (parte bronze) — `StructType` da tabela `bronze_address_txs`.
5. `btc_tracker/ingest.py` — `run_ingest(address, output_path, spark) -> IngestResult` + wrapper `__main__`:
   ```
   python -m btc_tracker.ingest --address <endereco> --output-path <caminho_delta>
   ```
6. `btc_tracker/config.py` — constantes (base URL, retries, backoff inicial, timeout).

## Fluxo previsto de `run_ingest`

1. Valida endereço (falha antes de rede se inválido).
2. Consulta Bronze existente para o endereço (reuso: txids já persistidos).
3. Busca confirmadas paginando + mempool; para cada tx monta linha `(address, txid, raw_json, confirmed, fetched_at)` — `raw_json` é o corpo JSON completo, **sem parse de campos internos**.
4. MERGE Delta na `bronze_address_txs` por chave `(address, txid)` (upsert — nunca duplica).
5. Loga e retorna: novas inseridas × já existentes × total acumulado do endereço.

## Contrato de dados

Exatamente o da spec: `address` (string, not null), `txid` (string, not null), `raw_json` (string, not null), `confirmed` (boolean, not null), `fetched_at` (timestamp, not null). Chave lógica `(address, txid)`. Tx do mempool entram com `confirmed=false` (o `block_height=null` fica registrado dentro do `raw_json`; a coluna tipada é assunto da Silver).

## Critérios de aceite (da spec) e como validar cada um

| # | Critério | Estratégia de validação |
|---|---|---|
| 1 | Endereço público com histórico → Bronze contém ≥ nº de tx confirmadas visível em block explorer | teste de integração contra endereço de doação conhecido; comparação manual documentada |
| 2 | Duas execuções seguidas → contagem de linhas inalterada | teste automatizado: `count()` antes/depois da 2ª run |
| 3 | Endereço com checksum quebrado → erro claro, zero chamadas de rede, zero linhas | teste unitário (mock do client provando 0 chamadas) + integração |
| 4 | Endereço válido sem transações → sem exceção, tabela vazia/sem novas linhas | endereço recém-derivado válido e nunca usado |
| 5 | HTTP 429 → recuperação via backoff sem falha imediata | teste unitário com mock retornando 429×2 e depois 200; verificação dos delays |

## Testes

- **Unit (offline):** validação de endereço (casos válidos dos 4 formatos + checksums quebrados), lógica de paginação e backoff com mock HTTP, montagem de linhas Bronze sobre fixtures.
- **Integração:** os 5 critérios acima; fixtures JSON gravadas da API para rodar sem rede quando possível.

## Fora de escopo (vinculante)

Parse de `vin`/`vout` (spec-02); full node (fase 2); qualquer dado de preço/mercado.

## Definição de pronto da fatia

Os 5 critérios de aceite passando (critério 1 com evidência de conferência contra explorer registrada no PR); log final de novas/existentes funcionando; branch `feature/spec-01-ingestao` com PR aprovado em `dev`.
