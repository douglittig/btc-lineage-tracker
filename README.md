# btc-lineage-tracker

Rastreador de proveniência de UTXO na rede Bitcoin: dado um endereço, mostra de onde vieram os sats e pra onde foram. Projeto de engenharia de dados (Spark + Delta Lake) e laboratório de desenvolvimento agêntico via SDD — a mesma spec é implementada em paralelo por múltiplos agentes de codificação, para comparar abordagem e resultado.

Sem foco em preço, trading ou especulação — fundamentos antes do hype.

## Estrutura

- `specs/` — fonte única de verdade. Comece por `specs/00-contexto-projeto.md`, depois `spec-01` a `spec-04` em ordem.
- `genie-code/`, `claude-code/`, `codex/` — implementação independente da mesma spec por cada agente, isolada em sua própria pasta.
- `omnigent/` — não é uma 4ª implementação. É a camada de controle (meta-harness Databricks) que orquestra Claude Code e Codex a partir de um único ponto; Genie Code roda em paralelo no workspace, fora do Omnigent — ver `omnigent/README.md`.

## Como começar

Leia `specs/00-contexto-projeto.md` primeiro — define objetivo, arquitetura em camadas, stack e a fatia mínima (keystone) do MVP. Cada spec numerada depois disso é uma fatia vertical entregável e testável isoladamente.
