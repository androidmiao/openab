---
title: "Message Flow / 訊息流"
type: architecture
sources:
  - ../AGENTS.md
  - ../src/adapter.rs
  - ../src/dispatch.rs
  - ../src/acp/connection.rs
  - ../src/acp/pool.rs
related: [system-overview, acp-protocol, ../modules/dispatch, ../modules/adapter, ../concepts/thread-detection]
updated: 2026-05-12
---

一則 user message 從 Discord / Slack / gateway 進來,到 agent 產生回覆並推回平
台,中間經過 7 道濾鏡 + 1 個 dispatch queue + 1 個 ACP session。理解這個管線後,
所有「為什麼沒收到」「為什麼收兩次」的問題都有檢查路徑。

A single inbound message passes **seven gates → one dispatch queue → one ACP
session → streamed edit-back-to-platform**. Understanding this pipeline is a
prerequisite for debugging any "why didn't it reply" or "why did it reply
twice" question.

---

## The seven inbound gates / 七道濾鏡

Source: `../AGENTS.md` (the broker pipeline) and `../src/adapter.rs`.

```
Discord/Slack/Gateway event
  │
  ├─ ① channel allowlist check        (allowed_channels / allow_all_channels)
  ├─ ② user allowlist check           (allowed_users / allow_all_users) — bot msgs skip
  ├─ ③ bot message filter             (allow_bot_messages: off | mentions | all)
  ├─ ④ thread detection               (thread_metadata is the truth, NOT parent_id)
  ├─ ⑤ bot ownership check            (bot_owns = thread.owner_id == bot_user_id)
  ├─ ⑥ multibot detection             (allowUserMessages mode)
  └─ ⑦ trigger check                  (@mention / role mention / DM-as-mention)
     │
     ▼
  handle_message → Dispatcher::submit
     │
     ▼
  Dispatcher (per-thread mpsc buffer; ADR-005)
     │  drains greedily at turn boundary
     ▼
  pool.get_or_create(thread_key)
     │
     ▼
  AcpConnection::session_prompt(content_blocks)
     │  JSON-RPC stdin
     ▼
  agent CLI (kiro / claude / codex / ...)
     │
     ▼
  notifications stream back via stdout
     │  classify_notification → AcpEvent::{Text, Thinking, ToolStart, ToolDone, ConfigUpdate, Status}
     ▼
  AdapterRouter
     │  ┌─ reaction FSM update (👀→🤔→🔥→🆗)
     │  ├─ markdown render + table mode
     │  ├─ message split (Discord 2000 / Slack ~40k / LINE 5000)
     │  └─ edit-stream (debounce 1.5s) ──► platform
     ▼
  agent done → final edit + 🆗 reaction (+ optional mood)
```

The pipeline is shared by all platforms. The first three gates and the last
two (edit-stream + reaction) live in **adapter-specific code** because each
platform has its own API; everything in the middle is **platform-agnostic**.

---

## Gate-by-gate detail / 逐道細節

### ① Channel allowlist / 頻道白名單

- Source: `[discord].allowed_channels` / `[slack].allowed_channels` /
  `[gateway].allowed_channels`.
- Resolved with `allow_all_channels`: explicit → wins; omitted → auto-detect
  (empty list = allow all, non-empty list = restrict). See
  `../src/config.rs` (`allow_all_channels: Option<bool>`).

### ② User allowlist / 使用者白名單

- Same semantics as channel allowlist (`allowed_users`, `allow_all_users`).
- **Bot messages bypass this gate** — they go through gate ③ instead.

### ③ Bot message filter / Bot 訊息過濾

`AllowBots` enum (`../src/config.rs`):

| Value | Behaviour |
|---|---|
| `off` (default) | ignore all bot messages — including the bot's own (always ignored anyway). |
| `mentions` | only process bot messages that explicitly @mention this bot. Natural loop breaker. |
| `all` | process all bot messages; hard-capped at `max_bot_turns` consecutive turns. |

