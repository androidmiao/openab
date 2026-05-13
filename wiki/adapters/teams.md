---
title: "MS Teams (via gateway)"
type: adapter
sources:
  - ../gateway/src/adapters/teams.rs
  - ../docs/msteams-enterprise.md
  - ../docs/msteams-selfhosted.md
related: [gateway-service, gateway-client]
updated: 2026-05-12
---

`gateway/src/adapters/teams.rs` (808 行)實作 Microsoft Bot Framework 對 Teams
的整合。Webhook 進來 → 驗 Microsoft 簽發的 JWT(Open ID metadata)→ 解析
Activity → 標準化。回覆要對 inbound activity 帶來的 `serviceUrl` 路由,該 URL
**因人因 channel 而異**,所以 gateway 維護一張 `conversation_id → service_url`
的快取。

`teams.rs` (808 LOC) integrates with Microsoft Bot Framework for Teams.
Webhook → validate Microsoft-issued JWT (via Open ID metadata) → parse the
Activity → normalise. Replies must route back to the **`serviceUrl`** that
came with the inbound activity — and that URL **varies per
tenant/channel/conversation**, so the gateway caches
`conversation_id → (service_url, last_seen)`.

---

## Authentication / 認證

Bot Framework's auth is JWT-based:

- Inbound: `Authorization: Bearer <jwt>` signed by Microsoft.
- The adapter fetches OpenID metadata from
  `https://login.botframework.com/v1/.well-known/openidconfiguration`
  (with cache + periodic refresh) and verifies signature, audience, and
  issuer.
- Outbound: the adapter mints a Bot Framework token by POSTing to
  `https://login.microsoftonline.com/botframework.com/oauth2/v2.0/token`
  with the bot's MicrosoftAppId / MicrosoftAppPassword.

---

## `service_url` routing / `service_url` 路由

Teams (and Bot Framework in general) embeds a `service_url` in every
inbound activity. **This URL is the only valid endpoint to reply to that
conversation.** It differs per tenant region (e.g.
`https://smba.trafficmanager.net/amer/`, `https://smba.trafficmanager.net/emea/`).

The gateway keeps:

```rust
teams_service_urls: Mutex<HashMap<conversation_id, (service_url, Instant)>>
```

On reply, the gateway looks up the conversation_id in this cache. If absent
(e.g. agent sends an unsolicited message after restart), the reply has to be
dropped or routed to a default URL with a warning.

ADR-002 § 3 calls out `reply_context` as a deferred schema concern; this is
exactly the kind of context that should eventually be part of the unified
`GatewayEvent` rather than a side-cache.

---

## Threading / 串接

Teams supports **channel posts with replies** (channel conversations) and
**1:1 / group chats** (no formal threading). The adapter maps:

| Teams concept | Unified schema |
|---|---|
| `channelData.teamsChannelId` | `channel.id` for channel conversations |
| Root activity ID of a channel post | `channel.thread_id` (thread = the replies under that root) |
| 1:1 / group conversation ID | `channel.id`, `thread_id = None` |

This produces the right session-key behaviour: one session per "channel
post + its replies" thread.

---

## Replies / 回覆

| Operation | Bot Framework API |
|---|---|
| Send | `POST <service_url>/v3/conversations/<id>/activities` |
| Reply to root | with `replyToId = <root_activity_id>` |
| Edit | `PUT <service_url>/v3/conversations/<id>/activities/<activity_id>` |

Edit-streaming works; the adapter keeps the activity ID returned by the
first send and updates it.

---

## Enterprise vs self-hosted / Enterprise 與自架

Two flavours of Teams integration (separate docs):

- **Enterprise** (`../docs/msteams-enterprise.md`) — uses Azure Bot
  resource, Bot Framework auth flow, standard Microsoft cloud.
- **Self-hosted** (`../docs/msteams-selfhosted.md`) — for Microsoft 365
  GCC / sovereign cloud deployments where the bot must register against a
  different audience.

The adapter code is the same; the difference is in the JWT issuer / audience
config and the OpenID metadata URL.

---

## Limits / 限制

| Limit | Value |
|---|---|
| Activity text length | ~28000 chars |
| File attachments | via `attachments[].contentUrl` |
| Rate limits | per Bot Framework documentation |

---

## Cross-references / 交叉引用

- Enterprise setup: `../docs/msteams-enterprise.md`.
- Self-hosted setup: `../docs/msteams-selfhosted.md`.
- Gateway architecture: [adapters/gateway-service](gateway-service.md).
