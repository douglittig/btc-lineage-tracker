# Spec-03 — Grafo de linhagem (BFS por hops)

Depende de: `spec-02-modelagem-silver.md` (`silver_tx`, `silver_vin`, `silver_vout`)
Contexto do projeto: ver `00-contexto-projeto.md`

## Objetivo

Dado um endereço de partida e uma profundidade `max_hops`, construir o grafo de proveniência: **backward** (de onde vieram os sats, seguindo `vin.prev_txid` pra trás) e **forward** (pra onde foram os sats gastos, seguindo outputs não gastos pra frente).

## Conceito central: expansão incremental

As specs 01/02 só ingerem transações que tocam **diretamente** o endereço de entrada. Para seguir o grafo além do hop 0, é necessário descobrir novos endereços/transações na fronteira do BFS e reexecutar `spec-01` + `spec-02` para cada um antes de avançar ao hop seguinte. `spec-03` orquestra esse loop — não reimplementa ingestão/modelagem.

### Definição de hop

- **Hop 0**: o endereço de entrada.
- **Hop 1**: endereços que enviaram (backward) ou receberam (forward) diretamente do endereço de entrada, via alguma transação.
- **Hop N**: endereços conectados a algum endereço do hop N-1, ainda não visitados em hops anteriores.

## Requisitos funcionais

1. **Parâmetro `max_hops`** (default: 2). Roda o BFS até atingir essa profundidade ou esgotar a fronteira.
2. **Direção backward**: para cada transação que toca um endereço da fronteira atual, seguir `silver_vin.prev_txid` + `prev_vout_index` até a transação de origem, resolver o endereço correspondente em `silver_vout` dessa transação de origem. Esse endereço entra na fronteira do próximo hop.
3. **Direção forward**: para cada transação que toca um endereço da fronteira atual, olhar os `silver_vout` dessa transação e verificar, via `GET /api/tx/:txid/outspend/:vout`, se o output já foi gasto. Se sim, seguir para a transação que o gastou; o(s) endereço(s) de destino entram na fronteira do próximo hop.
4. **Transação coinbase no caminho backward**: fim de linha — a moeda foi minerada, não tem origem anterior. Marcar explicitamente no resultado (`is_coinbase_origin = true`), não tratar como erro.
5. **Limite de fan-out**: se uma transação tem mais de `max_fanout` inputs ou outputs (default: 500 — típico de transação de consolidação de exchange), marcar essa transação como `truncated = true` no resultado e **não** expandir a partir dela. Isso evita explosão combinatória em transações de alto volume.
6. **Anti-ciclo**: manter um conjunto de endereços já visitados; não reprocessar um endereço já visitado em hop anterior ou igual.

## Requisitos não funcionais

- Implementação como BFS iterativo (cada hop é uma passada completa de join no Spark), não recursão irrestrita em memória.
- Incremental: se um endereço já foi processado (existe em `gold_lineage_edges` com hop suficiente), reexecutar não deve reprocessar do zero — usar MERGE por chave.
- Sem chamadas de rede sem necessidade: reaproveitar dado já ingerido em Bronze/Silver antes de buscar mais via API.

## Contrato de dados — tabela `gold_lineage_edges`

| Coluna | Tipo | Nulo? | Descrição |
|---|---|---|---|
| `from_address` | string | sim | endereço de origem da aresta (nulo se `is_coinbase_origin`) |
| `to_address` | string | não | endereço de destino da aresta |
| `txid` | string | não | transação que liga os dois |
| `value_sats` | long | não | valor transferido nessa aresta |
| `block_height` | long | sim | altura do bloco da tx (nulo se não confirmada) |
| `hop_distance` | int | não | distância em hops a partir do endereço de entrada |
| `direction` | string | não | `"backward"` ou `"forward"` |
| `is_coinbase_origin` | boolean | não | true se a aresta termina em uma coinbase |
| `truncated` | boolean | não | true se a tx de origem foi truncada por fan-out excessivo |

## Critérios de aceite

1. Para um endereço de teste público com histórico pequeno e conhecido (ex.: endereço de doação com poucas dezenas de transações), rodar com `max_hops=2` e validar manualmente, contra um block explorer, que os hops 0 e 1 do resultado batem com a realidade.
2. Rodar com `max_hops=1` e depois com `max_hops=2` para o mesmo endereço → o resultado de `max_hops=2` contém estritamente todas as arestas do resultado de `max_hops=1` (é superset).
3. Uma transação com fan-out acima do limite configurado aparece em `gold_lineage_edges` com `truncated=true` e não gera arestas de hop seguinte a partir dela.
4. Rodar duas vezes seguidas com o mesmo endereço e mesmo `max_hops` → sem duplicação de linhas em `gold_lineage_edges`.
5. Um caminho backward que chega em uma transação coinbase aparece com `is_coinbase_origin=true` e `from_address=null`, sem erro de execução.

## Fora de escopo (v1)

- Clusterização de entidade (heurística de co-spend para agrupar endereços de uma mesma carteira) — fica para uma spec futura, pós-MVP.
- Resolução do grafo completo até o bloco gênesis — inviável sem full node e processamento em escala; `max_hops` existe justamente para conter isso.
- Streaming em tempo real (novo bloco chegando e atualizando o grafo automaticamente) — pertence a uma evolução futura desta spec, quando o pipeline batch já estiver validado.

## Interface de execução sugerida

```
python -m btc_tracker.lineage --address <endereco> --max-hops 2 --output-path <gold_delta>
```
