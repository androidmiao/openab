---
title: "Telegram (via gateway)"
type: adapter
sources:
  - ../gateway/src/adapters/telegram.rs
  - ../docs/telegram.md
related: [gateway-service, gateway-client, ../adrs/adr-002-custom-gateway]
updated: 2026-05-12
---

`gateway/src/adapters/telegram.rs` (264 行)是 Telegram 平台的 gateway 端 adapter。
Webhook 進來 → 驗 `X-Telegram-Bot-Api-Secret-Token` → 解析 `Update` JSON →
標準化成 `GatewayEvent`。回覆方向用 `sendMessage` / `editMessageText` /
`setMessageReaction`。整體結構與 LINE 很像(都是 webhook + 推回 API),但
**Telegram 有 reactions**,所以 reaction 路徑有實作。

`gateway/src/adapters/telegram.rs` (264 LOC) is the Telegram gateway-side
adapter. Webhook → validate secret token → parse `Update` JSON → normalise
to `GatewayEvent`. Replies use `sendMessage` / `editMessageText`
(Telegram supports edits) / `setMessageReaction`.

---

## Webhook setup / 設定流程

From `../docs/telegram.md`:

1. Create the bot via [@BotFather](https://t.me/BotFather), capture the bot
   token.
2. Optional: `/setprivacy` → Disable, so the bot can see all group messages
   (required for @mention gating in groups).
3. Tell Telegram where the webhook is:
   ```
   curl "https://api.telegram.org/bot<TOKEN>/setWebhook?url=https://<GATEWAY>/webhook/telegram&secret_token=<SECRET>"
   ```
4. Configure `TELEGRAM_BOT_TOKEN` + `TELEGRAM_SECRET_TOKEN` on the gateway.

---

## Signature / 簽章

Telegram doesn't HMAC-sign requests; it relies on a **shared secret token**
sent in the `X-Telegram-Bot-Api-Secret-Token` header. The gateway compares
this against `state.telegram_secret_token` using a constant-time compare
(`subtle` crate).

---

## Threading / 串接

Telegram has **no native threads**. Reply chains via `reply_to_message_id`
exist but don't isolate sessions. The gateway therefore sets
`channel.thread_id = None` and `channel.type = "group"` / `"private"`.

Session keys land as `gateway:telegram:<chat_id>:_` — one session per chat.

For large groups this is a limitation (everyone shares one session); for
1:1 DMs and small groups it's fine.

---

## Replies / 回覆

| Operation | Telegram API |
|---|---|
| Send new message | `sendMessage` |
| Edit message | `editMessageText` |
| Set/clear reaction | `setMessageReaction` |
| Send photo | `sendPhoto` |

The gateway maintains per-message reaction state in
`reaction_state: HashMap<String, Vec<String>>` because Telegram replaces all
reactions atomically — you have to know the current set to compute the new
one.

---

## Limits / 限制

| Limit | Value |
|---|---|
| Message length | 4096 chars |
| Photo caption | 1024 chars |
| Rate limit (broadcast) | 30 msgs/sec across all chats |
| Rate limit (one chat) | 1 msg/sec |

`format::split_for_limit` honours the 4096 cap.

---

## Privacy and @mention / 隱私與 @mention

Telegram bots in groups see only messages that **directly mention them** or
are commands (`/foo@botname`), unless privacy mode is disabled. To make
group `@mention` gating work, set privacy mode to Disabled in BotFather.

The gateway parses entities of type `mention` / `text_mention` and populates
`event.mentions` so OpenAB's gate ⑦ can do its job.

---

## Cross-references / 交叉引用

- Setup walk-through: `../docs/telegram.md`.
- Gateway-side architecture: [adapters/gateway-service](gateway-service.md).
- OAB-side handling of the unified event: [adapters/gateway-client](gateway-client.md).
