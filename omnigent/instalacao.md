# Estudo — Instalação do Omnigent

Estudo feito em 2026-07-13 para preparar a instalação do Omnigent nesta máquina (macOS Intel). Fontes ao final. Este documento é também matéria-prima de conteúdo: o roteiro de instalação vira demo gravável do laboratório multi-agente.

## O que é (recap de 30 segundos)

Omnigent é o meta-harness open-source da Databricks (Apache 2.0, lançado 16/06/2026). Ele não substitui os harnesses de codificação — fica **uma camada acima**: um *runner* embrulha qualquer agente (Claude Code, Codex, Cursor, Pi, custom) numa sessão sandboxed com API uniforme, e um *server* adiciona políticas (budget de custo, permissões — enforcement na camada de harness, não via prompt), além de compartilhamento da sessão ao vivo via URL (web/mobile/desktop, com convidados podendo ver, comentar e comandar).

Para o nosso laboratório, isso resolve exatamente o problema: rodar a mesma spec em Claude Code + Codex + Genie Code lado a lado numa sessão só, gravável.

## Pré-requisitos

| Requisito | Status nesta máquina (13/07) | Ação |
|---|---|---|
| Python 3.12+ | ❌ sistema tem 3.9.6 | resolvido pelo próprio `uv` (`--python 3.12`) |
| `uv` | ✅ 0.11.19 | — |
| `git` | ✅ 2.54.0 | — |
| Node.js 22 LTS + `npm` | ❌ não instalado | `brew install node@22` |
| `tmux` | ❌ não instalado | `brew install tmux` |
| Sandbox de SO | ✅ macOS usa seatbelt (nativo) | nada a fazer (bubblewrap é só Linux) |
| Claude Code | ✅ 2.1.169 | — |
| Codex CLI | ❌ não instalado | depende de conta OpenAI + assinatura (pendência já mapeada) |
| Databricks CLI (opcional) | ❌ não instalado | `brew tap databricks/tap && brew install databricks` |

## Instalação (opções)

**Script automatizado (caminho recomendado pra gravar — um comando só):**

```bash
curl -fsSL https://raw.githubusercontent.com/omnigent-ai/omnigent/main/scripts/install_oss.sh | sh
```

**Via uv (mais controle, bom pra explicar em aula):**

```bash
uv tool install -q --python 3.12 omnigent
```

**Via Homebrew:**

```bash
brew install omnigent-ai/tap/omnigent
```

Também existe `pip install omnigent`, mas nesta máquina o Python de sistema é 3.9 — usar `uv` ou brew.

## Configuração pós-instalação

```bash
omnigent setup
```

O `setup` gerencia credenciais de: API keys (Anthropic, OpenAI), **assinaturas** (Claude Pro/Max, ChatGPT — importante: dá pra usar a assinatura em vez de API key), gateways compatíveis (OpenRouter, Ollama, Azure) e **workspaces Databricks**.

## Uso — comandos que interessam ao lab

```bash
omnigent            # sessão interativa (alias: omni)
omnigent claude     # roda o harness Claude Code
omnigent codex      # roda o harness Codex
omnigent run <dir>  # roda um agente custom (YAML)

omnigent server start   # server local (web UI em http://localhost:6767)
omnigent host           # registra esta máquina como host de execução
OMNIGENT_AUTH_ENABLED=1 omnigent server start   # multi-usuário (convidar gente pra sessão)

omni upgrade        # atualizar
```

## Genie Code como agente customizado — achado (13/07, resolve o ponto em aberto nº 1)

A documentação do Omnigent **não suporta o Genie Code como harness**. Os executors suportados são: `claude-sdk`/`claude-native`, `codex`/`codex-native`, `cursor`/`cursor-native`, `opencode`, `hermes`/`hermes-native`, `pi`/`pi-native`, `openai-agents`. "Agente customizado" no Omnigent significa **prompt + tools (funções Python, MCP, sub-agentes) rodando sobre um desses executors** — não um mecanismo pra embrulhar um agente externo arbitrário.

Do lado do Genie Code: ele roda **dentro do workspace** (notebooks, SQL editor, Lakeflow, dashboards, MLflow), sem CLI/REST/SDK documentados — ele *consome* MCP servers, não expõe um.

