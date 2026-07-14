# Plano — Spec-03: Grafo de linhagem (BFS por hops)

Spec de referência: `specs/spec-03-grafo-linhagem.md` · Depende de: spec-02 (`silver_tx`, `silver_vin`, `silver_vout`) · Índice: [../PLANO.md](../PLANO.md)

## Entregáveis concretos

1. `btc_tracker/lineage.py` — `run_lineage(address, max_hops=2, direction="both", max_fanout=500, ...) -> LineageResult` + wrapper `__main__`:
   ```
   python -m btc_tracker.lineage --address <endereco> --max-hops 2 --output-path <gold_delta>
   ```
2. Extensão do `api_client.py`: `GET /api/tx/:txid/outspend/:vout` (verificação de output gasto, direção forward).
3. Tabela `gold_lineage_edges` com o contrato exato da spec: `from_address` (null se coinbase), `to_address`, `txid`, `value_sats`, `block_height` (null se não confirmada), `hop_distance`, `direction`, `is_coinbase_origin`, `truncated`.

## Conceito central: orquestração, não reimplementação

Specs 01/02 só ingerem transações que tocam diretamente um endereço. `lineage` **orquestra o loop**: para cada endereço novo na fronteira do hop, chama `run_ingest()` + `run_silver()` (importados como funções — decisão registrada no PLANO.md) antes de expandir o hop seguinte. Nada de ingestão duplicada dentro de `lineage.py`.

## Algoritmo previsto (BFS iterativo por hop)

```
fronteira = {address_entrada}; visitados = {}; hop = 0
enquanto fronteira ≠ ∅ e hop < max_hops:
    1. para cada endereço da fronteira ainda sem dado: run_ingest + run_silver   # reuso antes de rede
    2. BACKWARD: joins Spark — txs que creditam a fronteira → silver_vin.prev_txid/prev_vout_index
       → silver_vout da tx de origem → endereço de origem. Se a tx de origem não está em Silver,
       ingeri-la (por txid) antes de resolver.
       • vin.is_coinbase → aresta com is_coinbase_origin=true, from_address=null; NÃO expande.
    3. FORWARD: outputs das txs que tocam a fronteira → gasto? Primeiro tenta resolver via silver_vin
       já ingerido (gasto conhecido localmente); só então GET /tx/:txid/outspend/:vout.
       Se gasto → endereço(s) de destino na tx gastadora.
    4. FAN-OUT: tx com > max_fanout (default 500) inputs ou outputs → arestas marcadas truncated=true,
       tx NÃO expande para o próximo hop.
    5. ANTI-CICLO: novos endereços − visitados → fronteira do hop+1; visitados ∪= fronteira atual.
    6. MERGE das arestas do hop em gold_lineage_edges; hop += 1
```

- **Cada hop é uma passada de joins no Spark** (exigência da spec: BFS iterativo, não recursão em memória).
- Chave de MERGE das arestas: `(txid, from_address, to_address, direction)` — reexecução nunca duplica; execução com `max_hops` maior só acrescenta (propriedade de superset do critério 2).
- `hop_distance` no MERGE: manter o menor valor já registrado (endereço alcançado por caminho mais curto prevalece).

## Critérios de aceite (da spec) e validação

| # | Critério | Estratégia |
|---|---|---|
| 1 | `max_hops=2` em endereço pequeno conhecido → hops 0 e 1 batem com block explorer | conferência manual documentada no PR |
| 2 | Resultado de `max_hops=2` ⊇ resultado de `max_hops=1` | teste automatizado: run 1, snapshot, run 2, assert de superset |
| 3 | Tx com fan-out > limite → `truncated=true` e sem arestas de hop seguinte a partir dela | teste com `max_fanout` artificialmente baixo (ex.: 2) sobre fixture |
| 4 | Duas execuções mesmos parâmetros → sem duplicação em `gold_lineage_edges` | assert de contagem |
| 5 | Backward chegando em coinbase → `is_coinbase_origin=true`, `from_address=null`, sem erro | endereço de teste cujo backward alcança coinbase em ≤2 hops (ex.: recebedor direto de recompensa de bloco) |

## Testes

- **Unit (offline):** lógica de fronteira/visitados/truncamento sobre tabelas Silver sintéticas (sem rede); resolução backward e forward com fixtures.
- **Integração:** os 5 critérios; endereço real pequeno; `outspend` mockado nos testes offline.

## Riscos específicos desta fatia

- **Volume de chamadas `outspend` (1 por output)** → mitigar resolvendo gasto via Silver local primeiro e respeitando backoff; endereço de teste pequeno.
- **Latência do loop ingest→silver por endereço da fronteira** → aceita no MVP; medir baseline (spec-04) antes de otimizar.

## Fora de escopo (vinculante)

Clusterização de entidade (co-spend); grafo completo até o gênesis; streaming em tempo real.

## Definição de pronto da fatia

5 critérios passando (critério 1 com evidência no PR); branch `feature/spec-03-linhagem` com PR aprovado em `dev`.
