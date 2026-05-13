---
title: "ADR-005 — Turn-Boundary Message Batching"
type: adr
sources:
  - ../docs/adr/turn-boundary-batching.md
related: [../modules/dispatch, ../modules/acp, ../architecture/message-flow]
updated: 2026-05-12
---

> **Status**: Proposed.
> **Date**: 2026-04-29 · **Author**: @brettchien.
> **Tracking issue**: #580 · **Implementation PR**: #686.
> **Anchor pinning**: released-code anchors pinned to v0.8.2-beta.1
> (`52052b8`); new modules described conceptually.

把「1 訊息 = 1 ACP turn」改成「**期間內到達的 N 訊息 = 下一 turn 1 次處理**」。
用 **per-thread bounded `mpsc::channel`** 在 turn 進行中緩衝後到的訊息,turn 結
束時 greedy-drain 打包成單一 `session_prompt` 的 `Vec<ContentBlock>`。第一則訊
息零延遲。實作在 `src/dispatch.rs`。

Replace "1 message = 1 ACP turn" with "**N messages arriving during one
turn → 1 next turn**". A per-thread bounded `mpsc::channel` buffers
messages that arrive during an in-flight turn; the consumer greedy-drains at
the turn boundary and packs them into one `session_prompt` `Vec<ContentBlock>`.
**Zero added latency** for the first message.

---

## Problem / 問題

Three concrete workload patterns the old "1→1" behaviour hurt:

1. **Stream-of-thought split** — `"can you check the build"` → `"actually wait"`
   → `"check the build *and* run the e2e tests"` in 5s. Old: 3 turns; turn
   1 wastes work, turn 2 reacts before correction, turn 3 finally has full
   intent.
2. **Late attachment** — text question, then 8s later the screenshot.
   Old: 2 turns; first answers without the screenshot.
3. **Interleaved topics** — two unrelated asks back-to-back. The broker
   should merge; the agent handles multi-intent prompts well.

---

## Why at the broker / 為何放在 broker 層

- ACP CLIs (Claude Code, Codex, Cursor) consume **one turn at a time**;
  they don't introspect chat traffic and don't batch themselves.
- Tokio `Mutex` is FIFO-ish but can't batch — wakers are opaque.
- Adapter-level pre-turn debouncing imposes latency on **every** message
  (including isolated ones). The broker can buffer **only during** an
  in-flight turn, when the user is already waiting on the agent —
  zero-latency first message, free batching after.

⇒ The broker is the only layer that can do this correctly.

---

## Mechanism / 機制

A long-lived per-thread (or per-lane) consumer task. Producers (platform
event loops) call `Dispatcher::submit(thread_key, ContentBlock)`. The
consumer:

- Holds the per-connection mutex (RFC #78 §2b) for the in-flight ACP turn.
- On turn completion, **greedy-drains** the channel until empty (subject
  to `max_batch_tokens`).
- Builds **one** `Vec<ContentBlock>` from all drained items and issues
  one `session/prompt`.

```
producer (event loop)         consumer (dispatch task)        ACP pool
─────────────────────         ────────────────────────        ────────
  submit(t, block) ─► mpsc ─►  drain on turn boundary
                                Vec<ContentBlock>           ─► session_prompt
                                  └── 1 ACP turn for N msgs
```

Three modes (`message_processing_mode`):

| Mode | Buffer key | When |
|---|---|---|
| `per-message` (default) | none | preserve v0.8.2-beta.1 behaviour |
| `per-thread` | thread_key | one batch per thread |
| `per-lane` | (thread_key, sender_id) | no silent-drop risk for multi-sender |

---

## Invariants / 三大不變式

ADR § 2.1:

1. **Zero added latency on first message** — if no turn is in flight,
   dispatch is identical to the old path.
2. **Deterministic same-thread ordering** — within one thread, messages
   are processed in arrival order.
3. **Per-connection mutex unchanged** — RFC #78 §2b's per-thread mutex is
   the only inter-thread isolation mechanism; the dispatcher does not
   replace it.

---

## Non-goals / 不做

| Concern | Owner |
|---|---|
| Inter-thread isolation | RFC #78 §2b's per-connection mutex |
| Cross-session blocking (#307) | a different layer (new-thread session startup) |
| Pre-turn debouncing | rejected (adds first-message latency) |
| Topic detection / semantic grouping | deferred to the agent |
| Cancelling in-flight turns | unchanged; `/cancel` / `/cancel-all` |
| Persisting buffer across pod restart | in-flight turn is lost anyway |

---

## Status / 狀態

Proposed in the ADR; implementation landed via PR #686 (Phase 1 wiring),
producing `src/dispatch.rs`. The default mode remains `per-message` to
preserve v0.8.2-beta.1 behaviour for existing deployments.

---

## Related / 相關

- Module page: [modules/dispatch](../modules/dispatch.md).
- ACP session prompt mechanics: [modules/acp](../modules/acp.md).
- Pipeline placement: [architecture/message-flow](../architecture/message-flow.md).
