---
title: "ADR-002 — Custom Gateway for Webhook Platforms"
type: adr
sources:
  - ../docs/adr/custom-gateway.md
related: [../adapters/gateway-client, ../adapters/gateway-service, adr-003-line-adapter, ../architecture/security-model]
updated: 2026-05-12
---

> **Status**: Proposed.
> **Date**: 2026-04-22 · **Author**: @chaodu-agent
> **Supersedes**: sections of [ADR-003 LINE Adapter](adr-003-line-adapter.md)
> (v2 Target Architecture).

把所有 webhook 平台(Telegram、LINE、Teams、Feishu、Google Chat、GitHub、CI/CD、
監控)從 OAB binary **挪出去**,放進獨立的 **Custom Gateway** 服務,並用一個
標準化的 `openab.gateway.event.v1` schema 跟 OAB 之間通訊。最大成果:OAB pod
**永遠不開 inbound port**,新增平台 = 寫 gateway adapter,**OAB 0 改動**。

Moves every webhook-based platform out of the OpenAB binary into a
standalone **Custom Gateway** service. OAB and the gateway communicate over
a unified `openab.gateway.event.v1` schema. Headline outcome: OAB pods stay
**permanently outbound-only**, and adding a new webhook platform becomes a
gateway-adapter implementation with **zero OAB changes**.

---

## Architecture / 架構

```
                    External (inbound HTTPS)              Internal (cluster)
LINE Platform ──POST──▶ ┌─────────────────────┐
Telegram      ──POST──▶ │                     │
GitHub Events ──POST──▶ │   Custom Gateway    │ ◀──WebSocket── OAB Pod
CI/CD webhook ──POST──▶ │     :443/8080       │     (OAB initiates)
Custom app    ──POST──▶ │                     │
                         └─────────────────────┘

Discord Gateway ◀──WebSocket───── OAB Pod  (unchanged, native)
Slack Socket    ◀──WebSocket───── OAB Pod  (unchanged, native)
```

Inside the gateway:

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│   LINE   │  │ Telegram │  │  GitHub  │  ...   (one adapter per platform)
└────┬─────┘  └────┬─────┘  └────┬─────┘
     ▼              ▼              ▼
  ┌──────────────────────────────────┐
  │     Event Normalizer             │   platform payload → unified event
  └──────────────────┬───────────────┘
                     ▼
  ┌──────────────────────────────────┐
  │     WebSocket Server (:9090)     │   OAB connects here as a client
  └──────────────────┬───────────────┘
                     ▼
  ┌──────────────────────────────────┐
  │     Reply Router                 │   OAB reply → correct platform API
  └──────────────────────────────────┘
```

---

## Why / 為什麼

| Aspect | v1 (LINE in OAB) | Custom Gateway |
|---|---|---|
| OAB inbound ports | Yes (webhook :8080) | None — pure outbound |
| Platform code in OAB | `line.rs`, config patches | Zero |
| New platform = | edit OAB | gateway plugin |
| TLS / K8s Service for OAB | required | not required |
| Deploy for DC/Slack-only | carries unused LINE code | gateway not deployed at all |
| Credential location | OAB | Gateway |

---

## Event schema / 事件 schema

The contract. Detailed in [adapters/gateway-service § schema](../adapters/gateway-service.md#schema--標準化-schema)
and source `../gateway/src/schema.rs`.

```
GatewayEvent (gateway → OAB):
  schema: "openab.gateway.event.v1"
  event_id, timestamp, platform, event_type
  channel { id, type, thread_id? }
  sender  { id, name, display_name, is_bot }
  content { type, text, attachments[] }
  mentions[], message_id

GatewayReply (OAB → gateway):
  schema: "openab.gateway.reply.v1"
  reply_to (event_id) — for LINE reply-token lookup, etc.
  platform, channel, content, command?, request_id?
```

---

## Beyond chat / 不只是聊天

The gateway turns OAB from a "chat bot" into an **event-driven agent
platform**:

| Event source | Webhook | OAB can do |
|---|---|---|
| LINE / Telegram / Feishu | user message | AI coding assistant (existing) |
| GitHub webhook | PR opened, issue created | auto-triage, auto-review, auto-label |
| CI/CD (Actions, Jenkins) | build failed | analyze logs, suggest fix |
| Monitoring (PagerDuty, CloudWatch) | alert fired | runbook execution |
| Custom app | any JSON payload | user-defined workflows |

A `curl -X POST .../webhook/custom -d '{"text":"run daily scan"}'` is enough
to start an agent session — no bot, no platform account needed.

---

## Open design questions / 開放問題

| Question | v0.4.0 choice | Note |
|---|---|---|
| WebSocket direction | OAB connects to gateway | matches Discord/Slack pattern |
| Multi-tenancy | one gateway per OAB instance | shared gateway needs routing table |
| Reconnect / backpressure | broadcast channel, no replay | events during outage lost |
| Event ordering | best-effort | strict ordering needs per-channel queues |
| Plugin distribution | built-in adapters | dynamic loading deferred to v3+ |

Schema fields known to be needed but not yet finalised: `conversation_key`/
`session_hint`, `trace_id`/`correlation_id`, `capabilities`/
`delivery_constraints`, `reply_context` (for Teams `service_url`),
`tenant`/`gateway_instance`.

---

## Compliance / 合規

1. OAB stays outbound-only.
2. Schema versioned (`v1`); breaking changes require version bump + migration.
3. Platform credentials in gateway, not in OAB.
4. Every adapter implements `validate`, `parse`, `send`, `health`.
5. Signature validation **must** be against raw request body bytes (see
   ADR-003 Compliance #1).

---

## Status / 狀態

Proposed. The gateway binary already exists at `gateway/` (`openab-gateway`
v0.4.0) with Telegram, LINE, Feishu, Teams, Google Chat adapters
implemented. Migration of LINE out of OAB core (Phase 4 of the rollout
plan) **has not yet landed** in the OAB binary at the time of this wiki —
`src/` does not contain a `line.rs`, so the v1 path is already retired in
the broker; the v2 gateway is the production path.

> Note: ADR text marks status as Proposed; the codebase has effectively
> moved to the gateway path. Worth re-confirming the ADR status in the
> next lint pass.

---

## Related / 相關

- LINE v1 adapter (superseded for v2): [ADR-003](adr-003-line-adapter.md).
- The gateway-side architecture: [adapters/gateway-service](../adapters/gateway-service.md).
- The OAB-side client: [adapters/gateway-client](../adapters/gateway-client.md).
