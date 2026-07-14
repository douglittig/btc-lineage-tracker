# PLANO.md — Plano de execução das specs (agente: Claude Code)

> Fase de planejamento do laboratório SDD. Nenhum código foi escrito ainda — este documento e os planos por spec em [`plano/`](plano/) são o entregável desta fase. Fonte única de verdade: `specs/` (este plano conforma-se às specs; nunca o contrário).

## 1. Objetivo e arquitetura

**O que será construído:** um rastreador de proveniência de UTXO na rede Bitcoin. Dado um endereço, o sistema responde de onde vieram os sats (cadeia backward até coinbase ou `max_hops`) e para onde foram os já gastos (cadeia forward), como um grafo de linhagem.

**Keystone (fatia vertical mínima, definida em `specs/00-contexto-projeto.md`):** _rastro de um único endereço, profundidade configurável (default 2 hops)_. Tudo neste plano serve diretamente a essa fatia; qualquer coisa fora dela (API HTTP, clusterização, full node, UI) fica fora do v1, conforme as seções "Fora de escopo" de cada spec — que são vinculantes.

**Arquitetura em camadas (medallion), toda em PySpark + Delta Lake** — escolha deliberada da spec mesmo com volume pequeno no MVP, porque o pipeline é também material didático de Spark. Não substituir por Pandas em nenhuma hipótese:

| Camada | Tabela(s) | Conteúdo | Spec |
|---|---|---|---|
| Bronze | `bronze_address_txs` | JSON cru por transação, chave `(address, txid)`, upsert via MERGE | spec-01 |
| Silver | `silver_tx`, `silver_vin`, `silver_vout` | parse tipado com `from_json` + schema explícito + `explode` | spec-02 |
| Gold | `gold_lineage_edges` | grafo de linhagem via BFS iterativo por hops, incremental | spec-03 |
| Interface | — | CLI `trace` (JSON + árvore textual); API HTTP documentada mas fora do MVP | spec-04 |

**Fonte de dados:** API REST pública mempool.space, chamada direta sem SDK (mantém a spec auditável), sem chave.

## 2. Ordem de execução e dependências

As specs formam uma cadeia linear — cada fatia precisa estar **funcionando** (critérios de aceite passando contra dados reais), não apenas escrita, antes da próxima começar:

```
spec-01 (ingestão Bronze)
   └─► spec-02 (modelagem Silver)            [precisa de bronze_address_txs populada]
          └─► spec-03 (grafo Gold, BFS)      [precisa de silver_tx/vin/vout; ORQUESTRA re-execução de 01+02 por fronteira]
                 └─► spec-04 (CLI trace)     [camada fina sobre 01+02+03; zero lógica de pipeline própria]
```

Duas dependências merecem destaque porque moldam o design desde a spec-01:

- **spec-03 não reimplementa ingestão** — ela chama, como biblioteca, as funções de 01 e 02 para cada novo endereço da fronteira do BFS. Logo, `ingest` e `silver` nascem como **funções importáveis** com um wrapper `__main__` fino, não como scripts monolíticos.
- **spec-04 (CLI) e a futura API v2 compartilham o mesmo módulo** — a lógica de orquestração ponta a ponta vive em um módulo interno reutilizável; a CLI só faz parse de argumentos e renderização.

## 3. Estrutura de arquivos prevista (dentro de `claude-code/`)

```
claude-code/
  PLANO.md                      # este arquivo
  plano/                        # planos detalhados por spec (esta fase)
    spec-01.md  spec-02.md  spec-03.md  spec-04.md
  # ——— fase de implementação (ainda não existe) ———
  README.md                     # como rodar esta implementação
  pyproject.toml                # deps: pyspark, delta-spark, requests, pytest
  btc_tracker/
    __init__.py
    config.py                   # constantes: base URL, retries, backoff, max_fanout, paths default
    session.py                  # factory da SparkSession local com extensão Delta
    address.py                  # validação de endereço (base58check + bech32/bech32m)
    api_client.py               # cliente REST mempool.space: paginação, 429, timeout, backoff
    schemas.py                  # schemas Spark explícitos (bronze e tx da API)
    ingest.py                   # spec-01: run_ingest(address, output_path) + __main__
    silver.py                   # spec-02: run_silver(input_path, output_path) + __main__
    lineage.py                  # spec-03: run_lineage(address, max_hops, ...) + __main__
    trace.py                    # spec-04: orquestração ponta a ponta + __main__ da CLI
    render.py                   # spec-04: árvore textual e serialização JSON
  tests/
    unit/                       # validação de endereço, backoff, parsing, renderização (sem rede)
    integration/                # critérios de aceite por spec, contra API real / fixtures
    fixtures/                   # respostas JSON gravadas da API p/ testes offline
  data/                         # tabelas Delta locais geradas em execução (gitignored)
```

Interfaces de execução exatamente como definidas nas specs (`python -m btc_tracker.ingest|silver|lineage|trace`).

## 4. Resumo por spec (detalhe completo em `plano/spec-0X.md`)

