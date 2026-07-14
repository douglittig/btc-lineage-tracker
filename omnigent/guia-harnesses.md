# Guia — instalação, configuração e uso: Claude Code, Codex e Copilot no Omnigent

Guia operacional escrito em 2026-07-13, com base na documentação oficial do Omnigent (README, `docs/AGENT_YAML_SPEC.md`) e no estudo prévio em `instalacao.md`. O `instalacao.md` é o estudo (contexto, achados, pontos em aberto); este arquivo é o passo a passo executável por harness.

**Atualização importante sobre a "lista fechada" de executors:** o estudo de 13/07 registrou que o Omnigent só suportava claude, codex, cursor, opencode, hermes, pi e openai-agents. A lista **cresceu**: a spec YAML atual já documenta também `copilot` (adicionado via issue [#92](https://github.com/omnigent-ai/omnigent/issues/92) → PR #330), além de `kiro-native`, `antigravity`, `qwen` e `kimi`. Isso **não muda** a conclusão sobre o Genie Code (ele segue sem CLI/API/MCP exposto, então segue fora do Omnigent), mas mostra que a lista de harnesses é viva — vale reconferir a spec a cada decisão.

## 1. Instalar o Omnigent

Pré-requisitos nesta máquina (auditoria completa em `instalacao.md`): faltam Node 22 e tmux.

```bash
brew install node@22 tmux
```

Instalação — três opções, qualquer uma serve:

```bash
# script oficial (um comando, bom pra gravar)
curl -fsSL https://raw.githubusercontent.com/omnigent-ai/omnigent/main/scripts/install_oss.sh | sh

# via uv (mais controle; o Python 3.9 do sistema não serve, o uv resolve)
uv tool install -q --python 3.12 omnigent

# via Homebrew
brew install omnigent-ai/tap/omnigent
```

Atualização depois: `omni upgrade`.

## 2. Configuração geral

```bash
omnigent setup
```

O `setup` é o cofre de credenciais único — tudo que os harnesses precisam passa por ele:

- **API keys**: Anthropic, OpenAI
- **Assinaturas**: Claude Pro/Max e ChatGPT — dá pra usar a assinatura no lugar de API key (nosso caso com o Claude)
- **Gateways**: OpenRouter, Ollama, Azure
- **Workspaces Databricks**: o workspace entra como provedor de credencial/modelo (caminho do lab com a Free Edition)

## 3. Claude Code

**Pré-requisito:** Claude Code instalado (✅ já temos, 2.1.169) e assinatura Claude conectada no `omnigent setup`.

**Uso direto (harness nativo):**

```bash
omnigent claude
```

Isso abre o Claude Code *dentro* de uma sessão Omnigent — sandboxed (seatbelt no macOS), com políticas de budget/permissão aplicáveis na camada de harness e sessão compartilhável via server.

**Em agente custom (YAML):** usar `harness: claude-sdk` (ou `claude-native`) no bloco `executor`. É o executor recomendado quando o agente custom precisa de prompt próprio + tools (funções Python, MCP, sub-agentes).

## 4. Codex

**Pré-requisito:** Codex CLI instalado + conta OpenAI com assinatura ChatGPT (pendência já mapeada no roteiro do lab). Conectar a assinatura ou API key no `omnigent setup`.

**Uso direto (harness nativo):**

```bash
omnigent codex
```

**Em agente custom (YAML):** `harness: codex` (ou `codex-native`).

## 5. Copilot

Diferente dos dois acima, o Copilot **não tem comando direto** — não existe `omnigent copilot`. Ele roda como executor de um **agente custom**, via `omnigent run`.

**Pré-requisitos:**

1. Assinatura GitHub Copilot ativa na conta GitHub.
2. Um token GitHub **com acesso Copilot** — duas formas aceitas:
   - fine-grained PAT com a permissão **"Copilot Requests"**; ou
   - token OAuth do GitHub CLI (`gh auth login` e depois `gh auth token`).
   - ⚠️ PATs clássicos (`ghp_...`) **não são aceitos** pelo Copilot.

**Configuração (YAML do agente):**

```yaml
name: copilot_agent
prompt: <instruções do agente>
executor:
  harness: copilot             # alias: github-copilot
  model: claude-haiku-4.5      # id de modelo Copilot (ex.: gpt-5-mini); omitir = auto-select
  auth:
    type: api_key
    api_key: ${GH_TOKEN}       # token GitHub com acesso Copilot
```

**Uso:**

```bash
export GH_TOKEN=$(gh auth token)   # ou o fine-grained PAT
omnigent run caminho/do/agente/
```

**Observação pro lab:** o Copilot aqui é o *modelo/backend* servido pela assinatura Copilot dirigido pelo harness do Omnigent — não o plugin do VS Code. Se ele entrar na comparação do laboratório, entra nessa condição (mesma ressalva de honestidade que fizemos pro caminho B do Genie Code: dizer exatamente o que está sendo comparado).

## 6. Modo de uso — sessão do lab

```bash
omnigent                 # sessão interativa (alias: omni)
omnigent server start    # web UI em http://localhost:6767
omnigent host            # registra esta máquina como host de execução
OMNIGENT_AUTH_ENABLED=1 omnigent server start   # multi-usuário (convidados veem/comentam/comandam)
```

Fluxo da comparação (detalhado no roteiro de 8 passos do `instalacao.md`): subir o server, abrir `omnigent claude` e `omnigent codex` (e, se decidido, o agente custom Copilot via `omnigent run`) na mesma sessão, apontar todos pra mesma spec (`specs/spec-01-ingestao.md`) com cada um escrevendo apenas na sua pasta (`../claude-code`, `../codex`), e compartilhar a URL da sessão pra gravação. Genie Code segue em paralelo no workspace, fora do Omnigent.

## Fontes

- [GitHub — omnigent-ai/omnigent](https://github.com/omnigent-ai/omnigent) (README)
- [Spec YAML de agentes — docs/AGENT_YAML_SPEC.md](https://github.com/omnigent-ai/omnigent/blob/main/docs/AGENT_YAML_SPEC.md) (executors suportados, config do harness `copilot`)
- [Issue #92 — Add GitHub Copilot support](https://github.com/omnigent-ai/omnigent/issues/92) (fechada via PR #330)
- Estudo local: `instalacao.md` (pré-requisitos da máquina, conexão Databricks, achado Genie Code)
