---
title: "Thread Detection — the definitive rules"
type: concept
sources:
  - ../AGENTS.md
  - ../src/discord.rs
related: [../adapters/discord, ../adapters/slack, ../architecture/message-flow]
updated: 2026-05-12
---

`../AGENTS.md` Critical Rule #2 把 thread 偵測列為硬規則,因為這是最常被搞錯的
地方。**`thread_metadata` 是真值,不要看 `parent_id`**;**`bot_owns` = thread
的 `owner_id` 等於 bot user ID**;**用 `ChannelType` enum,不要做結構性 heuristic**;
**論壇貼文是 `parent_id` 指向 forum channel 的 thread**。

`../AGENTS.md` Critical Rule #2 makes thread detection a **hard rule**
because it's the single most error-prone area when modifying adapters.

---

## The four rules / 四條鐵則

Quoting `../AGENTS.md` verbatim:

> The definitive rules — do NOT reinvent this:
> - A channel is a thread if `thread_metadata` is present (NOT `parent_id`)
> - `bot_owns` = thread's `owner_id` matches the bot's user ID
> - Use `ChannelType` enum, not structural heuristics
> - Forum posts are threads with `parent_id` pointing to a forum channel

### Why `thread_metadata`, not `parent_id`?

Many non-thread channels have a `parent_id` (e.g. a channel inside a
category has the category as parent). `parent_id` is meaningful for "where
does this live in the channel hierarchy"; `thread_metadata` is meaningful
for "is this a thread". They're orthogonal in the data model.

Discord exposes `thread_metadata` only on actual threads — public, private,
and forum-post variants.

### Why `bot_owns` matters / 為何要追蹤 bot 擁有

`bot_owns` decides whether the bot can reply in a thread **without an
@mention**. Combined with `allow_user_messages: "involved"`, this lets the
bot continue a conversation it started without requiring users to keep
@-pinging it in every follow-up.

The check is **strict identity match** (thread `owner_id` == bot's
`user_id`), not "has the bot ever posted in this thread". Otherwise a bot
that wandered into a human-owned thread would gain follow-up privileges,
which is the wrong behaviour.

### Why `ChannelType`, not heuristics / 為何用 enum

Earlier prototypes tried "thread iff there's a `last_message_id` but no
`topic`" or similar structural guesses. They all break on edge cases
(forum posts, archived threads, parents that lack topics, …). The
`ChannelType` enum is Discord's intent layer:

```rust
match channel.kind {
    ChannelType::PublicThread |
    ChannelType::PrivateThread |
    ChannelType::NewsThread     => true,    // is_thread
    ChannelType::Forum          => false,   // forum *channel* is not a thread
    _                           => false,
}
```

Forum **posts** are `ChannelType::PublicThread` whose `parent_id` is a
`ChannelType::Forum`. So the "forum post → thread" identification falls
out for free from the enum + parent check.

---

## Slack / Slack

Slack threads are simpler:

```rust
fn is_thread_event(event) -> bool {
    event.thread_ts.is_some() && event.thread_ts != Some(event.ts)
}
```

`bot_owns` on Slack = the bot is the user that started the thread (the
`thread_ts` message's `user` == bot user ID).

---

## Gateway / Gateway

The unified schema's `channel.thread_id` is `Option<String>`:

- Discord-mapped: thread channel ID.
- Slack-mapped: `thread_ts`.
- Feishu-mapped: `parent_id`.
- Google Chat-mapped: `thread.name`.
- Teams-mapped: root activity ID.
- LINE / Telegram: always `None` (no threads).

Each gateway adapter is responsible for setting this correctly — see
[adapters/gateway-service](../adapters/gateway-service.md) for the
contract.

---

## Where this matters most / 影響範圍

The session pool keys conversations on thread identity. Misidentifying a
non-thread as a thread (or vice versa) means:

- Two unrelated conversations sharing one agent process.
- Or a single conversation getting a new agent process per message
  (no memory across turns).

Both are bad. The four rules above keep them from happening.