Plus `trusted_bot_ids` to further restrict which bots count.

### ④ Thread detection / Thread 偵測

> "thread_metadata is the truth, NOT parent_id."

The definitive rules from `../AGENTS.md` § Thread Detection — restated in
[concepts/thread-detection](../concepts/thread-detection.md):

- A channel is a thread iff `thread_metadata` is present.
- Forum posts are threads whose `parent_id` points at a forum channel.
- `bot_owns` = the thread's `owner_id` matches the bot user ID.

### ⑤ Bot ownership / Bot 擁有

`bot_owns` determines whether the bot can reply in a thread **without an
@mention**. The bot owns a thread when it (or an automated cron) created
the thread.

### ⑥ Multibot detection / 多 bot 偵測

`AllowUsers` enum:

| Value | Behaviour |
|---|---|
| `involved` (default) | reply in threads where the bot has participated (posted ≥1 msg or thread parent @mentions it). |
| `mentions` | always require @mention. |
| `multibot-mentions` | `involved` until a second bot posts; then upgrades to `mentions`. |

### ⑦ Trigger check / 觸發

A message **triggers a turn** when:

- it @mentions the bot user, **or**
- it mentions a role in `allowed_role_ids`, **or**
- it's a DM and `allow_dm = true` (treated as implicit mention), **or**
- the user is already in an "involved" thread per gate ⑥.

---

## Dispatch queue (ADR-005) / 派發佇列

After the seven gates, the message lands in a **per-thread bounded mpsc
queue** (`max_buffered_messages`, default 10). The dispatcher consumer drains
greedily at **turn boundaries** and packs N arrival messages into one ACP
`session_prompt` call. This replaces "1 message → 1 turn" with
"N messages-arrived-during-turn → 1 next turn", which solves three concrete
patterns: stream-of-thought split, late attachment, interleaved topics.

ADR: [adrs/adr-005-turn-boundary-batching](../adrs/adr-005-turn-boundary-batching.md).
Module page: [modules/dispatch](../modules/dispatch.md).

Modes (`message_processing_mode`):

| Mode | Buffer key | When to use |
|---|---|---|
| `per-message` (default) | none — bypass queue | preserves v0.8.2-beta.1 behaviour |
| `per-thread` | thread_key | one batch per thread |
| `per-lane` | (thread_key, sender_id) | per sender — avoids silent drops when multiple senders address the same thread |

---

## ACP session / ACP 對話

The thread_key maps (via `SessionPool`) to **one** `AcpConnection`, which
**owns one CLI subprocess**. The first message creates the connection
(`session/new`), subsequent messages reuse it (`session/prompt`). If the pool
is full, the least-recently-used connection is evicted and its session ID is
saved to `~/.openab/thread_map.json` for later resume via `session/load`.

See [concepts/session-pool](../concepts/session-pool.md) and
[modules/acp](../modules/acp.md).

---

## Outbound stream / 對外串流回覆

`classify_notification` (`../src/acp/protocol.rs`) maps ACP `sessionUpdate`
events to `AcpEvent` variants. The `AdapterRouter` consumes them and:

1. Updates the **reaction FSM** (👀 queued → 🤔 thinking → 🔥/👨‍💻/⚡
   active → 🆗 done; with stall detection).
2. Renders the accumulated text through `markdown.rs` (Discord-flavoured;
   table mode `code` / `bullets` / `off`).
3. Splits at platform-specific length limits via `format.rs`.
4. **Edit-streams** the final message in place every ~1.5s instead of posting
   new messages — keeps thread noise low.
5. On `tool_call_update.status = completed`, emits the tool-done line and
   updates the reaction.

Output directives (`[[reply_to:id]]`) consumed at the **start** of agent
output let agents redirect their reply to a specific upstream message — see
[concepts/output-directives](../concepts/output-directives.md).
