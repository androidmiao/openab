---
title: "Gateway Client (OAB side) — `src/gateway.rs`"
type: adapter
sources:
  - ../src/gateway.rs
related: [gateway-service, ../adrs/adr-002-custom-gateway, ../modules/adapter]
updated: 2026-05-12
---

`src/gateway.rs` (801 行)是 OpenAB **這一側**的 gateway 客戶端:一個 outbound
WebSocket client，把 standalone gateway 推來的 `GatewayEvent` 翻譯成
`ChatAdapter` 動作,並把 OpenAB 的回覆打包成 `GatewayReply` 推回 gateway。
所有 webhook 平台都共用這個 client。

`src/gateway.rs` (801 LOC) is the **OAB-side** WebSocket client that talks
to the standalone gateway service. It receives `GatewayEvent`s from the
gateway and translates them into `ChatAdapter` operations; it packages
OpenAB's replies into `GatewayReply`s and sends them back. **All
webhook-based platforms share this single client.**

---

## Why a single shared client / 為何只有一個 client

The standalone gateway (`gateway/` binary) normalises every supported
platform (Telegram, LINE, Feishu, Teams, Google Chat, …) into the **same**
`GatewayEvent` schema (see [gateway-service § schema](gateway-service.md)).
On the OpenAB side, that means **one adapter handles all of them**.

Platform-specific code in OpenAB stays at zero — adding a new gateway-side
platform requires no OpenAB code change at all (this is the ADR-002
guarantee).

---

## Connection / 連線

```toml
[gateway]
url           = "ws://openab-gateway:8080/ws"
platform      = "line"      # primary platform label for session keys
token         = "${GATEWAY_TOKEN}"
bot_username  = "my_bot"    # for @mention gating in groups
allowed_channels = []        # per gateway-platform allowlist
allowed_users    = []
```

The client connects outbound to `[gateway].url`, sending `?token=...` if
set. The gateway authenticates and starts streaming events.

Reconnect: on disconnect the client backs off and retries; events buffered
on the gateway-side broadcast channel may be lost during the gap (the
`event_tx` is a `tokio::sync::broadcast` channel — there's no replay).
ADR-002 § 7 flags this as an open design question.

---

## Inbound: GatewayEvent → ChatAdapter / 進來

```
gateway WS message
  ├─ deserialize GatewayEvent
  ├─ build platform-prefixed session key: "gateway:<platform>:<channel.id>:<thread_id or '_'>"
  ├─ apply allowlists (channel, user)
  ├─ build ContentBlock(s) from event.content + event.content.attachments
  └─ dispatch via the shared AdapterRouter
```

Attachments arrive as base64 already (the gateway downloads from the
platform's CDN and re-encodes). For images larger than the broker's cap, an
extra resize pass happens in `media.rs` ([modules/media-stt](../modules/media-stt.md)).

---

## Outbound: ChatAdapter → GatewayReply / 出去

The reverse direction. When the broker emits an outbound message, the
client wraps it in `GatewayReply`:

```json
{
  "schema": "openab.gateway.reply.v1",
  "reply_to": "evt_abc123",     // upstream event_id for reply-token lookup (LINE), etc.
  "platform": "line",
  "channel": { "id": "Cxyz789", "thread_id": null },
  "content": { "type": "text", "text": "..." }
}
```

The gateway's reply router looks up the platform-specific credentials and
calls the platform's API. See [gateway-service § reply router](gateway-service.md#reply-router--回覆路由).

---

## Edit-streaming through the gateway / 透過 gateway 邊串邊改

Edits flow through the same channel as new messages — the reply includes a
`message_id` for the gateway to issue an edit instead of a send.

Platform-specific quirks (LINE has no edit, Telegram has `editMessageText`,
Feishu has `update_message`) are absorbed by the gateway-side adapter.

---

## When the gateway is down / Gateway 斷線時

- Events from platforms during the outage are **lost** (the gateway either
  buffers in memory or drops, depending on its config).
- OpenAB's chat workflows on Discord / Slack are **unaffected** — they
  bypass the gateway entirely.
- The reaction FSM does not move when no events arrive — users see no
  spurious "still thinking" updates.

ADR-002 § 7 lists reconnect / at-least-once delivery as an open question
for v2 of the gateway protocol.

---

## Cross-references / 交叉引用

- The standalone gateway service: [gateway-service](gateway-service.md).
- The motivating ADR: [adrs/adr-002-custom-gateway](../adrs/adr-002-custom-gateway.md).
- The unified event/reply schema: `../gateway/src/schema.rs`.
