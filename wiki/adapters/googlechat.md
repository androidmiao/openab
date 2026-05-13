---
title: "Google Chat (via gateway)"
type: adapter
sources:
  - ../gateway/src/adapters/googlechat.rs
  - ../docs/google-chat.md
related: [gateway-service, gateway-client]
updated: 2026-05-12
---

`gateway/src/adapters/googlechat.rs` (1703 行) 在 gateway 端處理 Google Chat
Apps SDK。Webhook 進來 → 驗 Google 簽發的 JWT(Bearer token 或 OIDC ID token)
→ 解析 Chat event payload → 標準化。Thread 由 `thread.name`(`spaces/.../threads/...`)
唯一識別。

`googlechat.rs` (1703 LOC) handles Google Chat Apps. Webhook → validate
Google-issued JWT (Bearer token or OIDC ID token) → parse Chat event →
normalise. Threads are identified by `thread.name`
(`spaces/<id>/threads/<id>`).

---

## Authentication / 認證

Google Chat App URLs are configured in the Cloud Console. Inbound requests
carry an `Authorization: Bearer <id_token>` JWT issued by Google for the
configured service account. The adapter:

1. Fetches Google's public keys from `https://www.googleapis.com/oauth2/v3/certs`
   (cached with TTL).
2. Verifies the JWT signature using `jsonwebtoken` crate (RS256).
3. Asserts the audience claim matches the configured app URL.

Outbound calls use a service-account JSON to mint short-lived access tokens
for the Chat REST API.

---

## Threading / 串接

Google Chat has **first-class threads** in Spaces:

- `space.name = "spaces/<id>"`
- `thread.name = "spaces/<id>/threads/<id>"`
- `message.name = "spaces/<id>/messages/<id>"`

The adapter sets `channel.id = space.name`, `channel.thread_id = thread.name`,
so OpenAB's per-thread session pool works exactly as it does on Discord.

When the bot creates a new thread (auto-thread mode), it uses
`messageReplyOption = REPLY_MESSAGE_OR_FAIL` on the first send, with
`threadKey` set to a UUID.

---

## Replies / 回覆

| Operation | API |
|---|---|
| Send | `POST spaces/<id>/messages` |
| Edit | `PATCH spaces/<id>/messages/<msg_id>?updateMask=text` |
| Add reaction | (limited — Chat reactions are different from Discord/Slack model) |

Edit is supported, so edit-streaming works.

---

## @mentions / 提及

Chat events include `message.annotations` with type `USER_MENTION` that
point to the mentioned user (`user.name`). The adapter populates
`event.mentions` from these annotations so OpenAB's @mention gating works.

In a Space DM (1:1), every message is treated as implicit mention by the
upstream Chat platform — the adapter doesn't need to special-case this.

---

## Limits / 限制

| Limit | Value |
|---|---|
| Message length | 4096 chars |
| Attachments | Drive-link only — no direct uploads |
| Rate limit | per-app per-space (varies) |

---

## Configuration / 設定

```yaml
# gateway-side env / values
GOOGLE_CHAT_SERVICE_ACCOUNT: "<json>"   # SA key
GOOGLE_CHAT_AUDIENCE: "https://your-gateway/webhook/googlechat"

# OAB-side
[gateway]
url = "ws://openab-gateway:8080/ws"
platform = "googlechat"
allowed_channels = ["spaces/AAA..."]
```

---

## Cross-references / 交叉引用

- Setup walk-through: `../docs/google-chat.md`.
- Gateway architecture: [adapters/gateway-service](gateway-service.md).
