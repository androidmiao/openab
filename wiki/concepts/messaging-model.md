---
title: "Messaging Model — Five Patterns"
type: concept
sources:
  - ../docs/messaging.md
related: [../architecture/message-flow, multi-agent, thread-detection, ../adapters/discord, ../adapters/slack]
updated: 2026-05-12
---

`../docs/messaging.md` 把 OpenAB 的對話模式整理成五個漸進的 pattern,每一個都
是前一個的擴展:**0. DM**、**1. Channel @mention 開頭**、**2. Thread 接續(免
@mention)**、**3. Thread 多 bot**、**4. Bot 對 Bot**。

OpenAB's conversational surface is five patterns, each building on the
last: **0. DM**, **1. Channel-via-@mention**, **2. Thread follow-up**,
**3. Multi-bot thread**, **4. Bot-to-bot in thread**.

---

## 0. Human → Bot in DM / 私訊

**Off by default.** Set `allow_dm = true` to opt in (Discord only — Slack
DMs work natively without extra flags).

A DM is treated as an **implicit @mention**:

```
User DMs BotA: help me with X
  → BotA replies in DM (no thread, no @mention needed)
```

Constraints that still apply:
- `allowed_users` — DMs are not a backdoor.
- `max_bot_turns` — loop guard.
- Session pool — each DM user consumes one session slot
  (`discord:<dm_channel_id>`).

---

## 1. Human → Bot in Channel / 頻道內 @mention 起頭

The starting point of every conversation in a public channel:

```
#general
> User: @BotA explain VPC peering
> BotA creates thread "explain VPC peering"
> BotA's reply lands inside the new thread
```

Gates:
- `allowed_channels` must include this channel.
- `allowed_users` (or DM-style allowlist) must permit the sender.
- The message must @mention the bot (gate ⑦).

Auto-thread-creation happens because of `allow_user_messages: "involved"`
(default) — the bot creates a thread it owns so it can keep replying
without @mention.

---

## 2. Human → Bot in Thread / Thread 內接續

Once inside a bot-owned thread:

```
# thread: "explain VPC peering" (owned by BotA)
> User: ok but what about transit gateway?
> BotA replies (no @mention required)
```

`allow_user_messages` controls this:

| Value | Behaviour |
|---|---|
| `involved` (default) | reply in threads bot owns or participated in |
| `mentions` | always require @mention |
| `multibot-mentions` (Slack) | `involved` until a second bot posts; then `mentions` |

`bot_owns` is the truth check — see [concepts/thread-detection](thread-detection.md).

---

## 3. Human → Multiple Bots in Thread / 多 bot 同 thread

```
# thread (owned by BotA)
> BotA: …
> BotB joins (was @mentioned earlier in the channel)
> User: ok BotB now's your turn ← who replies?
```

Two strategies:

- **Always @-pin the target bot.** Set both bots to `allow_user_messages =
  "mentions"`. Predictable, slightly verbose.
- **Auto-upgrade.** Set Slack bots to `multibot-mentions`. Single-bot
  threads stay easy; once a second bot shows up, the user has to
  disambiguate.

`max_bot_turns` and `allow_bot_messages` are unaffected by this gate —
they control bot-to-bot, not human-to-bot routing.

---

## 4. Bot → Bot in Thread / Bot 互動

```
# thread
> BotA emits "[[reply_to:msg_123]] @BotB please review"
> BotB receives the message (because BotA is in trusted_bot_ids)
> BotB replies
```

Gates:
- `allow_bot_messages` must be `mentions` or `all`.
- Optionally `trusted_bot_ids` filters which other bots are accepted.
- `max_bot_turns` caps runaway loops.
- The `[[reply_to:]]` directive ([concepts/output-directives](output-directives.md))
  helps make the conversation thread-able by humans afterwards.

Use cases:
- **Review → Deploy** handoff.
- **Plan → Code → Test** chain.
- **Critic / Coder** pair-programming pattern.

---

## Pattern relationships / 五者關係

```
0. DM           ◀── starting case (1:1, no thread)
       │
       ▼ adds: channels, threads, @mention gating
1. Channel @mention
       │
       ▼ adds: thread membership (involved)
2. Thread follow-up (no @mention)
       │
       ▼ adds: another bot in the thread
3. Multi-bot thread (mentions / multibot-mentions)
       │
       ▼ adds: bot messages count
4. Bot-to-bot (allow_bot_messages + trusted_bot_ids + max_bot_turns)
```

Each step adds a primitive; earlier steps still work the same way.

---

## Cross-references / 交叉引用

- Where these gates live in the pipeline:
  [architecture/message-flow](../architecture/message-flow.md).
- Detecting threads correctly: [concepts/thread-detection](thread-detection.md).
- The multi-agent topologies built on top of these patterns:
  [concepts/multi-agent](multi-agent.md).
- Full prose walk-through: `../docs/messaging.md`.
