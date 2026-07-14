# Plano — Spec-04: Interface de saída (CLI → API)

Spec de referência: `specs/spec-04-interface-saida.md` · Depende de: spec-03 (`gold_lineage_edges`) · Índice: [../PLANO.md](../PLANO.md)

## Entregáveis concretos (v1 = CLI; API HTTP fica documentada e fora do MVP)

1. `btc_tracker/trace.py` — orquestração ponta a ponta + `__main__` da CLI:
   ```
   python -m btc_tracker.trace --address <endereco> --hops <N> [--export out.json] [--direction backward|forward|both]
   ```
   - Função `run_trace(address, hops, direction="both") -> TraceResult` que executa `ingest → silver → lineage` (na prática delega a `run_lineage`, que já orquestra 01+02 por fronteira) e devolve as arestas.
   - **Toda a lógica vive em `run_trace`**; o `__main__` só faz argparse, validação de flags e impressão — é exatamente essa separação que permite à API v2 ser uma camada fina sem duplicar pipeline.
2. `btc_tracker/render.py` — os dois formatos de saída, gerados **do mesmo resultado**:
   - **JSON estruturado**: lista de arestas com o schema de `gold_lineage_edges`;
   - **Árvore textual**: endereço raiz no topo, ramificação indentada por hop, uma linha por aresta com direção (`←` backward, `→` forward), valor em sats, e marcadores de origem coinbase e truncamento.
3. Validação de argumentos com mensagens humanas (sem stack trace):
   - `--address` ausente → mensagem de uso/ajuda;
   - endereço inválido → mensagem clara (reusa `address.py`);
   - `--hops` fora do intervalo permitido `[0, 5]` → erro citando o intervalo, **sem executar o pipeline**.
4. `--direction backward|forward|both` (default `both`) propagado até `run_lineage` — roda só a direção pedida.
5. `--export <arquivo.json>` → grava o JSON em arquivo **e** imprime a árvore no terminal; conteúdo do arquivo idêntico ao JSON do resultado impresso.
6. **Baseline de tempo**: na primeira execução real de `--hops 2`, medir e **registrar** o tempo obtido (no PR e no README da implementação) como baseline para otimizações futuras — a spec proíbe estimar antes de medir.

## Documentação v2 (sem implementação)

Registrar no README da implementação a evolução já especificada: `GET /trace?address=...&hops=...&direction=...` devolvendo o mesmo JSON, sem autenticação no v2, com rate limiting + auth como pré-requisitos explícitos antes de qualquer exposição pública. Nenhum código de API no MVP.

## Critérios de aceite (da spec) e validação

| # | Critério | Estratégia |
|---|---|---|
| 1 | Ponta a ponta em endereço conhecido → JSON parseável + árvore legível | teste de integração: `json.loads` no output + inspeção manual da árvore documentada no PR |
| 2 | `--export saida.json` → arquivo criado, conteúdo idêntico ao resultado impresso | teste automatizado comparando arquivo × saída |
| 3 | Sem `--address` → ajuda clara, sem stack trace | teste de CLI capturando stderr/exit code |
| 4 | `--hops -1` / `--hops 999` → erro citando intervalo, pipeline não executa | teste de CLI + assert de que nenhuma tabela foi tocada |
| 5 | `--direction backward` → só arestas `direction="backward"` | assert sobre o JSON |

## Testes

- **Unit (offline):** renderização da árvore sobre conjuntos de arestas sintéticos (backward, forward, coinbase, truncada, multi-hop); validação de argumentos; serialização JSON.
- **Integração:** os 5 critérios + a **definição de pronto do projeto** (`00-contexto-projeto.md`): `trace --address <addr> --hops 2` contra endereço público conhecido, grafo validado manualmente contra block explorer, tempo razoável, sem duplicação em reexecução.

## Fora de escopo (vinculante)

UI gráfica/visualização interativa; autenticação, deploy ou hospedagem da API; cache de resultados entre execuções.

## Definição de pronto da fatia

CLI funcional testada contra endereço público, JSON exportável, árvore legível, baseline de tempo registrado — sem depender de nenhum componente da v2. Encerra o keystone do projeto. Branch `feature/spec-04-cli` com PR aprovado em `dev`.
