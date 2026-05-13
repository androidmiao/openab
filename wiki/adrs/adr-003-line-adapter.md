---
title: "ADR-003 — LINE Messaging API Adapter"
type: adr
sources:
  - ../docs/adr/line-adapter.md
related: [adr-002-custom-gateway, ../adapters/line]
updated: 2026-05-12
---

> **Status**: Accepted.
> **Initial date**: 2026-04-22 · **Last updated**: 2026-04-28.
> **Authors**: @chaodu-agent, @iamninihuang.

OpenAB 加入 LINE 平台支援的設計。原本 v1 把 LINE adapter 放在 OAB binary 內,
之後 v2 (透過 [ADR-002](adr-002-custom-gateway.md)) 整支搬到 standalone gateway。
本 ADR 同時包含「**為什麼選 LINE**」「**簽章規則**」「**1:1 vs 群組的差異**」三個
維度,後兩個概念被 ADR-002 接收為通用合規規則。

The design for LINE platform support. v1 placed the adapter in the OAB
binary; v2 (per [ADR-002](adr-002-custom-gateway.md)) moved it to the
standalone gateway. This ADR also enshrines two general rules: signature
validation against **raw request bytes** and the **1:1 vs group** capability
trade-off.

---

## Decision / 決議

LINE Messaging API integration via webhook + HMAC-SHA256 signature
validation. Reply path prefers the **free Reply API** (single-use,
short-lived token) and falls back to the **Push API** (consumes monthly
quota) when the reply token has expired or been used.

In v1: integrated into OAB binary as `src/line.rs`.
In v2: moved entirely to `gateway/src/adapters/line.rs`.

---

## When to use LINE / 何時用 LINE

LINE shines when:
- Users are LINE-dominant (TW, JP, TH, ID).
- Use case is **1:1 private** with the agent (one user = one dedicated
  session, similar to Discord DM).
- Mobile-first.

LINE struggles when:
- Multiple users collaborate in **one group** (a LINE group shares **one**
  session, leading to context pollution).
- Long, multi-turn team threads — LINE has no threading.
- Concurrent usage > 10 users (always-on session model creates memory
  pressure).
- Rich interaction needs (no reactions, 5000-char cap, no threads).

| Scenario | Recommended |
|---|---|
| Individual developer, mobile-first | LINE |
| Small team async collab | Discord / Slack |
| Large team concurrent | Discord / Slack |
| Region with LINE dominance | LINE (1:1) + Discord (team) |
| Public community bot | Discord |

---

## Signature / 簽章

> Adapters must validate signatures against **exact raw request body
> bytes** — Compliance #1.

```
expected = HMAC-SHA256(channel_secret, raw_body_bytes)
                base64-encoded
actual   = header "X-Line-Signature"
constant-time compare → reject on mismatch
```

This rule was promoted to ADR-002 § Compliance #5 (applies to **all**
gateway adapters).

---

## Reply token / Reply token

LINE issues a `replyToken` per inbound event:
- Valid for ~60 seconds.
- Single-use.
- Using it costs **0** quota (Reply API is free).
- After expiry / single use → fall back to Push API, which consumes
  monthly quota.

The gateway maintains a TTL-50s cache so the first OAB consumer to
`remove()` a token wins the free path. See
[adapters/gateway-service § AppState](../adapters/gateway-service.md#appstate--應用狀態).

---

## Migration / 遷移路徑

The ADR documents an explicit v1 → v2 migration plan:

```
Phase 1: Deploy gateway alongside v1 OAB (LINE traffic still on v1)
Phase 2: Validate gateway on a separate LINE test channel
Phase 3: Cut over — point production LINE webhook URL to gateway
Phase 4: Remove line.rs from OAB
```

Because LINE allows only **one webhook URL per channel**, Phase 3 must be
atomic per channel — there's no traffic split.

The OAB codebase at the time of this wiki bootstrap has already completed
Phase 4: there is **no `src/line.rs`**. The gateway is the production path.

---

## Status / 狀態

Accepted; v2 architecture in production.

---

## Related / 相關

- Custom Gateway successor: [ADR-002](adr-002-custom-gateway.md).
- LINE adapter implementation: [adapters/line](../adapters/line.md).
- Gateway-side service: [adapters/gateway-service](../adapters/gateway-service.md).
