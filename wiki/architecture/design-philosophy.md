---
title: "Design Philosophy / 設計哲學 — Four Pillars"
type: architecture
sources:
  - ../DESIGN.md
  - ../README.md
related: [system-overview, security-model, ../concepts/multi-agent]
updated: 2026-05-12
---

OpenAB 的設計建立在四個刻意對立於常見「agent 平台」做法的支柱上:**薄橋接**
(Thin Bridge)、**多 bot 友善**(Multi-Bot Ready)、**AI 原生操作**(AI-Native)、
**設計即安全**(Security by Design)。每個支柱都決定了「不做什麼」,而那些「不
做」本身就是設計。

OpenAB rests on four pillars that explicitly **opt out** of common
"agent platform" patterns: **Thin Bridge**, **Multi-Bot Ready**,
**AI-Native**, **Security by Design**. Each pillar defines what OpenAB
intentionally does *not* do; the omissions are the design.

---

## 1. Thin Bridge / 薄橋接

```
Platform ──► OpenAB ──► Agent CLI (via ACP stdio)
Platform ◄── OpenAB ◄── Agent CLI
```

OpenAB 是**運輸層**。它搬訊息,不轉譯。具體不做的事:

OpenAB is a **transport layer**. It moves messages. What it deliberately
doesn't do:

- ❌ Manage agent memory — agents own their own memory.
- ❌ Orchestrate multi-agent workflows — that's the user's design space.
- ❌ Inject system prompts or modify the context window.
- ❌ Govern agent behaviour or filter outputs (beyond markdown rendering and
  the `[[reply_to:…]]` directive).

對照 OpenClaw / Hermes Agent 把 memory、orchestration、governance 都收進平台
本身,OpenAB 的立場是「橋越薄,上面越自由」。

In contrast to OpenClaw or Hermes Agent — which bundle memory, orchestration,
and governance into the platform — OpenAB's bet is: **the thinner the bridge,
the more freedom above it.**

## 2. Multi-Bot Ready / 多 bot 共存

OpenAB was designed for multiple agents to share a channel from day one.
Every agent runs as an **independent pod** with its own bot token, its own
config, and its own session pool. OpenAB only provides **primitives** for
multi-agent coexistence — it never picks the orchestration strategy:

OpenAB 從第一天就為「同一個 channel 多個 agent」設計。每個 agent 都是**獨立的
pod**,自己的 bot token、自己的 config、自己的 session pool。OpenAB 只提供
共存的 primitive,**從不替使用者決定 orchestration**:

| Primitive | Purpose |
|---|---|
| `allow_bot_messages` (`off` / `mentions` / `all`) | gate bot-originated messages |
| `trusted_bot_ids` | further restrict which bots are accepted |
| `@mention` gating + role gating | who triggers a turn |
| `max_bot_turns` | hard cap on consecutive bot turns (default 100) |

可組合出的 pattern:sequential handoff、parallel collaboration、human-in-the-loop、
agent-to-agent discussion。詳見 [concepts/multi-agent](../concepts/multi-agent.md)。

## 3. AI-Native / AI 原生操作

OpenAB **assumes** that operators run an AI agent on the side. Documentation
under `docs/` is written **for AI consumption first** — each agent has a
standalone guide (`kiro.md`, `claude-code.md`, `codex.md`, …); the config
reference is structured for machine parsing.

OpenAB **假設**操作者旁邊有 AI 代理。`docs/` 是寫給 AI 看的:每個 agent 一份
獨立 guide,config reference 為機器解析設計。

```
User: "Set up Claude Code as a second agent"
  → Agent reads docs/claude-code.md and docs/multi-agent.md
  → Agent generates the Helm command
  → User reviews and applies
```

Not a documentation shortcut — a design choice. AI agents are better at
synthesizing multi-file instructions and resolving environment-specific
variables than static step-by-step guides.

詳見 [agent-installable-tools](../../docs/agent-installable-tools.md) 與
[ai-install-upgrade](../../docs/ai-install-upgrade.md)。

## 4. Security by Design / 安全即設計

OpenAB 設計為「**只能**跑在 sandbox 裡」。直接在 host(Linux/macOS/Windows)
上跑只用於 local dev / testing,生產環境**必須**進 pod。具體機制:

OpenAB is designed to run **only** inside Kubernetes pods. Running on a bare
host is supported strictly for local dev/testing. In production:

| Mechanism | Detail |
|---|---|
| Process isolation | one Deployment per agent, one CLI subprocess per thread |
| Read-only root FS | `containerSecurityContext.readOnlyRootFilesystem: true` (default in `values.yaml`) |
| `env_clear()` | agent subprocesses start with a clean env; only `HOME`, `PATH`, `USER` (unix) or `USERPROFILE`/`USERNAME`/`SystemRoot`/`SystemDrive` (Windows) pass through |
| Credential separation | each agent pod gets only its own bot token via K8s Secret |
| No host access | no hostNetwork, no host volume mounts in the default chart |
| Drop ALL capabilities | `containerSecurityContext.capabilities.drop: [ALL]` |

對照 OpenClaw 原先預設給 agent 完整 host 存取、之後才加 sandbox,OpenAB 是
**反向設計**:sandbox 是唯一的選項,沒有 opt-in 跳脫。

Contrast with OpenClaw, which started with full host access and added a
sandbox later as an opt-in hardening measure. OpenAB **reverses** that: the
sandbox is the only option; there is no opt-in escape hatch.

Full mechanics in [Security Model](security-model.md).

---

## Why this matters / 為什麼這樣設計

The agent ecosystem is moving fast. Memory systems, orchestration patterns,
governance frameworks — all in flux. By **staying thin**, OpenAB avoids
coupling to any particular approach. A user can give each of five agents
(Kiro, Claude, Codex, Gemini, Copilot) a completely different steering doc,
memory backend, and tool config; OpenAB does not know or care. It just moves
messages.

> OpenAB is a pipe, not a container. It transports messages without
> transforming them. The intelligence lives in the agents. The orchestration
> lives with the user. The bridge just works. — `../DESIGN.md`

> OpenAB 是條 pipe,不是容器。智慧在 agent,orchestration 在使用者,橋就只是
> 把訊息送過去。
