# Spec-04 — Interface de saída (CLI → API)

Depende de: `spec-03-grafo-linhagem.md` (`gold_lineage_edges`)
Contexto do projeto: ver `00-contexto-projeto.md`

## Objetivo

Expor o grafo de linhagem de forma usável. V1 é uma CLI que roda o pipeline completo e devolve o resultado; a API HTTP é a evolução documentada aqui, mas fora do MVP.

## Requisitos funcionais — v1 (CLI)

1. **Comando principal**:
   ```
   python -m btc_tracker.trace --address <endereco> --hops <N>
   ```
   Executa o pipeline completo na ordem (`ingest` → `silver` → `lineage`) para o endereço de entrada e para cada endereço descoberto na fronteira, até `max_hops`.
2. **Saída em dois formatos**, ambos gerados a partir do mesmo resultado:
   - **JSON estruturado**: lista de arestas (mesmo schema de `gold_lineage_edges`), pronta para consumo programático.
   - **Árvore textual no terminal**: endereço raiz no topo, ramificações indentadas por hop, uma linha por aresta, indicando direção (`←` backward, `→` forward), valor em sats e se é origem coinbase ou truncada.
3. **Flag `--export <arquivo.json>`**: salva o JSON em arquivo, além de imprimir a árvore no terminal.
4. **Flag `--direction`**: opcional, valores `backward` | `forward` | `both` (default `both`) — permite rodar só uma direção para pipelines mais rápidos.

## Requisitos funcionais — v2 (API, documentado agora, implementação fora do MVP)

1. **Endpoint**: `GET /trace?address=...&hops=...&direction=...` — retorna o mesmo JSON da CLI.
2. **Sem autenticação no v2** (uso pessoal/demo) — mas a spec já deixa registrado que rate limiting e autenticação são pré-requisito antes de qualquer exposição pública do endpoint.
3. Reaproveita o mesmo pipeline da CLI (não duplicar lógica — API é uma camada fina sobre o mesmo módulo).

## Requisitos não funcionais

- Tempo de resposta para `max_hops=2` em um endereço de teste pequeno: **não estimar previamente** — medir na primeira execução real e registrar o número obtido como baseline para futuras otimizações.
- Mensagens de erro devem ser legíveis por humano (não stack trace cru) para os casos: endereço ausente, endereço inválido, `hops` fora do intervalo permitido (ex.: negativo ou > 5).

## Critérios de aceite

1. Rodar a CLI ponta a ponta para um endereço de teste conhecido → gera JSON válido (parseável sem erro) e uma árvore textual legível no terminal.
2. Rodar com `--export saida.json` → arquivo criado, conteúdo idêntico ao impresso no terminal.
3. Rodar sem o argumento `--address` → mensagem de uso/ajuda clara, sem stack trace.
4. Rodar com `--hops -1` ou `--hops 999` → erro claro informando o intervalo permitido, sem executar o pipeline.
5. Rodar com `--direction backward` → resultado contém apenas arestas `direction = "backward"`.

## Fora de escopo (v1)

- UI gráfica ou visualização interativa do grafo (ex.: D3.js, Graphviz renderizado) — candidato a `spec-05` depois que o pipeline básico provar valor.
- Autenticação, deploy público, ou qualquer hospedagem da API.
- Cache de resultados entre execuções (cada chamada roda o pipeline; otimização de cache é uma iteração futura, só depois de ter uma medição real de tempo de execução).

## Definição de "pronto" desta spec

CLI funcional, testada contra um endereço público conhecido, com JSON exportável e árvore textual legível — sem depender de nenhum componente da v2 (API) para ser considerada completa.
