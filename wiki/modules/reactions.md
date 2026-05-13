---
title: "`reactions` module — Status emoji FSM"
type: module
sources:
  - ../src/reactions.rs
  - ../config.toml.example
related: [adapter, ../architecture/message-flow]
updated: 2026-05-12
---

`src/reactions.rs` (276 行) 是 `StatusReactionController` — 一個 debounce +
stall-aware 的有限狀態機,負責把 agent 的當前狀態翻譯成訊息上的單一 emoji:
👀 排隊 → 🤔 思考 → 🔥/👨‍💻/⚡ 動作中 → 🆗 完成(+ 可選的心情 emoji)。
受 OpenClaw 的 `StatusReactionController` 啟發。

`reactions.rs` (276 LOC) is `StatusReactionController` — a debounce-aware,
stall-detecting state machine that translates the current ACP event stream
into **one** emoji reaction on the user's triggering message.

---

## States & emojis / 狀態與表情

Default (from `../config.toml.example`):

```
👀  queued        message accepted, agent not yet running
🤔  thinking      agent emitted agent_thought_chunk
🔥  tool          generic ToolStart
👨‍💻 coding        coding-flavoured tool (edit/run/etc.) — title heuristic
⚡  web           web-flavoured tool (fetch/search) — title heuristic
🆗  done          turn completed
😱  error         turn errored
```

All emojis are user-configurable under `[reactions.emojis]`.

---

## Why debounce + stall-aware / 為何需要 debounce 與 stall 偵測

Without debouncing, rapid `tool_call_update` chatter would flap the emoji at
the platform's rate limit — particularly bad on Discord which throttles
reaction add/remove. The FSM:

- **Debounces** at `debounce_ms` (default 700ms) — only commit a state
  transition after the new state has held for that window.
- **Stall-detects** in two tiers:
  - `stall_soft_ms` (10s default) — silently swap to a "still thinking" hint
    if no events arrive.
  - `stall_hard_ms` (30s default) — surface the stall to the user (typically
    by holding 🤔 instead of escalating).
- **Holds** terminal states for `done_hold_ms` / `error_hold_ms` so the
  emoji is readable before being cleared (if `remove_after_reply = true`).

---

## Configuration / 設定

```toml
[reactions]
enabled = true
remove_after_reply = false  # keep the 🆗 in place after the agent finishes

[reactions.emojis]
queued = "👀"
thinking = "🤔"
tool = "🔥"
coding = "👨‍💻"
web = "⚡"
done = "🆗"
error = "😱"

[reactions.timing]
debounce_ms     = 700
stall_soft_ms   = 10_000
stall_hard_ms   = 30_000
done_hold_ms    = 1_500
error_hold_ms   = 2_500
```

The Helm chart exposes `reactions.enabled`, `reactions.removeAfterReply`,
and `reactions.toolDisplay` ("full" / "compact" / "none") for convenience.
The fine timing knobs are config-only.

---

## Tool display verbosity / 工具顯示詳簡

Separate from the emoji FSM but bundled in the same surface:

| `toolDisplay` | What appears in the message body |
|---|---|
| `full` (default) | each tool call gets a one-line entry with title and status |
| `compact` | only summary lines (e.g. "3 tools used") |
| `none` | tool calls suppressed entirely; only agent text shown |

---

## Where it slots into the pipeline / 在管線中的位置

```
AcpEvent  →  StatusReactionController::step(state)
                       │
                       └── if state changed (post-debounce):
                              ├── add_reaction(emoji)
                              └── remove previous emoji
```

The controller is held per-triggering-message and is **per-platform-agnostic**
— it talks to `ChatAdapter::add_reaction` / `remove_reaction`. Platforms
without reaction support (LINE, Telegram*) simply receive a no-op call.

> *Telegram does support reactions, and `gateway/src/main.rs` tracks them
> per-message; LINE genuinely has no reactions and the call is dropped.

---

## Pattern lineage / 啟發來源

The original pattern (and emoji choices) comes from
[OpenClaw](https://github.com/openclaw/openclaw)'s
`StatusReactionController` — credited in `../README.md` § Inspired By.
