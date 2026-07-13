# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A Bitcoin UTXO provenance tracker (given an address, trace where its sats came from and where they went) built as a **Spec-Driven Development lab**: the same specs are implemented independently and in parallel by three coding agents, each isolated in its own folder. All specs and documentation are written in Portuguese (pt-BR).

There is no implementation code yet — only specs. Commands below are the execution interfaces defined by the specs, not existing tooling.

## Critical rule: agent folder isolation

- `specs/` is the single source of truth. **Never modify specs while implementing** — implementations conform to specs, not the other way around.
- **Claude Code implements exclusively inside `claude-code/`.** Do not read from, copy from, or write to `genie-code/` or `codex/` — those are parallel implementations by other agents, and cross-contamination invalidates the comparison that is the point of the lab.
- `omnigent/` is **not** a fourth implementation. It is the control layer (Databricks Omnigent meta-harness) that orchestrates the three agents; it holds orchestration/session config only, never BTC-tracker implementation code.

## Reading order

Start with `specs/00-contexto-projeto.md` (goals, layered architecture, stack, MVP keystone), then `spec-01` through `spec-04` in order. Each spec is a vertical slice that must be **working** (not just written) before the next one starts:

1. `spec-01-ingestao.md` — paginated ingestion of an address's raw transactions from the mempool.space API into Delta table `bronze_address_txs`
2. `spec-02-modelagem-silver.md` — parse raw JSON into typed tables `silver_tx`, `silver_vin`, `silver_vout`
3. `spec-03-grafo-linhagem.md` — lineage graph via iterative BFS by hops (backward = coin origin, forward = where spent) into `gold_lineage_edges`
4. `spec-04-interface-saida.md` — CLI (`trace`) exposing the result as JSON + textual tree; HTTP API is documented but out of MVP

## Architecture

Medallion layers, all in PySpark + Delta Lake (deliberate choice even at MVP volume — the pipeline doubles as Spark teaching material, so **do not substitute Pandas** even where it would be simpler):

- **Bronze**: raw JSON per transaction, keyed `(address, txid)`, upserted via MERGE
- **Silver**: explicit-schema `from_json` + `explode` into tx/vin/vout tables (no schema inference)
- **Gold**: incremental BFS — spec-03 orchestrates re-running spec-01/02 for each new frontier address per hop; it does not reimplement ingestion. Key mechanics: visited-set anti-cycle, `max_fanout` truncation (default 500) to avoid combinatorial explosion, coinbase = end of backward chain (`is_coinbase_origin`, not an error)
- **Interface**: CLI first; API v2 must be a thin layer over the same module (no duplicated pipeline logic)

Cross-cutting requirements from the specs: every layer is idempotent (re-runs never duplicate rows — MERGE or full overwrite by key); reuse already-ingested Bronze/Silver data before making network calls; HTTP 429 and timeouts handled with exponential backoff (3 retries, starting at 1s); one pagination chain at a time per address.

## Stack constraints (v1)

- Python 3.11+, PySpark local mode, Delta Lake for all tables
- mempool.space public REST API called directly (no SDK — keeps the spec auditable); no key required
- No relational DB, no external services beyond the public API
- **Scope rule for everything**: no price, trading, fiat conversion, or market logic anywhere — the specs treat Bitcoin as a public high-volume dataset, and each spec's "Fora de escopo" section is binding

## Git workflow (binding)

- **Never commit directly to `main`.** `main` only receives promotions from `dev` after tests pass.
- Every piece of work starts in its own branch off `dev`, named `<type>/<kebab-case-slug>` — types: `feature/` (spec implementation), `fix/`, `docs/`, `chore/`.
- Flow: `<type>/*` → merge into `dev` (integration + testing happens here) → after tests validate (the spec's "Critérios de aceite" are the test cases), `dev` → `main`.
- **Delete the feature branch as soon as its PR merges into `dev`** — remote (the repo has auto-delete on merge enabled) and local (`git branch -d`). Branches are short-lived; long-lived branches are only `main` and `dev`.
- Commit messages follow `tipo: resumo` convention (`feat`, `fix`, `docs`, `test`, `refactor`, `chore`), small and descriptive.

## Execution interfaces defined by the specs

```
python -m btc_tracker.ingest  --address <addr> --output-path <bronze_delta>
python -m btc_tracker.silver  --input-path <bronze_delta> --output-path <silver_delta>
python -m btc_tracker.lineage --address <addr> --max-hops 2 --output-path <gold_delta>
python -m btc_tracker.trace   --address <addr> --hops <N> [--export out.json] [--direction backward|forward|both]
```

## Definition of done (v1)

`trace --address <addr> --hops 2` against a known public address produces a lineage graph manually validated against a block explorer, in reasonable time, with no duplicated data on re-runs. Each spec also lists its own acceptance criteria ("Critérios de aceite") — those are the test cases to satisfy.
