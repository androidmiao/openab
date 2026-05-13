---
title: "Agent Backends — Kiro, Claude Code, Codex, Gemini, OpenCode, Copilot, Cursor"
type: agent
sources:
  - ../docs/kiro.md
  - ../docs/claude-code.md
  - ../docs/codex.md
  - ../docs/gemini.md
  - ../docs/opencode.md
  - ../docs/copilot.md
  - ../docs/cursor.md
  - ../README.md
related: [../architecture/acp-protocol, ../modules/acp, ../deployment/docker-images, ../deployment/helm-chart, ../adrs/adr-006-context-aware-token]
updated: 2026-05-12
---

OpenAB 不綁定特定 AI 模型,只要 CLI 能講 ACP(JSON-RPC over stdio)就能用。本
頁比較目前支援的七個 backend:**Kiro CLI(預設)、Claude Code、Codex、Gemini、
OpenCode、Copilot CLI、Cursor**。每個 backend 都有獨立的 Docker image、獨立
的認證流程、可能還有自家的 ACP 行為怪癖(像 kiro-cli 不發 `configOptions` 而
發 `models`/`modes`)。

OpenAB is agnostic to which AI model is behind the CLI — any ACP-compatible
binary works. This page compares the seven supported backends side-by-side
and links to the per-agent setup doc for each.

---

## At a glance / 一覽

| Agent | CLI command | Native ACP? | Docker image | Auth |
|---|---|---|---|---|
| **Kiro** (default) | `kiro-cli acp` | yes | `ghcr.io/openabdev/openab` | `kiro-cli login --use-device-flow` |
| **Claude Code** | `claude-agent-acp` (adapter) | via `@agentclientprotocol/claude-agent-acp` | `ghcr.io/openabdev/openab-claude` | `claude login` (OAuth) or `ANTHROPIC_API_KEY` |
| **Codex** | `codex-acp` (adapter) | via `@zed-industries/codex-acp` | `ghcr.io/openabdev/openab-codex` | `codex login` or `OPENAI_API_KEY` |
| **Gemini** | `gemini --acp` | yes | `ghcr.io/openabdev/openab-gemini` | `gemini auth login` or `GEMINI_API_KEY` |
| **OpenCode** | `opencode acp` | yes | `ghcr.io/openabdev/openab-opencode` | `opencode auth login` |
| **Copilot CLI** ⚠️ | `copilot --acp --stdio` | yes | `ghcr.io/openabdev/openab-copilot` | `gh auth login -p https -w` |
| **Cursor** | `cursor-agent acp` | yes | `ghcr.io/openabdev/openab-cursor` | `cursor-agent login` |

Swap with one config line: `[agent].command = "..."`. See
[modules/acp](../modules/acp.md) for the ACP plumbing.

---

## Kiro CLI (default) / 預設

`../docs/kiro.md`

```toml
[agent]
command     = "kiro-cli"
args        = ["acp", "--trust-all-tools"]
working_dir = "/home/agent"
```

- Bundled in the **default Dockerfile** alongside `openab`.
- Authentication: `kiro-cli login --use-device-flow` (one-time inside the
  pod). Tokens persist to PVC under `~/.local/share/kiro-cli/`.
