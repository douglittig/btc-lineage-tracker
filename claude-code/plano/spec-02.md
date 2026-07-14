# Plano — Spec-02: Modelagem Silver (tx, vin, vout)

Spec de referência: `specs/spec-02-modelagem-silver.md` · Depende de: spec-01 (`bronze_address_txs` populada) · Índice: [../PLANO.md](../PLANO.md)

## Entregáveis concretos

1. `btc_tracker/schemas.py` (extensão) — `StructType` **explícito e completo** da transação mempool.space: `txid`, `version`, `locktime`, `vin[]` (com `is_coinbase`, `txid` do prev, `vout`, `sequence`, `prevout`), `vout[]` (com `scriptpubkey_address`, `value`, `scriptpubkey_type`), `status{confirmed, block_height, block_time}`. **Nenhuma inferência de schema.**
2. `btc_tracker/silver.py` — `run_silver(input_path, output_path, spark) -> SilverResult` + wrapper `__main__`:
   ```
   python -m btc_tracker.silver --input-path <bronze_delta> --output-path <silver_delta>
   ```

## Fluxo previsto de `run_silver`

1. `spark.read.format("delta")` sobre a Bronze; dedup por `txid` (o mesmo txid pode existir para vários `address`).
2. `from_json(raw_json, schema_explicito)` → coluna estruturada.
3. Deriva as três tabelas:
   - **`silver_tx`** (1 linha/tx): `txid`, `version`, `locktime`, `block_height` (null se não confirmada), `block_time` (timestamp, null idem), `is_coinbase` (= `vin[0].is_coinbase`).
   - **`silver_vin`** via `posexplode(vin)`: `txid`, `vin_index`, `prev_txid` (null se coinbase), `prev_vout_index` (null idem), `sequence`. Chave `(txid, vin_index)`.
   - **`silver_vout`** via `posexplode(vout)`: `txid`, `vout_index`, `address` (null se não decodificável — ex. OP_RETURN; **linha não é descartada**), `value_sats`, `script_type` (= `scriptpubkey_type` cru). Chave `(txid, vout_index)`.
4. Tipos de script fora de {`p2pkh`, `p2sh`, `v0_p2wpkh`, `v0_p2wsh`, `v1_p2tr`}: gravar como está + **warning no log**, nunca falhar.
5. Escrita: **overwrite completo** das três tabelas por execução — derivação determinística do Bronze, idempotente por construção.

## Decisões desta fatia

- PySpark obrigatório (`spark.read`, `from_json`, `explode`) — exigência didática da spec; nada de Pandas.
- Overwrite em vez de MERGE na Silver: mais simples, atende idempotência e a Silver é 100% derivável do Bronze. (Bronze usa MERGE; Gold usa MERGE — a escolha aqui é local e permitida pela spec.)

## Critérios de aceite (da spec) e validação

| # | Critério | Estratégia |
|---|---|---|
| 1 | Tx coinbase → `silver_tx.is_coinbase=true` e `silver_vin.prev_txid=null` | ingerir a coinbase de um bloco confirmado conhecido; assert nas duas tabelas |
| 2 | Conservação de valor: Σ vout + taxa == Σ outputs anteriores referenciados | conferência manual documentada de ≥1 tx contra block explorer; taxa via API |
| 3 | `COUNT(DISTINCT txid)` em `silver_tx` == na Bronze | assert automatizado pós-run |
| 4 | Duas execuções sobre o mesmo Bronze → contagens idênticas nas 3 tabelas | assert automatizado |
| 5 | ≥1 linha em `silver_vout` com `address=null` e `script_type` não endereçável, sem erro | fixture/endereço contendo tx com OP_RETURN |

## Testes

- **Unit (offline):** `from_json` com o schema explícito sobre fixtures JSON reais (tx normal, coinbase, OP_RETURN, taproot); derivação de cada tabela; warning para script_type desconhecido.
- **Integração:** os 5 critérios sobre um Bronze real produzido pela spec-01.

## Fora de escopo (vinculante)

Resolução de endereço além dos 5 tipos padrão (logar + `unknown`); saldo agregado (Gold/spec-03); conversão fiat.

## Definição de pronto da fatia

5 critérios passando (critério 2 com evidência da conferência no PR); branch `feature/spec-02-silver` com PR aprovado em `dev`.
