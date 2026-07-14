# Plano de Execução — BTC Lineage Tracker (Agente OpenCode)

## Resumo do Objetivo

Implementar um rastreador de proveniência de UTXOs Bitcoin que, dado um endereço, responde:
1. **De onde vieram os sats** (backward — seguindo vin.prev_txid)
2. **Para onde foram os sats gastos** (forward — seguindo vout → outspend)

Arquitetura em camadas medallion (Bronze → Silver → Gold), tudo em PySpark + Delta Lake. O keystone do MVP (definido em `00-contexto-projeto.md`) é: **rastro de um único endereço, profundidade configurável (default 2 hops)**.

---

## Ordem das Fatias e Dependências

| Ordem | Spec | Dependência | Motivo |
|---|---|---|---|
| 1 | spec-01-ingestao.md | nenhuma | Pipeline de entrada — sem isso, nada existe |
| 2 | spec-02-modelagem-silver.md | spec-01 | Estrutura o dado cru |
| 3 | spec-03-grafo-linhagem.md | spec-02 | Constrói o grafo (BFS) |
| 4 | spec-04-interface-saida.md | spec-03 | Expõe o resultado (CLI) |

Cada spec deve estar **funcionando** (não apenas escrita) antes de iniciar a próxima.

---

## Execução por Spec

### Spec-01 — Ingestão de transações por endereço

**Entregáveis:**
- Módulo de ingestão HTTP paginada (mempool.space REST API)
- Validação de endereço Bitcoin (checksum)
- Persistência em Delta Lake `bronze_address_txs`
- Idempotência via MERGE (upsert por `(address, txid)`)

**Estrutura de arquivos prevista em `opencode/`:**
```
opencode/
  src/btc_tracker/
    __init__.py
    ingest/
      __init__.py
      api.py          # chamadas HTTP + rate limiting
      validator.py    # validação de endereço
      bronze.py       # escrita Delta
    main.py           # CLI entry point (python -m btc_tracker.ingest)
  tests/
    test_ingest/
      test_api.py
      test_validator.py
      test_bronze.py
  requirements.txt
```

**Critérios de aceite (da spec):**
1. Endereços públicos conhecidos → `bronze_address_txs` contém ≥ transações visíveis no block explorer
2. Segunda execução → mesma contagem (idempotência)
3. Endereço inválido → erro claro antes de chamar API
4. Endereço sem transações → execução bem-sucedida, tabela vazia
5. HTTP 429 → recuperação via backoff exponencial

**Estratégia de teste:**
- Testes unitários para validação de endereço (mocks)
- Testes de integração com mempool.space (endereços públicos conhecidos)
- Teste de idempotência: rodar duas vezes, contar linhas com Delta
- Simular 429? Se API não retornar, usar patch/mock no HTTP client

---

### Spec-02 — Modelagem Silver (tx, vin, vout)

**Entregáveis:**
- Schema explícito PySpark para `from_json()`
- Três tabelas Delta: `silver_tx`, `silver_vin`, `silver_vout`
- Identificação de coinbase (`vin[0].is_coinbase = true`)
- Tratamento de outputs sem endereço decodificável (`OP_RETURN`, etc.)

**Estrutura de arquivos prevista em `opencode/`:**
```
opencode/
  src/btc_tracker/
    silver/
      __init__.py
      schema.py         # schemas explícitos
      transform.py      # from_json + explode
      models.py         # dados tipados
    main.py             # CLI entry point
  tests/
    test_silver/
      test_schema.py
      test_transform.py
```

**Critérios de aceite (da spec):**
1. Coinbase conhecida → `is_coinbase = true`, `prev_txid = null`
2. Conservação de valor: soma outputs + taxa == soma inputs (validar em 1 tx contra explorer)
3. `COUNT(DISTINCT txid)` em silver == bronze
4. Idempotência: rodar duas vezes → mesma contagem
5. EXISTS linha com `address = null` e `script_type` preenchido

**Estratégia de teste:**
- Fixture: tx coinbase conhecida (ex.: genesis ou bloco baixo)
-Fixture: tx normal com múltiplos inputs/outputs
- Verificar tipos de script esperados: p2pkh, p2sh, v0_p2wpkh, v0_p2wsh, v1_p2tr

---

### Spec-03 — Grafo de linhagem (BFS por hops)

**Entregáveis:**
- BFS iterativo (não recursão) por hops
- Expansão backward: seguir `vin.prev_txid` até origem
- Expansão forward: verificar `outspend` API, seguir para transação consumidora
- Anti-ciclo: conjunto de endereços visitados
- Limite de fan-out (default 500) com truncamento
- Coinbase =终点 backward (`is_coinbase_origin = true`)
- Tabela Delta `gold_lineage_edges`

**Estrutura de arquivos prevista em `opencode/`:**
```
opencode/
  src/btc_tracker/
    lineage/
      __init__.py
      bfs.py            # lógica de busca por hops
      backward.py       # expansão backward
      forward.py        # expansão forward
      gold.py           # escrita Delta
    main.py             # CLI entry point
  tests/
    test_lineage/
      test_bfs.py
      test_backward.py
      test_forward.py
```

**Critérios de aceite (da spec):**
1. Endereços de teste → hops 0 e 1 validados manualmente contra explorer
2. `max_hops=2` contém estritamente `max_hops=1` (superset)
3. Fan-out acima do limite → `truncated=true`, sem expansão
4. Idempotência: rodar duas vezes → sem duplicação
5. Coinbase origin → `from_address=null`, `is_coinbase_origin=true`

