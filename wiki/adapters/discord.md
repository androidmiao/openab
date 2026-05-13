---
title: "Discord Adapter (native)"
type: adapter
sources:
  - ../src/discord.rs
  - ../docs/discord.md
related: [../modules/adapter, ../concepts/thread-detection, ../concepts/messaging-model]
updated: 2026-05-12
---

`src/discord.rs` (2308 行) 是 OpenAB 最大的 adapter,用 `serenity` 0.12 連接
Discord Gateway WebSocket。負責 message intent 處理、thread 偵測(`thread_metadata`
不是 `parent_id`)、`bot_owns` 判定、edit-streaming、reaction 控制、DM、role
mention、forum threads、Discord-特有的 2000 字元限制。

Discord adapter via `serenity` 0.12 (Gateway WebSocket + cache + REST). The
largest adapter in the codebase. Handles message-intent dispatch, thread
detection, `bot_owns`, edit-streaming, reactions, DMs, role mentions, forum
threads, and the platform's 2000-char message cap.

---

## Connection model / 連線模型

```
serenity::Client (Gateway WS, outbound)
  └─ ShardManager
     └─ Shard ───events───► DiscordAdapter (EventHandler impl)
                              └─ AdapterRouter dispatch
```

Outbound-only — no inbound port. Gateway WebSocket is the only ingress and
it's established by OpenAB. See [system-overview](../architecture/system-overview.md).

---

## Intents / 權限位元

Required intents (set in `serenity::GatewayIntents`):

| Intent | Why |
|---|---|
| `GUILD_MESSAGES` | receive channel/thread messages |
| `GUILD_MESSAGE_REACTIONS` | reaction add/remove tracking |
| `MESSAGE_CONTENT` | privileged — needed to read message text |
| `GUILD_MEMBERS` | resolve `@user` display names |
| `DIRECT_MESSAGES` | when `allow_dm = true` |

`MESSAGE_CONTENT` and `GUILD_MEMBERS` are **privileged intents** and must be
enabled in the Discord Developer Portal. The README quick-start lists this
as step 0.

---

## Thread detection / Thread 偵測

Following `../AGENTS.md` Critical Rule #2:

> A channel is a thread iff `thread_metadata` is present (NOT `parent_id`).
> `bot_owns` = thread's `owner_id` matches the bot's user ID.
> Use `ChannelType` enum, not structural heuristics.
> Forum posts are threads with `parent_id` pointing to a forum channel.

Implementation lives in `discord.rs`'s `is_thread` and `bot_owns_thread`
helpers. Full discussion: [concepts/thread-detection](../concepts/thread-detection.md).

---

## Edit-streaming / 邊串邊改

For each ACP `AgentMessageChunk`:

1. Append to a per-turn accumulator.
2. On a 1500 ms timer (configurable), edit the bot's reply message in place.
3. If accumulated text exceeds 2000 chars, finalise current message and
   start a new one (see [modules/markdown-format § splitting](../modules/markdown-format.md#1-message-splitting--訊息切段)).

Edit-streaming is what gives Discord the "live typing" feel without spamming
new messages.

---

## Reactions / 表情狀態機

`StatusReactionController` (from `../src/reactions.rs`) is held per inbound
user message. Discord's `add_reaction` / `remove_reaction` are rate-limited;
the FSM's debounce keeps the request rate well under serenity's built-in
ratelimiter ceiling.

If `[reactions].remove_after_reply = true`, the final 🆗 is removed after
`done_hold_ms`. The default is to keep it.

---

## DMs / 私訊

`allow_dm` defaults to `false`. When `true`:

- A DM channel is treated as **implicit @mention** (no @mention needed).
- No thread is created (DMs don't support threads).
- `allowed_users` still applies — DMs are not a backdoor.
- Bot turn tracking (`max_bot_turns`) still applies.

See [concepts/messaging-model § 0](../concepts/messaging-model.md).

---

## Role mentions / 角色提及

`allowed_role_ids` lists Discord role IDs that **trigger the bot** the same
way a direct @mention does. Useful for "@dev-bots" style group pings.

> Caveat: if multiple OpenAB bots share the same role, **all of them** will
> respond simultaneously. The role mechanism is symmetric — there's no
> "pick one" arbitration.

---

## Forum channels / 論壇頻道

Forum posts are special threads whose `parent_id` is a forum channel
(`ChannelType::Forum`). The bot can post in them like any other thread; the
`bot_owns` rule still applies (the bot owns a forum post iff it created it).

Auto-thread-creation does **not** apply to forum channels — Discord forces
all forum messages to live inside posts (threads) already.

---

## Discord-specific limits / Discord 特定限制

From `../AGENTS.md` § Critical Rules § Discord API Limits:

| Limit | Value | Handling |
|---|--:|---|
| Message content | 2000 chars | `format::split_for_limit` |
| Select-menu options | 25 | truncate with count, don't crash |
| Rate limits | serenity-managed | respect built-in ratelimiter |

---

## Configuration / 設定

The full `[discord]` block is summarised in
[config/config-reference § [discord]](../config/config-reference.md#discord-block).
Highlights:

```toml
[discord]
bot_token = "${DISCORD_BOT_TOKEN}"
allowed_channels = ["YOUR_CHANNEL_ID"]
allow_dm = false
allow_bot_messages = "off"           # off / mentions / all
allow_user_messages = "involved"     # involved / mentions
max_bot_turns = 100
message_processing_mode = "per-message"  # per-message / per-thread / per-lane
```

Setup walk-through: `../docs/discord.md`.
