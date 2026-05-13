---
title: "Slack Adapter (native)"
type: adapter
sources:
  - ../src/slack.rs
  - ../docs/slack-bot-howto.md
related: [../modules/adapter, ../concepts/thread-detection, ../concepts/messaging-model]
updated: 2026-05-12
---

`src/slack.rs` (1382 行) 用 Slack **Socket Mode** 跑 outbound WebSocket。所有
inbound 事件都從這個 WS 來,所以 OpenAB pod 不需要 public URL。它處理 `thread_ts`、
mpim / DM、`app_mention`、edit-streaming、reactions、以及 `multibot-mentions`
這個 Slack 特有的 multi-bot 行為。

Slack adapter using **Socket Mode** — a persistent outbound WebSocket. All
inbound events arrive through this WS, so the OpenAB pod needs no public URL.
The adapter handles `thread_ts`, MPDMs, DMs, `app_mention`, edit-streaming,
reactions, and Slack's `multibot-mentions` policy.

---

## Why Socket Mode / 為何用 Socket Mode

Slack apps can be configured for either HTTP webhook (Events API) or Socket
Mode. OpenAB picks **Socket Mode** because:

- Outbound-only — matches OpenAB's pod model (see [system-overview](../architecture/system-overview.md)).
- No TLS termination, no public Service.
- Simpler dev loop: works from a laptop without ngrok.

Tradeoff: Socket Mode requires both a **Bot Token** (`xoxb-…`) and an
**App-Level Token** (`xapp-…`) with `connections:write` scope.

---

## Tokens / Token 配對

```toml
[slack]
bot_token = "${SLACK_BOT_TOKEN}"   # Bot User OAuth Token (xoxb-...)
app_token = "${SLACK_APP_TOKEN}"   # App-Level Token (xapp-...) for Socket Mode
```

- `bot_token` = identity for sending messages, reactions, lookups.
- `app_token` = identity for the Socket Mode WS connection.

Both are required.

---

## Event types subscribed / 訂閱的事件

From `../docs/slack-bot-howto.md` step 3:

- `app_mention` — triggers when someone @mentions the bot.
- `message.channels` — receives messages in public channels (needed for
  thread follow-ups).
- Optional: `message.groups`, `message.im`, `message.mpim` for private
  channels / DMs / multi-party DMs.

---

## Threading / Slack thread

Slack threads are identified by the **`thread_ts`** field. A reply lives in
a thread iff `event.thread_ts != event.ts` (or, equivalently, `thread_ts`
is set and not equal to the message's own `ts`).

Session key construction: `"slack:<channel>:<thread_ts>"`. New top-level
messages use `"slack:<channel>:<ts>"` (the message's own `ts` becomes the
new thread root).

---

## Multibot mentions / 多 bot 提及

`AllowUsers::MultibotMentions` is **Slack-specific** semantics:

| State | Behaviour |
|---|---|
| Only one bot has posted in the thread | act like `involved` (no @mention needed for follow-ups) |
| ≥ 2 bots have posted in the thread | act like `mentions` (require @mention) |

Why it exists: in single-bot threads users get the "type freely" experience
they want. The moment a second bot joins, ambiguity rises (which bot is
being addressed?), so the policy upgrades to require @mention to disambiguate.

---

## Edit-streaming / 邊串邊改

Same pattern as Discord (see [adapters/discord § edit-streaming](discord.md#edit-streaming--邊串邊改)).
Slack's `chat.update` is the underlying API. The cap is Slack's mrkdwn limit
(~40000 chars) which is large enough that splitting rarely happens.

---

## Reactions / 表情

Slack reactions are added with `reactions.add` and removed with
`reactions.remove`. The shortcodes (`:eyes:`, `:thinking_face:`, `:fire:`,
`:white_check_mark:`, `:scream:`) are mapped from the emoji set in
`[reactions.emojis]` at startup.

The same debounce/stall FSM applies — see [modules/reactions](../modules/reactions.md).

---

## Configuration / 設定

```toml
[slack]
bot_token  = "${SLACK_BOT_TOKEN}"
app_token  = "${SLACK_APP_TOKEN}"
allowed_channels = ["C0123456789"]
allowed_users    = ["U0123456789"]
allow_bot_messages  = "off"
trusted_bot_ids     = []
allow_user_messages = "involved"   # involved / mentions / multibot-mentions
max_bot_turns       = 100
```

Setup walk-through: `../docs/slack-bot-howto.md`.
