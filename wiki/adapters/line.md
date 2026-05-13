---
title: "LINE (via gateway)"
type: adapter
sources:
  - ../gateway/src/adapters/line.rs
  - ../docs/line.md
  - ../docs/adr/line-adapter.md
related: [gateway-service, gateway-client, ../adrs/adr-002-custom-gateway, ../adrs/adr-003-line-adapter]
updated: 2026-05-12
---

`gateway/src/adapters/line.rs` (268 行)是 LINE 在 gateway 端的 adapter。HTTPS
webhook 進來 → HMAC-SHA256 對 raw body 驗章 → 解析 LINE event JSON → 標準化。
LINE 最特別的地方是 **reply token**:每筆事件附帶一個一次性、約 60 秒過期的
reply token,用它呼叫 Reply API 是**免費**的;一旦過期或用掉,只能用 Push API,
而 Push 有月配額。

`gateway/src/adapters/line.rs` (268 LOC) is the LINE adapter. HMAC-SHA256
verifies the raw request body, the event JSON parses, and the result is
normalised. LINE's quirk is the **single-use reply token** — each inbound
event ships with a token that's free to use for ~60 seconds; after that, the
gateway falls back to Push API (which consumes the monthly quota).

---

## Signature / 簽章

```
HMAC-SHA256(channel_secret, raw_request_body)  →  base64
       compared (constant-time) to header X-Line-Signature
```

The body **must be the raw bytes** — not re-serialised, not whitespace-
trimmed. ADR-003 / ADR-002 Compliance #5 calls this out explicitly:

> All adapters must validate signatures against exact raw request body
> bytes.

`subtle::ConstantTimeEq` is used to avoid timing attacks.

---

## Reply token cache / Reply token 快取

`AppState.reply_token_cache: Arc<Mutex<HashMap<event_id, (token, insertion_time)>>>`
(see [adapters/gateway-service § AppState](gateway-service.md#appstate--應用狀態)).

```
inbound LINE event
  ├─ extract reply_token, event_id
  ├─ insert into cache with TTL = 50s (REPLY_TOKEN_TTL_SECS)
  └─ ...continue normalising

outbound LINE reply
  ├─ look up event_id in cache via reply.reply_to
  │     hit + unconsumed → Reply API (free)
  │     miss or stale    → Push API (quota)
  ├─ remove() the token on first use — first OAB client wins
```

Cache is hard-capped at `REPLY_TOKEN_CACHE_MAX = 10_000` entries with
periodic eviction of stale entries.

---

## Threading / 串接

LINE has **no threads, no edits, no reactions** in groups (1:1 DMs are
fine). Implications:

- `channel.thread_id` is always `None`.
- Edit-streaming degrades to "send a new message" mode.
- The reaction FSM's `add_reaction` / `remove_reaction` calls are dropped.

This is why [ADR-003](../adrs/adr-003-line-adapter.md) recommends LINE
primarily for **1:1 DMs** and channels Discord/Slack for team/group use.

---

## When to use LINE / 何時用 LINE

From ADR-003 § Best Fit by Scenario:

| Scenario | Recommended | Why |
|---|---|---|
| Mobile-first individual user | **LINE** | 1:1 DM works well; mobile-dominant |
| Small team async collab | **Discord / Slack** | Threads organise conversations |
| Large team concurrent | **Discord / Slack** | On-demand sessions scale better |
| LINE-dominant region (TW, JP, TH, ID) | **LINE** (1:1) + **Discord** (team) | mix-and-match |

---

## Limits / 限制

| Limit | Value |
|---|---|
| Message length | 5000 chars |
| Reply token TTL | ~60s (gateway cache holds 50s for safety) |
| Push API monthly quota | depends on plan (Free / Light / Standard) |

---

## Configuration / 設定

```yaml
# gateway-side env
LINE_CHANNEL_SECRET: "..."        # for signature
LINE_CHANNEL_ACCESS_TOKEN: "..."  # for Reply / Push calls

# OAB-side config.toml
[gateway]
url = "ws://openab-gateway:8080/ws"
platform = "line"
allowed_users = ["U..."]
```

---

## Cross-references / 交叉引用

- ADR (with v1 in-OAB → v2 gateway migration plan): [adrs/adr-003-line-adapter](../adrs/adr-003-line-adapter.md).
- ADR-002 supersedes v2 target architecture: [adrs/adr-002-custom-gateway](../adrs/adr-002-custom-gateway.md).
- Setup walk-through: `../docs/line.md`.
