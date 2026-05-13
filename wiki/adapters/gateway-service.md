---
title: "Gateway Service (standalone binary)"
type: adapter
sources:
  - ../gateway/Cargo.toml
  - ../gateway/src/main.rs
  - ../gateway/src/schema.rs
  - ../gateway/src/adapters/mod.rs
  - ../docs/adr/custom-gateway.md
related: [gateway-client, telegram, line, feishu, googlechat, teams, ../adrs/adr-002-custom-gateway]
updated: 2026-05-12
---

`gateway/` 是獨立的 Rust binary(crate `openab-gateway` v0.4.0,~5.8k 行)。
它在前面收 webhook(`axum`),解析平台特有的 payload,**標準化**成
`GatewayEvent`,透過 WebSocket 推給 OpenAB;OpenAB 的 `GatewayReply` 反向過來
時則路由到對應平台的 API。本頁是 service 層的整體說明,個別平台 adapter 各自
有專頁。

`gateway/` is the standalone Rust binary (crate `openab-gateway` v0.4.0,
~5.8k LOC). It terminates webhooks with `axum`, parses each platform's
payload, **normalises** to `GatewayEvent`, and pushes events to OpenAB via
WebSocket. The reverse direction translates `GatewayReply`s back to
platform-specific API calls. Per-platform pages live below.

---

## Crate layout / Crate 結構

```
gateway/
├── Cargo.toml                 # crate name: openab-gateway, v0.4.0
├── Dockerfile
├── README.md
└── src/
    ├── main.rs                # axum server + WS handler + AppState
    ├── schema.rs              # GatewayEvent / GatewayReply / GatewayResponse types
    └── adapters/
        ├── mod.rs
        ├── telegram.rs       (264 LOC)
        ├── line.rs           (268 LOC)
        ├── teams.rs          (808 LOC)
        ├── googlechat.rs     (1703 LOC)
        └── feishu.rs         (2945 LOC)
```

`Cargo.toml` notable deps: `axum 0.8` (server), `tokio-tungstenite 0.21`
(WS), `jsonwebtoken 9` (Teams), `hmac` / `sha2` (signature checks),
`aes` / `cbc` (Feishu payload decryption), `prost 0.13` (Feishu binary
push messages).

---

## Schema / 標準化 schema

Defined in `../gateway/src/schema.rs`:

```rust
GatewayEvent {
    schema:     "openab.gateway.event.v1",
    event_id:   "evt_<uuid>",
    timestamp:  ISO-8601 UTC,
    platform:   "telegram"|"line"|"feishu"|"teams"|"googlechat",
    event_type: "message" | ...,
    channel:    ChannelInfo { id, type, thread_id: Option<String> },
    sender:     SenderInfo  { id, name, display_name, is_bot },
    content:    Content { type: "text", text, attachments: Vec<Attachment> },
    mentions:   Vec<String>,
    message_id: String,
}

Attachment {
    type: "image" | "text_file",
    filename, mime_type, data: base64, size,
}

GatewayReply {
    schema:     "openab.gateway.reply.v1",
    reply_to:   String,          // upstream event_id (for LINE reply-token lookup, etc.)
    platform:   String,
    channel:    ReplyChannel { id, thread_id: Option<String> },
    content:    Content,
    command:    Option<String>,  // e.g. "create_topic"
    request_id: Option<String>,
}

GatewayResponse {
    schema:     "openab.gateway.response.v1",
    request_id: String,
    success:    bool,
    thread_id, message_id, error: Optionals,
}
```

This contract is the **only** thing OpenAB and the gateway agree on. Every
platform-specific quirk is absorbed below this line.

---

## `AppState` / 應用狀態

From `gateway/src/main.rs`:

```rust
pub struct AppState {
    pub telegram_bot_token:   Option<String>,
    pub telegram_secret_token: Option<String>,
    pub line_channel_secret:  Option<String>,
    pub line_access_token:    Option<String>,
    pub teams:                Option<TeamsAdapter>,
    pub teams_service_urls:   Mutex<HashMap<String, (String, Instant)>>,
    pub feishu:               Option<FeishuAdapter>,
    pub google_chat:          Option<GoogleChatAdapter>,
    pub ws_token:             Option<String>,
    pub event_tx:             broadcast::Sender<String>,
    pub reply_token_cache:    ReplyTokenCache,
}
```

Two cross-cutting state items worth flagging:

- **`event_tx: broadcast::Sender<String>`** — every adapter publishes
  normalised events to this single broadcast channel. All connected
  `openab` clients subscribe and receive every event. Multi-tenancy routing
  is **not implemented** at v0.4.0 — every client sees every event. ADR-002
  flags this as an open question.
- **`reply_token_cache: ReplyTokenCache`** — LINE's reply token is
  single-use and expires after ~60s. The cache stores `event_id → (token,
  insertion_time)` with `REPLY_TOKEN_TTL_SECS = 50`. The first OAB client
  to `remove()` the token wins the free Reply API call; others fall back
  to the quota-consuming Push API. Bounded at `REPLY_TOKEN_CACHE_MAX = 10_000`.

---

## Routes / 路由

```
POST /webhook/telegram   → telegram adapter
POST /webhook/line       → line adapter
POST /webhook/feishu     → feishu adapter (also WS long-conn)
POST /webhook/teams      → teams adapter
POST /webhook/googlechat → googlechat adapter
GET  /ws                 → OAB WebSocket upgrade
GET  /health             → health check
```

Each platform's webhook route does signature/auth validation **first**, then
parses, then calls `GatewayEvent::new` (or builds the struct directly when
attachments are involved) and broadcasts.

---

## Reply Router / 回覆路由

`handle_oab_connection` in `gateway/src/main.rs`:

- Receives `GatewayReply` over the WS.
- Switches on `reply.platform` and dispatches to the corresponding adapter:
  - `telegram` → `https://api.telegram.org/bot<token>/sendMessage`
  - `line` → either Reply API (if `reply_token` is cached and unconsumed)
    or Push API.
  - `feishu` → call `FeishuAdapter::send`.
  - `teams` → use cached `service_url` + Bot Framework JWT.
  - `googlechat` → POST to Chat REST API.

For commands (e.g. `command: "create_topic"`), the gateway returns a
`GatewayResponse` so OpenAB can record the new thread ID.

---

## Per-platform pages / 各平台頁

| Adapter | LOC | Page |
|---|--:|---|
| Telegram | 264 | [adapters/telegram](telegram.md) |
| LINE | 268 | [adapters/line](line.md) |
| Feishu/Lark | 2945 | [adapters/feishu](feishu.md) |
| Google Chat | 1703 | [adapters/googlechat](googlechat.md) |
| MS Teams | 808 | [adapters/teams](teams.md) |

---

## Open design questions / 開放設計問題

From ADR-002 § 7:

| Question | Today's choice (v0.4.0) | Note |
|---|---|---|
| WebSocket direction | OAB connects to GW | Matches Discord / Slack pattern. |
| Multi-tenancy | One GW per OAB instance | Shared GW needs a routing table. |
| Reconnect / backpressure | `broadcast` channel, no replay | Events during outage are lost. |
| Event ordering | best-effort | strict ordering would need per-channel queues. |
| Plugin distribution | adapters are built-in | dynamic loading deferred to v3+. |
