---
title: "OpenAB Overview / 專案概覽"
type: overview
sources:
  - ../README.md
  - ../DESIGN.md
  - ../AGENTS.md
  - ../Cargo.toml
related: [architecture/system-overview, architecture/design-philosophy, concepts/messaging-model]
updated: 2026-05-12
---

OpenAB (Open Agent Broker) 是一個用 **Rust 1.21k 行核心 + 5.8k 行 gateway** 寫
成的輕量、安全、雲原生 ACP harness,把 Discord / Slack / Telegram / LINE / Feishu /
Google Chat / MS Teams 等訊息平台,橋接到任何 [Agent Client Protocol](https://github.com/anthropics/agent-protocol)
相容的程式碼 CLI(Kiro、Claude Code、Codex、Gemini、OpenCode、Copilot、Cursor)。
核心交流方式是 **stdio JSON-RPC**——不是 HTTP、不是 server、不是框架。

OpenAB is a thin, secure, cloud-native ACP harness — written in Rust — that
bridges a growing list of messaging platforms to any [Agent Client Protocol](https://github.com/anthropics/agent-protocol)
-compatible coding CLI via **stdio JSON-RPC**. Not an HTTP service, not a
framework — a pipe.

## What OpenAB is / OpenAB 是什麼

```
┌──────────────┐  Gateway WS   ┌──────────────┐  ACP stdio    ┌──────────────┐
│   Discord    │◄─────────────►│              │──────────────►│  coding CLI  │
├──────────────┤  Socket Mode  │    openab    │◄── JSON-RPC ──│  (acp mode)  │
│   Slack      │◄─────────────►│    (Rust)    │               └──────────────┘
├──────────────┤               └──────┬───────┘
│   Telegram   │◄──webhook──┐         │ WebSocket (outbound)
│   LINE       │◄──webhook──┤         │
│ Feishu/Lark  │◄───WS──────┤   ┌─────▼──────────┐
│ Google Chat  │◄──webhook──┤   │ Custom Gateway │
│   MS Teams   │◄──webhook──┘   │  (standalone)  │
└──────────────┘                └────────────────┘
```

Native adapters (Discord, Slack) connect outbound from the OpenAB binary
directly. Webhook-based platforms talk to the standalone **Custom Gateway**
service, which OpenAB connects to over WebSocket — so the OpenAB pod never
needs an inbound port. Detailed wiring: [System Overview](architecture/system-overview.md).

原生 adapter(Discord、Slack)直接從 OpenAB binary 對外連線;以 webhook 為主的
平台(Telegram、LINE、Feishu、Google Chat、Teams)透過獨立的 **Custom Gateway**
服務,而 OpenAB 以 outbound WebSocket 連到 gateway——OpenAB pod 永遠不需要對外開
port。詳細接線:[系統總覽](architecture/system-overview.md)。

## What OpenAB is NOT / 不是什麼

引自 `../DESIGN.md`:

- **Not an agent framework** — no memory, no RAG, no tool registry, no prompt
  template engine. Use the agent's own capabilities.
- **Not an orchestration engine** — it doesn't pick which agent handles a
  message, or how agents collaborate.
- **Not a platform SDK** — it delegates to the agent's existing CLI rather
  than reimplementing protocols.
- **Not opinionated about models** — any ACP-compatible CLI works.

> "OpenAB is a pipe, not a container."
> 「OpenAB 是條 pipe,不是容器。」

## The Four Pillars / 四個支柱

Source of truth: `../DESIGN.md`. Full discussion: [Design Philosophy](architecture/design-philosophy.md).

| Pillar / 支柱 | One-liner |
|---|---|
| **Thin Bridge** / 薄橋接 | OpenAB transports messages; it doesn't transform them. |
| **Multi-Bot Ready** / 多 bot 共存 | Multiple agents can coexist in one channel from day one. |
| **AI-Native** / AI 原生 | Install/upgrade/configure best done by asking an agent to read `docs/`. |
| **Security by Design** / 安全設計 | `env_clear()`, pod isolation, outbound-only — non-negotiable. |

## What's in the code / 程式碼結構

| Path | Lines | What it is |
|---|--:|---|
| `../src/main.rs` | 580 | Multi-adapter startup, signal handling (SIGTERM-aware), graceful shutdown. |
| `../src/adapter.rs` | 1198 | `ChatAdapter` trait, `AdapterRouter`, `OutputDirectives`, `[[reply_to:…]]` parser. |
| `../src/discord.rs` | 2308 | Discord adapter via `serenity` (Gateway WS + cache + events). |
| `../src/slack.rs` | 1382 | Slack adapter via Socket Mode. |
| `../src/gateway.rs` | 801 | Outbound WebSocket client for the standalone gateway. |
| `../src/dispatch.rs` | 1509 | Turn-boundary message batching (ADR-005). |
| `../src/cron.rs` | 1272 | Config-driven scheduler + optional usercron hot-reload. |
| `../src/config.rs` | 925 | TOML config, `${VAR}` expansion, enums for the policy knobs. |
| `../src/acp/connection.rs` | 901 | Spawn the CLI, run JSON-RPC, env-clear allowlist, permission auto-reply. |
| `../src/acp/pool.rs` | 490 | Session pool: `thread_key → AcpConnection` map, eviction, suspended sessions. |
| `../src/acp/protocol.rs` | 385 | ACP types: JSON-RPC, `AcpEvent`, `ConfigOption`, Kiro `models/modes` fallback. |
| `../src/reactions.rs` | 276 | Emoji status FSM: 👀 → 🤔 → 🔥/👨‍💻/⚡ → 🆗 + mood. |
| `../src/media.rs` | 432 | Image resize/compress + STT attachment download. |
| `../src/stt.rs` | 354 | OpenAI-compatible Whisper API client (Groq, OpenAI, local). |
| `../src/markdown.rs` | 349 | Discord-flavoured markdown + table rendering modes. |
| `../src/format.rs` | 327 | 2000-char message splitting, thread name shortening. |
| `../src/bot_turns.rs` | 352 | Bot-turn counter per thread (resets on human msg). |
| `../src/remind.rs` | 399 | Reminder / "@later" subsystem. |
| `../src/setup/wizard.rs` | 667 | Interactive `openab setup` flow. |
| `../src/error_display.rs` | 219 | User-facing error formatting. |
| `../src/timestamp.rs` | 114 | Timestamp helpers. |
| `../gateway/src/main.rs` | 665 | Standalone gateway: webhooks in, WebSocket to OAB out. |
| `../gateway/src/schema.rs` | 112 | Unified `GatewayEvent` / `GatewayReply` schema. |
| `../gateway/src/adapters/feishu.rs` | 2945 | Feishu / Lark adapter (long-conn + webhook). |
| `../gateway/src/adapters/googlechat.rs` | 1703 | Google Chat adapter. |
| `../gateway/src/adapters/teams.rs` | 808 | MS Teams adapter. |
| `../gateway/src/adapters/telegram.rs` | 264 | Telegram adapter. |
| `../gateway/src/adapters/line.rs` | 268 | LINE adapter. |

Versions at the time of this wiki bootstrap: **OpenAB `0.8.3`**, **gateway `0.4.0`**.

## Quick links to next steps / 下一步

- 想理解全貌 → [System Overview / 系統總覽](architecture/system-overview.md)
- 想理解為什麼這樣設計 → [Design Philosophy / 設計哲學](architecture/design-philosophy.md)
- 想了解一則訊息怎麼從平台到 agent → [Message Flow / 訊息流](architecture/message-flow.md)
- 想部署 → [Helm chart](deployment/helm-chart.md) · [Docker images](deployment/docker-images.md)
- 想新增平台 → [Custom Gateway ADR](adrs/adr-002-custom-gateway.md)
- 想換 agent → [Agent backends](agents/agents.md)