Caminhos reais pro lab, do mais honesto ao mais especulativo:

- **(A) Genie Code fora do Omnigent:** o Omnigent orquestra Claude Code + Codex; o Genie Code roda em paralelo no workspace e entra na comparação manualmente. Menos elegante, mas é o que a documentação sustenta hoje.
- **(B) Agente custom "Databricks-powered":** YAML custom com executor suportado (ex.: `claude-sdk`) usando modelo servido pelo workspace (extra `databricks` do Omnigent). Emula um agente Databricks dentro da sessão — mas **não é o produto Genie Code**, e a comparação precisa dizer isso.
- **(C) Futuro:** se o Genie Code expuser API/MCP, ainda entraria como *tool* de um agente custom, não como harness — quem dirige continua sendo o executor.

```yaml
# formato real de agente custom (opção B)
name: databricks_agent
prompt: <instruções>
executor:
  harness: claude-sdk
tools:
  minha_funcao:
    type: function
    callable: modulo.funcao
  mcp_tools:
    type: mcp
    url: https://exemplo.com/mcp
```

**Implicação pro repo:** o `00-contexto-projeto.md` e o `omnigent/README.md` assumem "Genie Code entra como agente customizado" — a documentação atual não confirma isso. Decisão pendente do Doug: caminho A (3 agentes, orquestração parcial) ou B (3 agentes no Omnigent, mas o terceiro não é Genie Code de verdade). O gap em si é conteúdo: "o que o meta-harness da Databricks ainda não orquestra é o agente da própria Databricks".

## Conexão com Databricks

Dois modos distintos — não confundir:

1. **Omnigent local/self-hosted conectando a um workspace** — via `omnigent setup` (workspace entra como provedor de credencial/modelo). É o caminho do nosso lab.
2. **Omnigent on Databricks (gerenciado)** — server operado pela Databricks, integrado ao identity provider do workspace. Requer preview **Omnigent** habilitado e região com Unity AI Gateway; sandboxes exigem preview **Sandbox**. A documentação **não confirma disponibilidade na Free Edition** — provável limitação, já que previews e AI Gateway costumam ficar fora dela.

**Ponto em aberto nº 2:** testar na Free Edition o que funciona: (a) `omnigent setup` apontando pro workspace free deve funcionar como credencial; (b) o modo gerenciado provavelmente não. O teste em si é conteúdo ("o que a Free Edition te dá e o que ela não te dá").

## Roteiro de instalação pro lab (ordem proposta)

1. `brew install node@22 tmux` — pré-requisitos que faltam
2. Instalar Omnigent (script oficial ou `uv tool install`)
3. `omnigent setup` — conectar assinatura Claude (já temos) e workspace Databricks Free Edition (pendência: criar)
4. `omnigent claude` — smoke test com o harness que já temos
5. Assinar Codex (pendência OpenAI) → `omnigent codex` — segundo harness
6. Investigar YAML custom pro Genie Code (ponto em aberto nº 1)
7. `omnigent server start` + compartilhar sessão — validar o fluxo de gravação/colaboração
8. Rodar `spec-01` nos harnesses conectados, lado a lado — primeira comparação real

## Ganchos de conteúdo

- **"Instalei o meta-harness da Databricks num Mac Intel"** — walkthrough dos passos 1–4, terminando com Claude Code rodando *dentro* do Omnigent.
- **"3 agentes, 1 spec, 1 sessão"** — o momento passo 8; é a tese do laboratório inteiro em um take.
- **"O que a Free Edition aguenta?"** — resultado do ponto em aberto nº 2; conecta com o pilar Free Edition = porta de entrada.
- **Governança na camada de harness** — budget de custo e permissões por política, não por prompt; é harness engineering na prática ("fundamentos antes do hype" aplicado a IA agêntica).

## Fontes

- [GitHub — omnigent-ai/omnigent](https://github.com/omnigent-ai/omnigent)
- [Databricks Blog — Introducing Omnigent](https://www.databricks.com/blog/introducing-omnigent-meta-harness-combine-control-and-share-your-agents)
- [Docs — Omnigent on Databricks](https://docs.databricks.com/aws/en/omnigent/)
