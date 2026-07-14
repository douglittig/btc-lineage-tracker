# omnigent/ — camada de controle do laboratório

Esta pasta **não** é uma quarta implementação paralela da spec. O [Omnigent](https://github.com/omnigent-ai/omnigent) (Databricks, open-source, lançado em 06/2026) é um meta-harness: ele fica *acima* dos agentes de codificação e orquestra Claude Code, Codex e agentes customizados a partir de um único ponto de controle.

## Papel no projeto

- Conectar `../claude-code` e `../codex` num único ponto de controle, como harnesses nativos do Omnigent.
- **Genie Code fica fora do Omnigent** (achado de 13/07/2026, ver `instalacao.md`): a doc do Omnigent só suporta uma lista fechada de executors (claude, codex, cursor, opencode, hermes, pi, openai-agents — lista que cresce: `copilot` e outros já entraram, ver `guia-harnesses.md`), e "agente customizado" ali significa prompt + tools *sobre* um desses executors — não um wrapper pra agente externo. O Genie Code, por sua vez, não expõe CLI/API/MCP (roda só no workspace e *consome* MCP). Então `../genie-code` é implementado direto no workspace Databricks, em paralelo, e entra na comparação manualmente.
- Rodar a mesma spec nos harnesses conectados lado a lado, em vez de alternar terminal por terminal.
- Governança e sandboxing por sessão (controle de custo, políticas por agente).
- Colaboração em tempo real — a sessão fica acessível via web/mobile/desktop, dá pra convidar alguém pra ver, comentar ou comandar, o que é diretamente útil pra gravar conteúdo (em vez de printar 3 terminais separados depois, a comparação acontece ao vivo numa sessão só).

## O que vai aqui

Configuração da sessão de orquestração (conexão com cada agente, políticas, roteiro de execução das specs). Nenhum código de implementação da spec do BTC tracker propriamente dita — isso continua nas pastas dos 3 agentes.

- `instalacao.md` — estudo de instalação (pré-requisitos da máquina, conexão Databricks, achado Genie Code, roteiro de 8 passos)
- `guia-harnesses.md` — passo a passo executável de instalação, configuração e uso por harness: Claude Code, Codex e Copilot

## Status

Ainda não instalado. Estudo de instalação concluído em 13/07/2026 — ver `instalacao.md` (pré-requisitos auditados, roteiro de 8 passos, conexão com Databricks). Próximo passo: instalar pré-requisitos (Node 22, tmux) e o Omnigent, conectar Claude Code + Codex. A questão "Genie Code plugga como agente customizado?" foi investigada e respondida: **não plugga** — ver seção acima e `instalacao.md`.