- Native ACP. **Caveat:** kiro-cli doesn't emit `configOptions`; it uses
  the older `models`/`modes` shape. OpenAB's `parse_config_options` handles
  this — see [modules/acp § Kiro fallback](../modules/acp.md#protocolrs-385-loc).

---

## Claude Code

`../docs/claude-code.md`

```toml
[agent]
command     = "claude-agent-acp"
args        = []
working_dir = "/home/node"
# env = { ANTHROPIC_API_KEY = "${ANTHROPIC_API_KEY}" }  # alternative to OAuth
```

- Uses the [`@agentclientprotocol/claude-agent-acp`](https://github.com/agentclientprotocol/claude-agent-acp)
  adapter on top of `@anthropic-ai/claude-code`.
- Two-phase tool emission (`tool_call` + `tool_call_update`) is the reason
  `classify_notification` reads `toolCallId` — without it the same tool
  would render twice.
- Auth: prefer `claude login` (OAuth) over `ANTHROPIC_API_KEY` env var. ANY
  env var in `[agent].env` is reachable to the agent and could be exfilled
  via prompt injection — `../AGENTS.md` flags this in the example config.

---

## Codex

`../docs/codex.md`

```toml
[agent]
command     = "codex"
args        = ["--acp"]
working_dir = "/home/node"
# env = { OPENAI_API_KEY = "${OPENAI_API_KEY}" }
```

- Uses the [`@zed-industries/codex-acp`](https://github.com/zed-industries/codex-acp)
  adapter on top of OpenAI's Codex CLI.
- Auth: prefer `codex login` over env var.
- Reference architecture for cron-based screening uses Codex —
  [deployment/reference-architectures § K8s CronJob](../deployment/reference-architectures.md).

---

## Gemini

`../docs/gemini.md`

```toml
[agent]
command     = "gemini"
args        = ["--acp"]
working_dir = "/home/node"
# env = { GEMINI_API_KEY = "${GEMINI_API_KEY}" }
```

Native ACP. Auth via `gemini auth login` or API key.

---

## OpenCode

`../docs/opencode.md`

```toml
[agent]
command     = "opencode"
args        = ["acp"]
working_dir = "/home/node"
```

- Native ACP.
- **Notable quirk:** OpenCode handles tool authorization internally and
  **never emits `session/request_permission`**. All tools run without
  confirmation — equivalent to `--trust-all-tools` on other backends.
- Run `opencode auth login` once before starting OpenAB.

---

## Copilot CLI ⚠️

`../docs/copilot.md`

```toml
[agent]
command     = "copilot"
args        = ["--acp", "--stdio"]
working_dir = "/home/node"
env         = {}
```

- Auth via GitHub OAuth: `gh auth login -p https -w` inside the pod (device
  flow). See `../docs/gh-auth-device-flow.md`.
- ⚠️ Marked experimental in `../README.md` — behaviour may change.

---

## Cursor

`../docs/cursor.md`

```toml
[agent]
command     = "cursor-agent"
args        = ["acp", "--model", "auto", "--workspace", "/home/agent"]
working_dir = "/home/agent"
env         = {}
```

- Auth via `cursor-agent login` (browser-based device flow).
- The `--workspace` flag pins the agent's working directory inside the
  container.

---

## Choosing / 怎麼挑

Practical guidance from operators:

- **Just want it to work?** → Kiro (default; no extra setup).
- **Need Anthropic models?** → Claude Code.
- **Need OpenAI models?** → Codex.
- **Use Cursor IDE day-to-day?** → Cursor (reuses your Cursor account).
- **Already on Copilot?** → Copilot CLI (uses your gh auth).
- **Need an OSS model / self-hosted?** → OpenCode.
- **Want Gemini?** → Gemini.

All seven get the **same** OpenAB primitives (reactions, edit-streaming,
threading, session pool). The differences live above the broker.

---

## Multi-agent / 多 agent

You can run **all seven** in the same Helm release. Each `agents.<name>`
key in `../charts/openab/values.yaml` creates its own Deployment + Secret +
ConfigMap + PVC.

Full guide: `../docs/multi-agent.md` and
[concepts/multi-agent](../concepts/multi-agent.md).

---

## Cross-references / 交叉引用

- The ACP plumbing they share: [modules/acp](../modules/acp.md).
- The protocol they speak: [architecture/acp-protocol](../architecture/acp-protocol.md).
- The Dockerfiles that bundle them: [deployment/docker-images](../deployment/docker-images.md).
- An upcoming primitive for agent-initiated platform calls:
  [adrs/adr-006-context-aware-token](../adrs/adr-006-context-aware-token.md).
