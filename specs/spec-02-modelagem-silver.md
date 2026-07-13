# Spec-02 — Modelagem Silver (tx, vin, vout)

Depende de: `spec-01-ingestao.md` (tabela `bronze_address_txs` precisa existir e estar populada)
Contexto do projeto: ver `00-contexto-projeto.md`

## Objetivo

Transformar o JSON cru da camada Bronze em três tabelas Delta estruturadas e tipadas — `silver_tx`, `silver_vin`, `silver_vout` — usando PySpark com schema explícito (sem inferência automática).

## Requisitos funcionais

1. Ler `bronze_address_txs`, aplicar `from_json` sobre `raw_json` com um schema explícito que reflita o formato de transação da API mempool.space (campos: `txid`, `version`, `locktime`, `vin[]`, `vout[]`, `status.confirmed`, `status.block_height`, `status.block_time`).
2. Gerar **`silver_tx`**: uma linha por transação.
3. Gerar **`silver_vin`**: uma linha por input, via `explode()` do array `vin`.
4. Gerar **`silver_vout`**: uma linha por output, via `explode()` do array `vout`.
5. **Transação coinbase**: identificar pelo campo `vin[0].is_coinbase = true` da API. Nesse caso não existe `prev_txid`/`prev_vout_index` — marcar `is_coinbase = true` em `silver_tx` e gravar `prev_txid = null` em `silver_vin`.
6. **Output sem endereço decodificável** (ex.: `OP_RETURN`, script não padrão): gravar `address = null` e preencher `script_type` com o valor real retornado pela API (`scriptpubkey_type`). Não descartar a linha.
7. Tipos de script esperados como padrão: `p2pkh`, `p2sh`, `v0_p2wpkh`, `v0_p2wsh`, `v1_p2tr`. Qualquer tipo fora dessa lista: gravar como está, logar um warning (não falhar o processamento).

## Requisitos não funcionais

- Implementação obrigatoriamente em PySpark (`spark.read`, `explode`, `from_json` com schema explícito) — não usar Pandas. Este pipeline é também material de aula de Spark Fundamentals; a escolha de engine é didática, não só técnica.
- Idempotente: reprocessar o mesmo Bronze não deve duplicar linhas em Silver. Usar overwrite completo por execução ou MERGE por chave primária.

## Contrato de dados

### `silver_tx`

| Coluna | Tipo | Nulo? |
|---|---|---|
| `txid` | string | não (chave) |
| `version` | int | não |
| `locktime` | long | não |
| `block_height` | long | sim (nulo se não confirmada) |
| `block_time` | timestamp | sim |
| `is_coinbase` | boolean | não |

### `silver_vin`

| Coluna | Tipo | Nulo? |
|---|---|---|
| `txid` | string | não |
| `vin_index` | int | não |
| `prev_txid` | string | sim (nulo se coinbase) |
| `prev_vout_index` | int | sim |
| `sequence` | long | não |

Chave: `(txid, vin_index)`.

### `silver_vout`

| Coluna | Tipo | Nulo? |
|---|---|---|
| `txid` | string | não |
| `vout_index` | int | não |
| `address` | string | sim (nulo se sem endereço decodificável) |
| `value_sats` | long | não |
| `script_type` | string | não |

Chave: `(txid, vout_index)`.

## Critérios de aceite

1. Para uma transação coinbase conhecida (ex.: a primeira transação de um bloco qualquer confirmado), `silver_tx.is_coinbase = true` e a linha correspondente em `silver_vin` tem `prev_txid = null`.
2. Conferência manual de conservação de valor: para uma transação não coinbase escolhida manualmente, soma de `value_sats` em `silver_vout` + taxa (obtida via API) == soma dos `value_sats` dos outputs anteriores referenciados pelos `vin` dessa tx. Validar em pelo menos 1 caso contra um block explorer de referência.
3. `COUNT(DISTINCT txid)` em `silver_tx` == `COUNT(DISTINCT txid)` em `bronze_address_txs`.
4. Rodar o processamento duas vezes seguidas sobre o mesmo Bronze → contagem de linhas idêntica nas 3 tabelas (idempotência).
5. Existe pelo menos uma linha em `silver_vout` com `address = null` e `script_type` refletindo um tipo não endereçável, sem erro de execução.

## Fora de escopo (v1)

- Resolução de endereço a partir de scripts não padrão além dos 5 tipos listados — apenas logar e marcar `unknown`.
- Cálculo de saldo agregado por endereço (isso pertence à camada Gold, `spec-03`).
- Qualquer conversão para valor fiat.

## Interface de execução sugerida

```
python -m btc_tracker.silver --input-path <bronze_delta> --output-path <silver_delta>
```
