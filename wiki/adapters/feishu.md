---
title: "Feishu / Lark (via gateway)"
type: adapter
sources:
  - ../gateway/src/adapters/feishu.rs
  - ../docs/feishu.md
related: [gateway-service, gateway-client]
updated: 2026-05-12
---

`gateway/src/adapters/feishu.rs` (2945 行)是 gateway 端最大的 adapter,因為
Feishu 同時支援兩種接收模式 — **long-connection WebSocket**(預設,不需要 public
URL,類似 Slack Socket Mode)與 **HTTP webhook**(fallback) — 加上 payload 可
能被 AES-CBC 加密、訊息以 `prost` decode 的二進制 push frame 走、需要刷新
tenant access token 等多平台特有的麻煩。

`feishu.rs` (2945 LOC) is the largest gateway adapter because Feishu / Lark
supports **two receive modes**: a long-connection WebSocket (default, no
public URL needed — like Slack Socket Mode) and HTTP webhook (fallback).
Payloads may be AES-CBC encrypted, binary push frames are decoded with
`prost`, and the tenant access token must be refreshed periodically.

---

## Two receive modes / 兩種接收模式

### A. Long-connection WebSocket (default)

```
Gateway (initiates)
   ▼ wss://open.feishu.cn/connection/<…>
Feishu open-API
   ▲
   └─ pushes events as prost-encoded binary frames
```

- No public URL needed.
- Authentication via `app_id` + `app_secret`.
- Resilient to NAT — the gateway opens the connection.

### B. HTTP webhook (fallback)

```
Feishu open-API ──POST──► Gateway (:8080/webhook/feishu)
                              │
                              ├─ optional AES-CBC decrypt with encrypt_key
                              ├─ JSON parse
                              └─ same normalisation
```

Used when the deployment must avoid outbound WS to Feishu (e.g. egress
firewall). Requires a publicly reachable URL.

---

## Authentication / 認證

App credentials:

| Field | Source |
|---|---|
| `app_id` | Feishu open-platform console |
| `app_secret` | Feishu open-platform console |
| `encrypt_key` (optional) | console — enables payload encryption |
| `verification_token` (optional) | console — extra request validation |
| Domain | `feishu` (中國大陸) 或 `lark` (international) |

The adapter exchanges `app_id` + `app_secret` for a **tenant_access_token**
(short-lived). The token is refreshed before expiry; all outbound API calls
include it as `Authorization: Bearer <token>`.

---

## Required scopes / 所需權限

From `../docs/feishu.md`:

- `im:message` — receive messages
- `im:message:send_as_bot` — send messages as bot
- `contact:user.base:readonly` — resolve sender display names (recommended;
  without it senders show as `ou_xxx`)
- Event subscription: `im.message.receive_v1` (long-connection or webhook)

---

## Payload normalisation / 標準化

Feishu's `im.message.receive_v1` payload contains:

```
event.message.{message_id, chat_id, chat_type, content (JSON-stringified)}
event.sender.{sender_id.open_id, sender_type}
event.message.mentions[]   // list of @mentioned open_ids
```

The adapter:

1. Decodes the message content JSON. Text messages are easy; rich-text /
   post / image / file types need further parsing.
2. Resolves the sender's display name via `contact:user.base:readonly`
   (cached to avoid hammering the API).
3. Builds attachments (image / file) by re-downloading from Feishu's
   asset CDN with the tenant token.
4. Populates `mentions` from the `event.message.mentions` array.

---

## Threading / 串接

Feishu has **topics (parent_id-style threads)** on bot-supported chats:

- `chat_type = "group"` or `"p2p"`.
- `parent_id` (when set on the message) identifies the parent topic.

The adapter maps `parent_id` → `channel.thread_id` so OpenAB's session
keying gets per-topic isolation.

---

## Replies / 回覆

| Operation | API |
|---|---|
| Send | `POST /open-apis/im/v1/messages` |
| Edit | `PATCH /open-apis/im/v1/messages/<msg_id>` |
| Create topic | `POST /open-apis/im/v1/messages` with `parent_id` |

The reply path uses the tenant access token; rate limits are per-app and
per-tenant.

---

## Encryption / 加密

When `encrypt_key` is set in the console, webhook payloads come as:

```json
{ "encrypt": "<base64 ciphertext>" }
```

The adapter:

1. Computes `aes_key = SHA256(encrypt_key)`.
2. AES-CBC-decrypts the ciphertext (the first 16 bytes are the IV).
3. Parses the decrypted JSON.

For the long-connection mode this layer is unnecessary — the WS frames are
already authenticated by the open-API handshake.

---

## Limits / 限制

| Limit | Value |
|---|---|
| Message length | ~30000 chars (depends on type) |
| API rate (per tenant) | varies by endpoint |
| Token TTL | ~2 hours (refreshed before expiry) |

---

## Cross-references / 交叉引用

- Setup walk-through: `../docs/feishu.md`.
- Gateway architecture overall: [adapters/gateway-service](gateway-service.md).
