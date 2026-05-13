---
title: "`dispatch` module — Turn-Boundary Message Batching"
type: module
sources:
  - ../src/dispatch.rs
  - ../docs/adr/turn-boundary-batching.md
related: [acp, adapter, ../adrs/adr-005-turn-boundary-batching, ../architecture/message-flow]
updated: 2026-05-12
---

`src/dispatch.rs` (1509 行)實作 ADR-005:每個 thread (或 lane) 一個 bounded
`mpsc::channel`,訊息進來時若已有 turn 在跑就**緩衝**起來,等該 turn 結束時把
緩衝的 N 則訊息**一次**打包成單一 ACP `session/prompt` 的 `Vec<ContentBlock>`。
取代「1 訊息 = 1 turn」的舊行為,解決三種典型情境:思緒分段、晚到附件、
獨立話題交錯。

`dispatch.rs` (1509 LOC) implements ADR-005. A bounded per-thread (or
per-lane) `mpsc::channel` buffers messages that arrive **while an ACP turn
is in flight** and dispatches them as one batched `session/prompt` at the
next turn boundary — replacing "1 message = 1 turn" with "N → 1".

---

## Why a queue, why per-thread / 為何是 per-thread 佇列

From ADR-005:

- Tokio's `Mutex` is FIFO-ish but cannot batch — wakers are opaque.
- ACP CLIs (Claude Code, Codex, Cursor) consume one turn at a time; they
  don't introspect chat traffic.
- Adapter-level pre-turn debouncing imposes latency on every message; the
  broker can buffer **only during** an in-flight turn (when the user is
  already waiting on the agent), paying **zero added latency** to the
  first message.

That makes the broker the only layer that can do this correctly.

---

## Modes / 模式

Driven by `[discord/slack/gateway].message_processing_mode`
(default `per-message`):

| Mode | Buffer key | Why |
|---|---|---|
| `per-message` | none | preserves v0.8.2-beta.1 behaviour — no buffering at all |
| `per-thread` | `thread_key` | one shared batch per thread |
| `per-lane` | `(thread_key, sender_id)` | one batch per sender per thread — solves silent-drop risk in multi-sender threads |

`per-lane` is the **deadlock-free** option for multi-bot channels: bot A's
messages can't drown bot B's because they live in different lanes.

---

## Three workload patterns this fixes / 解決哪些問題

1. **Stream-of-thought split** — `"can you check the build"` → `"actually wait"`
   → `"check the build *and* run the e2e tests"` in 5 seconds.
   Before: 3 sequential turns; turn 1 wastes work, turn 2 reacts before the
   correction, turn 3 finally has full intent.
   After: 1 turn with the corrected combined ask.
2. **Late attachment / clarification** — text question now, screenshot 8s
   later. Before: 2 turns, first answers without the screenshot. After: 1.
3. **Independent topics interleaved** — two unrelated asks back-to-back. The
   broker merges them; the agent handles multi-intent prompts well.

---

## Limits / 配額

| Knob | Default | Purpose |
|---|--:|---|
| `max_buffered_messages` | 10 | bounded channel capacity per thread/lane |
| `max_batch_tokens` | 24_000 | soft token cap when greedy-draining |

When the bound is reached, the producer either drops oldest with a warning or
flushes early — see `dispatch.rs` for exact backpressure behaviour at the
time of any given commit.

---

## Packing / 打包

Each arrival ⇒ one or more `ContentBlock`s (text, image, transcribed audio).
At drain time, the dispatcher concatenates them in arrival order. Multiple
images in one batch end up as multiple `ContentBlock::Image` entries within
the same `session/prompt`.

The ACP server (the agent CLI) sees this as **one user turn** — same as if
the user pasted everything at once.

---

## What this module does **not** do / 刻意不做

ADR-005 § 1.3 lists the non-goals:

| Concern | Owned by |
|---|---|
| Inter-thread isolation | the per-connection mutex in `acp::pool` (RFC #78 §2b) |
| Cross-session blocking (#307) | a different layer (new-thread session startup) |
| Pre-turn debouncing | rejected (would add latency to isolated messages) |
| Topic detection / semantic grouping | deferred to the ACP agent |
| Cancelling in-flight turns | unchanged; `/cancel` and `/cancel-all` still work |
| Persisting buffer across pod restarts | the in-flight turn is lost on restart anyway |
| Replacing the per-connection mutex | mutex stays exactly as ACP pool already uses it |

---

## Cross-references / 交叉引用

- The ADR with full mechanism + invariants: [adrs/adr-005-turn-boundary-batching](../adrs/adr-005-turn-boundary-batching.md).
- Where the dispatcher slots into the pipeline: [architecture/message-flow](../architecture/message-flow.md).
- Underlying ACP method invoked at drain: [modules/acp § session_prompt](acp.md#connectionrs-901-loc).