| Spec | Entregáveis concretos | Validação principal | Plano detalhado |
|---|---|---|---|
| 01 — Ingestão | `address.py`, `api_client.py`, `ingest.py`, `config.py`, `session.py`, `schemas.py` (parte bronze); tabela `bronze_address_txs` | 5 critérios de aceite da spec: contagem vs. block explorer, idempotência (2ª run não muda contagem), erro pré-rede p/ endereço inválido, endereço vazio sem exceção, recuperação de 429 | [plano/spec-01.md](plano/spec-01.md) |
| 02 — Silver | `schemas.py` (schema tx completo), `silver.py`; tabelas `silver_tx`, `silver_vin`, `silver_vout` | coinbase com `prev_txid=null`, conservação de valor vs. explorer, contagem distinta tx == bronze, idempotência, `address=null` p/ OP_RETURN sem erro | [plano/spec-02.md](plano/spec-02.md) |
| 03 — Grafo | `lineage.py` (BFS iterativo orquestrando 01+02 por fronteira); tabela `gold_lineage_edges` | hops 0–1 validados manualmente vs. explorer, superset (`hops=2` ⊇ `hops=1`), `truncated=true` sem expansão, idempotência, coinbase como fim de linha (`is_coinbase_origin=true`, `from_address=null`) | [plano/spec-03.md](plano/spec-03.md) |
| 04 — CLI | `trace.py`, `render.py`; comando `trace` com `--export` e `--direction` | JSON parseável + árvore legível ponta a ponta, export idêntico ao terminal, erros humanos p/ address ausente/`hops` fora de [0,5], filtro por direção | [plano/spec-04.md](plano/spec-04.md) |

## 5. Decisões técnicas

1. **Módulos-função, CLI fina.** Cada spec expõe `run_*()` puro-Python chamável programaticamente; o bloco `__main__` só faz argparse + chamada. É o que permite à spec-03 orquestrar 01/02 e à spec-04 (e futura API v2) reusar tudo sem duplicação.
2. **Validação de endereço sem rede e sem dependência pesada.** Implementar base58check (legacy/P2SH) e bech32/bech32m (segwit/taproot) com checksum — algoritmos públicos (BIP-173/350), ~100 linhas, mantêm a spec auditável e cobrem o critério "erro claro antes de qualquer chamada de rede".
3. **Cliente HTTP com `requests` puro**, sessão única, backoff exponencial (3 tentativas, início 1s) para 429 e timeout, **uma cadeia de paginação por vez por endereço** (sequencial por construção — sem paralelismo de requisições).
4. **Idempotência por camada:** Bronze = MERGE Delta por `(address, txid)`; Silver = overwrite completo por execução (derivação determinística do Bronze — mais simples e igualmente idempotente); Gold = MERGE por chave de aresta `(txid, from_address, to_address, direction)`.
5. **Reuso antes de rede:** antes de chamar a API para um endereço, consultar Bronze; a checagem de "output gasto?" (forward) usa `GET /tx/:txid/outspend/:vout` só quando o gasto não é resolvível pelos dados já em Silver.
6. **Schema explícito sempre** (`from_json` com `StructType` completo do formato mempool.space) — inferência proibida pela spec-02.
7. **Sem preço/fiat/mercado em lugar nenhum** — regra de escopo global vinculante.

## 6. Premissas e riscos

| # | Risco / premissa | Mitigação |
|---|---|---|
| R1 | Rate limit da mempool.space durante BFS (muitos endereços na fronteira) | backoff exponencial já exigido pela spec; expansão sequencial; reuso agressivo de Bronze/Silver; endereço de teste pequeno |
| R2 | Explosão combinatória em transações de consolidação | `max_fanout=500` da spec-03: marca `truncated=true` e não expande |
| R3 | Ciclos no grafo (endereço reaparece em hop posterior) | visited-set de endereços, exigido pela spec-03 |
| R4 | Endereço de altíssimo volume torna a paginação (25 tx/página) impraticável | escolher endereço de teste com poucas dezenas de tx; volume alto não é meta do keystone |
| R5 | Overhead do Spark local para volumes minúsculos | aceito por decisão de projeto (fim didático); medir baseline na 1ª execução real, como manda a spec-04, sem otimização prematura |
| R6 | Campos da API fora do schema explícito (tipos de script novos etc.) | spec-02 já prevê: gravar como está + warning, nunca falhar; schema cobre só os campos contratados |
| P1 | Premissa: API mempool.space estável e disponível sem chave | fixtures gravadas para testes offline; full node é fase 2 explícita |
| P2 | Premissa: Python 3.11+, JVM disponível para PySpark local | documentar pré-requisitos no README da implementação |

## 7. Marcos (milestones) e definição de pronto por fatia

Cada marco = uma branch `feature/spec-0X-*` → PR para `dev` (testes = critérios de aceite da spec) → após validação, PR da mesma branch para `main` (gate do dono). Nunca commitar direto em `main`/`dev`.

| Marco | Entrega | Pronto quando |
|---|---|---|
| **M0** (este PR) | Plano aprovado | `PLANO.md` + `plano/` mergeados |
| **M1** — spec-01 | `ingest` funcionando | 5 critérios de aceite da spec-01 passando contra endereço público real; log de novas × existentes |
| **M2** — spec-02 | `silver` funcionando | 5 critérios da spec-02, incluindo conservação de valor validada contra block explorer |
| **M3** — spec-03 | `lineage` funcionando | 5 critérios da spec-03; hops 0–1 conferidos manualmente contra explorer |
| **M4** — spec-04 | CLI `trace` ponta a ponta | 5 critérios da spec-04 + **definição de pronto do projeto**: `trace --address <addr> --hops 2` gera grafo correto validado contra block explorer, em tempo razoável (baseline medido e registrado), sem duplicar dado em reexecução |

**Estratégia de teste transversal:** unit tests offline (validação de endereço, backoff com mock, parsing sobre fixtures, renderização da árvore) + testes de integração que executam literalmente cada critério de aceite da spec — os critérios são a suíte, não um proxy dela. Fixtures JSON gravadas da API permitem rodar a maior parte da suíte sem rede; a validação final de cada marco roda contra a API real e um block explorer de referência.