**Estratégia de teste:**
- Escolher endereço de doação open-source com histórico pequeno (dezenas de txs)
- Validar hops 0 e 1 manualmente contra mempool.space ou blockstream.info
- Teste de fan-out: criar mock de tx com 1000+ inputs (ou usar tx real se encontrar)

**Riscos:**
- Explosão combinatória: mitigar com `max_fanout` e `max_hops` baixos inicialmente
- Latência de API: reutilizar Bronze/Silver antes de chamar API
- Ciclo no grafo: conjunto de visited addresses (obrigatório)

---

### Spec-04 — Interface de saída (CLI → API)

**Entregáveis:**
- Comando `python -m btc_tracker.trace --address <addr> --hops <N>`
- Pipeline completo: ingest → silver → lineage (orquestrado)
- Saída JSON estruturado
- Saída árvore textual no terminal (indentação por hop, setas ←/→)
- Flag `--export <arquivo.json>`
- Flag `--direction backward|forward|both`

**Estrutura de arquivos prevista em `opencode/`:**
```
opencode/
  src/btc_tracker/
    cli/
      __init__.py
      trace.py          # orquestração + saída CLI
      renderer.py       # árvore textual
    main.py             # entry point único
  tests/
    test_cli/
      test_trace.py
      test_renderer.py
```

**Critérios de aceite (da spec):**
1. CLI ponta a ponta → JSON válido + árvore legível
2. `--export saida.json` → arquivo criado, conteúdo idêntico
3. Sem `--address` → mensagem de uso clara, sem stack trace
4. `--hops -1` ou `999` → erro com intervalo permitido
5. `--direction backward` → apenas backward edges

**Estratégia de teste:**
- Snapshot test: comparar árvore textual com saída esperada
- Teste de argumentos inválidos: argparse já cuida da maior parte
- Baseline de tempo: medir primeira execução real, registrar

---

## Decisões Técnicas

| Decisão | Justificativa |
|---|---|
| PySpark + Delta Lake (mesmo com volume pequeno) | Pipeline é também material de aula de Spark Fundamentals — escolha deliberada |
| REST puro (sem SDK) para mempool.space | Mantém a spec auditável e o código transparente |
| Idempotência como regra (MERGE / overwrite) | Evita duplicação em reexecuções, crítico para layered architecture |
| BFS iterativo (não recursivo) | Evita stack overflow, alinhado com Spark (DataFrame ops) |
| CoinBASE como fim de linha backward | Correto semânticamente — não é erro, é origem do valor |
| Sem mora de preço/mercado (nunca) | Regra de escopo do projeto — dataset público adversarial, não trading |

---

## Riscos e Mitigações

| Risco | Impacto | Mitigação |
|---|---|---|
| API mempool.rate limiting (429) bloqueia execução | Alto | Backoff exponencial, 3 retries, uma cadeia de paginação por vez |
| Explosão combinatória (endereço com milhares de txs) | Alto | `max_fanout=500`, `max_hops` default 2, truncamento com flag |
| Endereços sem símbolo decodificável (OP_RETURN etc.) | Baixo | Salvar `address=null`, preencher `script_type`, logar warning |
| Coinbase origin mal tratado (erro vs. fim de linha) | Médio | Identificar via `vin[0].is_coinbase`, marcar explicitamente |
| Ciclo no grafo (endereços reentrada) | Médio | Conjunto de visited addresses (obrigatório) |
| Falha de rede (timeout) | Médio | Retry com backoff (3 tentativas) |

---

## Milestones e Definição de Pronto

### Milestone 1 — Spec-01 (Ingestão)
- [ ] Validação de endereço funciona (checksum)
- [ ] Paginação busca todas as txs (vazio sinaliza fim)
- [ ] Delta table `bronze_address_txs` criada com schema correto
- [ ] Idempotência verificada (duas execuções → mesma contagem)
- [ ] 429 tratado com backoff

### Milestone 2 — Spec-02 (Silver)
- [ ] Schema explícito definido sem inferência automática
- [ ] Três tabelas criadas: `silver_tx`, `silver_vin`, `silver_vout`
- [ ] Coinbase identificada corretamente
- [ ] Conservation de valor validada em 1 tx manual
- [ ] Idempotência verificada

### Milestone 3 — Spec-03 (Grafo)
- [ ] BFS iterativo implementado (backward + forward)
- [ ] Anti-ciclo funcionando (visited set)
- [ ] Fan-out truncamento funciona
- [ ] gold_lineage_edges criado com schema completo
- [ ] Hops 0 e 1 validados manualmente contra block explorer

### Milestone 4 — Spec-04 (CLI)
- [ ] Comando `trace --address --hops` funciona ponta a ponta
- [ ] JSON estruturado exportável
- [ ] Árvore textual legível no terminal
- [ ] Flags `--export`, `--direction` funcionam
- [ ] Mensagens de erro amigáveis (sem stack trace)

### Definição de "Pronto" do Projeto (v1)
Rodar `trace --address <endereço_público> --hops 2` e obter:
- Grafo de linhagem correto (validado manualmente contra block explorer)
- Tempo de execução razoável (mesurar baseline)
- Reexecuções não duplicam dados
- Sem erros de rate limiting ou rede

---

## Considerações Finais

- **Nada fora de `opencode/`** — isolamento strito conforme regra de agent folder isolation
- **Sem modificações nas specs** — elas são fonte única de verdade
- **Cada spec é entregável isoladamente** — testar antes de avançar
- **Commit message padrão**: `feat: <descrição>`, commits pequenos
- **Branch naming**: `feature/<slug>` para cada spec
- **PR workflow**: master → dev (agent merges), dev → main (owner merges)

---

*Documento gerado em 14/07/2026 pelo agente OpenCode.*
