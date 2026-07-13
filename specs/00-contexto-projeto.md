# BTC Lineage Tracker — Contexto do Projeto

## O que é

Um rastreador de proveniência de moedas na rede Bitcoin. Dado um endereço, responde duas perguntas: **de onde vieram os sats associados a ele** (seguindo a cadeia de transações pra trás) e **pra onde foram os que já saíram** (seguindo pra frente). É um grafo de linhagem de UTXO, não um dashboard de preço.

## Por que existe

Dois motivos, na mesma proporção:

1. **Produto real.** O interesse do autor por Bitcoin é soberania e verificação técnica — "não confie, verifique" aplicado a um dataset público e adversarial. O rastreador é uma ferramenta que qualquer pessoa usa pra auditar a própria carteira ou entender a origem de fundos, sem depender de block explorer de terceiro.
2. **Laboratório de engenharia + desenvolvimento agêntico.** O projeto é construído via SDD (Spec-Driven Development), com a mesma especificação entregue a múltiplos agentes de codificação (Genie Code, Claude Code, Codex) rodando em paralelo, cada um em sua própria pasta, pra comparar abordagem e resultado — Claude Code e Codex orquestrados numa sessão única via Omnigent (meta-harness open-source da Databricks); Genie Code roda em paralelo direto no workspace Databricks, pois o Omnigent não o suporta como harness (ver `omnigent/instalacao.md`). Isso é conteúdo em si — validação prática de harness engineering, não opinião.

## Regra de escopo (vale pra toda spec deste projeto)

**Fundamentos antes do hype.** Nenhuma spec deste projeto trata de preço, trading, especulação ou "sinais de mercado". O dataset é tratado como qualquer outro dataset público de alto volume — o valor está na engenharia (ingestão, modelagem, grafo, escala), não na narrativa cripto.

## Arquitetura em camadas (e o gancho de conteúdo de cada uma)

| Camada | O que faz | Curso/conteúdo relacionado |
|---|---|---|
| Bronze | Ingestão crua de transações via API pública, sem transformação | Spark Fundamentals (schema, partições) |
| Silver | Parse estruturado em tabelas tipadas (tx, vin, vout) | Spark Fundamentals → Zero ao Streaming |
| Gold | Grafo de linhagem (BFS por hops), incremental | Zero ao Streaming |
| Interface | CLI / API que expõe o grafo | Avançado (o produto em si) |

Cada camada é implementada via Spark (PySpark) mesmo quando o volume do MVP não exigiria — a escolha é deliberada, porque o pipeline também é matéria-prima de aula.

## Fonte de dados

MVP usa a API pública [mempool.space](https://mempool.space/docs/api/rest) (sem custo, sem chave). Rodar um full node Bitcoin próprio (`bitcoind`) fica reservado para uma fase 2, quando a tese de soberania pedir eliminar a dependência de terceiro — não bloqueia o MVP.

## Keystone (fatia vertical mínima)

**Rastro de um único endereço, profundidade configurável (default 2 hops).** Não é "a plataforma" — é a menor fatia que já responde a pergunta real ponta a ponta. Tudo que não serve diretamente a essa fatia é fora de escopo do v1 (ver seção "Fora de escopo" de cada spec).

## Sequência das specs

1. `spec-01-ingestao.md` — busca as transações cruas de um endereço
2. `spec-02-modelagem-silver.md` — estrutura o dado cru em tabelas tipadas
3. `spec-03-grafo-linhagem.md` — constrói o grafo de proveniência (BFS por hops)
4. `spec-04-interface-saida.md` — expõe o resultado (CLI, depois API)

Cada spec é uma fatia entregável e testável isoladamente — a spec seguinte depende da anterior estar funcionando, não apenas escrita.

## Como usar isso no laboratório multi-agente

Estrutura de repositório:

```
btc-lineage-tracker/
  specs/                    # este conteúdo, fonte única de verdade
  genie-code/                # implementação gerada pelo Genie Code
  claude-code/                # implementação gerada pelo Claude Code
  codex/                # implementação gerada pelo Codex
  omnigent/                # NÃO é uma 4ª implementação — é a camada de controle
```

Cada pasta de agente recebe exatamente as mesmas specs, sem contexto adicional além do que está escrito aqui — isso é o que torna a comparação justa.

`omnigent/` é diferente das outras três: é o [Omnigent](https://github.com/omnigent-ai/omnigent), meta-harness open-source da Databricks (lançado 06/2026) que orquestra Claude Code, Codex e agentes customizados a partir de um único ponto de controle. **Nota (13/07/2026):** Genie Code *não* entra como agente customizado — na doc do Omnigent, agente custom roda sobre um executor da lista suportada (que não inclui Genie Code), e o Genie Code não expõe CLI/API/MCP; ele roda em paralelo no workspace e a comparação com ele é manual — ver `omnigent/instalacao.md`. Serve pra rodar a mesma spec nos três agentes lado a lado numa sessão só, com colaboração em tempo real (dá pra convidar alguém pra ver/comentar/comandar) — útil tanto pra comparar resultado quanto pra gravar a comparação como conteúdo. Detalhes em `omnigent/README.md`.

## Stack técnico (v1)

- Python 3.11+
- PySpark (modo local) + Delta Lake para todas as tabelas
- Requisições HTTP à API mempool.space (sem SDK — REST puro, para manter a spec auditável)
- Sem banco relacional, sem serviço externo além da API pública

## Definição de "pronto" do projeto (v1)

Rodar `trace --address <endereco> --hops 2` para um endereço público conhecido (ex.: endereço de doação de projeto open source) e obter um grafo de linhagem correto, validado manualmente contra um block explorer, em tempo de execução razoável, sem duplicar dado em reexecuções.
