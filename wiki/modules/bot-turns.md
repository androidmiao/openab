---
title: "`bot_turns` — Bot Loop Guard"
type: module
sources:
  - ../src/bot_turns.rs
  - ../config.toml.example
related: [../concepts/multi-agent, config, adapter]
updated: 2026-05-12
---

`src/bot_turns.rs` (352 行) 維護「每個 thread 的連續 bot turn 計數器」。當
`allow_bot_messages` ≠ `off` 時,bot 之間有機會無限互戳;此模組是最後一道保險:
連續 N 個 bot turn 後(預設 100)強制 throttle 直到人類介入。**人類訊息一進來
立刻重置計數器**。

`bot_turns.rs` (352 LOC) maintains a per-thread counter of **consecutive bot
turns**. When `allow_bot_messages` is not `off`, two bots can chat forever;
this counter is the last-resort cap. The default is 100 turns. A human
message resets the counter immediately.

---

## State / 狀態

A `BotTurnTracker` holds, per `thread_key`:

- `consecutive_bot_turns: u32`
- `last_human_message_ts: Option<Instant>`

That's it. Cheap, in-memory, lost on restart (intentionally — restarts are a
natural reset boundary).

---

## Behaviour / 行為

```
on_message(sender_is_human):
    if sender_is_human:
        consecutive_bot_turns = 0
        last_human_message_ts = now()
    else:
        consecutive_bot_turns += 1
        if consecutive_bot_turns > max_bot_turns:
            return Throttle  // adapter rejects the message before ACP dispatch
        return Accept
```

`max_bot_turns` is per-adapter (`[discord].max_bot_turns`, default 100).

---

## What "throttle" means / Throttle 的意義

When the cap is exceeded:

- The message is **not** dispatched to the agent.
- The adapter may log a warning so an operator notices the loop.
- The state stays "throttled" until any human posts; that human message
  resets `consecutive_bot_turns` to zero and unblocks the thread.

There is **no exponential backoff** and **no auto-recovery** — the design
choice is that the cap should be high enough to be unreachable in legitimate
use, and that crossing it implies an actual loop that needs human attention.

---

## Where it slots in / 管線位置

Inside gate ③ "bot message filter" of the inbound pipeline
([architecture/message-flow](../architecture/message-flow.md)):

- `allow_bot_messages` decides **whether** bot messages count at all.
- `trusted_bot_ids` filters **which** bots count.
- `BotTurnTracker` is the **rate limiter** on the surviving traffic.

---

## Cross-references / 交叉引用

- Multi-agent design considerations: [concepts/multi-agent](../concepts/multi-agent.md).
- Why this is needed at all (the OpenClaw / Hermes Agent precedents):
  [architecture/design-philosophy § 2 Multi-Bot Ready](../architecture/design-philosophy.md#2-multi-bot-ready--多-bot-共存).
