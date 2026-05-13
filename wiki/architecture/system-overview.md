---
title: "System Overview / 系統總覽"
type: architecture
sources:
  - ../README.md
  - ../DESIGN.md
  - ../AGENTS.md
  - ../src/main.rs
  - ../docs/adr/multi-platform-adapters.md
related: [design-philosophy, message-flow, acp-protocol, ../adapters/gateway-service, ../concepts/session-pool]
updated: 2026-05-12
---

OpenAB 是一個三層架構:**平台層 → broker 層(`openab` binary)→ ACP agent 層**。
broker 內部又分成「對外的 adapter」、「中間的 dispatch/session pool」、「對下的
ACP 連線」。所有 inbound 流量都是**從 openab 主動拉**或**從 gateway 主動推**,
openab pod 不開任何 inbound port。

OpenAB has three layers: **platform ↔ broker (`openab` binary) ↔ ACP agent**.
Inside the broker, the adapter façade, the dispatch / session pool, and the
ACP connection are clearly separated. Every inbound message either arrives
because OpenAB initiated an outbound connection (Discord/Slack) or because the
standalone gateway pushes it over the WebSocket OpenAB opened.

## High-level / 全景

```
  Platforms                       openab process                     Agent CLI
  ─────────                       ──────────────                     ─────────
                                  ┌────────────────────┐
  Discord  ───WS (serenity)─────► │  DiscordAdapter    │ ─┐
                                  └────────────────────┘  │
  Slack    ───Socket Mode WS────► │  SlackAdapter      │ ─┤
                                  └────────────────────┘  │
  Telegram ─┐                     ┌────────────────────┐  │   ┌──────────────┐
  LINE     ─┤  webhook            │  GatewayAdapter    │ ─┼──►│ AdapterRouter│
  Feishu   ─┼─►┌─────────────┐ WS │  (src/gateway.rs)  │  │   │ + Dispatcher │
  Teams    ─┤  │   gateway   │ ──►│                    │ ─┤   └──────┬───────┘
  GChat    ─┘  │ (standalone)│    └────────────────────┘  │          │
              └─────────────┘                              │   ┌──────▼───────┐
                                                           │   │ SessionPool  │
                                                           │   │ (thread_key  │
                                                           │   │  → AcpConn)  │
                                                           │   └──────┬───────┘
                                                           │          │  stdio
                                                           │          │  JSON-RPC
                                                           │          ▼
                                                           │   ┌──────────────┐
                                                           └──►│ kiro-cli /   │
                                                               │ claude /codex│
                                                               │ /gemini/...  │
                                                               └──────────────┘
```

## Two-tier adapter model / 雙層 adapter 模型

`../docs/adr/multi-platform-adapters.md` 定義的 `ChatAdapter` trait 是平台無關
抽象,由 `AdapterRouter` 統一處理(allowlist、thread 偵測、bot 防迴圈、訊息分
段、reaction 控制、edit-streaming)。詳見 [modules/adapter](../modules/adapter.md)
與 [adrs/adr-001-multi-platform-adapters](../adrs/adr-001-multi-platform-adapters.md)。

| Tier | Examples | Inbound | Connection |
|---|---|---|---|
| **Native adapter** (in `openab` binary) | Discord, Slack | outbound-only (serenity Gateway / Socket Mode) | always-on WebSocket |
| **Gateway adapter** (in standalone `gateway` binary) | Telegram, LINE, Feishu, Teams, Google Chat | inbound HTTPS webhook or long-conn WS | OpenAB connects outbound to gateway via WS |

The preferred long-term direction (ADR-002) is for **every** platform to go
through the gateway, so the broker permanently sheds inbound concerns.

長期方向(ADR-002)是讓**所有**平台都走 gateway,讓 broker 永遠不需要面對
inbound TLS / webhook / 平台憑證。

## Inside one `openab` pod / 一個 pod 內部

```
┌─ Kubernetes Pod ──────────────────────────────────────┐
│  openab (PID 1)                                       │
│    └─ kiro-cli acp --trust-all-tools (child process)  │
│       ├─ stdin  ◄── JSON-RPC requests                 │
│       └─ stdout ──► JSON-RPC responses                │
│                                                       │
│  PVC (/data)                                          │
│    ├─ ~/.kiro/                  (settings, sessions)  │
│    └─ ~/.local/share/kiro-cli/  (OAuth tokens)        │
└───────────────────────────────────────────────────────┘
```

- `openab` is **PID 1** with a SIGTERM handler that drains the ACP pool before
  exiting (see `../src/main.rs:33-54` for the signal logic).
- The agent CLI is a **child process**; its stdin/stdout are the only channel.
- Authentication state (OAuth tokens, settings, sessions) lives on a **PVC**.

The Helm chart creates **one Deployment per agent** — see [Helm chart](../deployment/helm-chart.md)
and the `agents.<name>` map in `../charts/openab/values.yaml`. To run two
agents in the same channel you deploy two Deployments with two bot tokens.

## Session keying / Session key

Inside the pod, the `SessionPool` keys conversations on a **platform-prefixed
thread key**: `"discord:<channel_or_thread_id>"`, `"slack:<channel>:<thread_ts>"`,
`"gateway:<platform>:<channel>:<thread_id>"`. Each key maps to one `AcpConnection`
(one child CLI process). See [concepts/session-pool](../concepts/session-pool.md).

## Outbound-only invariant / 純對外連線不變式

The single most important architectural invariant — embedded in DESIGN.md,
the security model, and ADR-002 — is that the OpenAB pod **does not accept
inbound platform traffic**.

`../DESIGN.md` 與 ADR-002 的核心不變式:**OpenAB pod 不接受任何 inbound
平台流量**。

- Discord and Slack natively support outbound-only modes (Gateway / Socket Mode).
- Webhook platforms are externalized to the gateway, which is a different pod
  and may even live behind a separate ingress.
- Cron triggers, GitHub webhooks, and other event sources land on the gateway
  too — see [Custom Gateway ADR](../adrs/adr-002-custom-gateway.md).

Consequences: OpenAB pods don't need TLS, don't need a `Service` of type
`LoadBalancer`, and have **zero externally reachable attack surface** in
default deployments.
