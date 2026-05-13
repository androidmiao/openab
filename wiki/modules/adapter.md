---
title: "`adapter` module — ChatAdapter trait / 平台無關抽象"
type: module
sources:
  - ../src/adapter.rs
  - ../docs/adr/multi-platform-adapters.md
  - ../docs/output-directives.md
related: [acp, ../adapters/discord, ../adapters/slack, ../adapters/gateway-client, ../concepts/output-directives, ../adrs/adr-001-multi-platform-adapters]
updated: 2026-05-12
---

`src/adapter.rs` (1198 行) 是 OpenAB 平台無關層的中樞:`ChatAdapter` trait
定義所有平台共通的能力,`AdapterRouter` 把 ACP 訊號統一翻譯成平台動作,
`OutputDirectives` 解析 `[[reply_to:…]]` 等 agent 端指令。

`adapter.rs` (1198 LOC) is the platform-agnostic core: `ChatAdapter` is the
trait every platform implements, `AdapterRouter` translates ACP events into
platform actions, and `OutputDirectives` parses agent-emitted `[[key:value]]`
header directives.

---

## ChatAdapter trait / 平台 trait

ADR-001 (multi-platform adapters) defines the trait shape. Every platform
exposes:

```rust
#[async_trait]
pub trait ChatAdapter: Send + Sync + 'static {
    fn platform(&self) -> &'static str;             // "discord", "slack", "telegram", …
    fn message_limit(&self) -> usize;               // 2000 / ~40_000 / 5000 …
    async fn run(&self, router: Arc<AdapterRouter>) -> Result<()>;
    async fn send_message(&self, channel: &ChannelRef, content: &str) -> Result<MessageRef>;
    async fn edit_message(&self, msg: &MessageRef, content: &str) -> Result<()>;
    async fn create_thread(&self, ...) -> Result<ChannelRef>;
    async fn add_reaction(&self, msg: &MessageRef, emoji: &str) -> Result<()>;
    async fn remove_reaction(&self, msg: &MessageRef, emoji: &str) -> Result<()>;
    /* ... */
}
```

| Impl | File | Notes |
|---|---|---|
| `DiscordAdapter` | `../src/discord.rs` | serenity Gateway WS, native API |
| `SlackAdapter` | `../src/slack.rs` | Socket Mode WS + Web API |
| `GatewayAdapter` | `../src/gateway.rs` | WebSocket client to the standalone gateway |

---

## `AdapterRouter`

The router is the **single consumer of ACP notifications** for the broker. It
owns:

- A reference to the `SessionPool` for `pool.with_connection(...)`.
- The `Dispatcher` (ADR-005 turn-boundary batching).
- The per-message reaction FSM via `StatusReactionController`.
- `MarkdownConfig` (table mode, etc.) and the user-facing error formatter.

Pseudocode flow per inbound user message:

```
adapter platform → adapter.handle_message
  → Dispatcher::submit(thread_key, ContentBlock)
  → drain at turn boundary
  → SessionPool::with_connection(thread_key, |conn| conn.session_prompt(...))
  → for each notification:
       AdapterRouter::dispatch_event(thread_key, AcpEvent)
         ├─ StatusReactionController step
         ├─ markdown render
         ├─ edit-stream to platform
         └─ on tool_call_update.completed: emit tool-done line
```

The router is **platform-agnostic** — it never reaches into Discord-specific
or Slack-specific data structures; everything goes through the trait's
`ChannelRef` / `MessageRef`.

---

## Output directives / Agent 端輸出指令

Agents prepend `[[key:value]]` lines at the **very start** of their output to
request adapter-side behaviour. Currently OpenAB recognizes:

| Directive | Effect |
|---|---|
| `[[reply_to:<message_id>]]` | reply specifically to that platform message ID (Discord `message_reference`, Slack `thread_ts` override) |

Parsing rules in `parse_output_directives` (`adapter.rs:26-97`):

- Only **consecutive `[[key:value]]` lines at the very beginning** count;
  anything else stops the parser.
- The value must be 1–64 chars, alphanumerics + `.`, `-`, `_`. Anything else
  drops the directive silently.
- Trailing text on the same line as a closing `]]` ends the directive header
  but is preserved as part of the content.
- Unknown keys log at `debug!` and are stripped.

Full user guide: `../docs/output-directives.md` and
[concepts/output-directives](../concepts/output-directives.md).

---

## Shared types / 共通型別

- `ChannelRef { platform, id, thread_id }` — uniformly identifies a channel
  or thread across platforms.
- `MessageRef { channel, id, ... }` — a sent platform message, used for edits
  and reactions.
- `OutputDirectives` — see above.

---

## Cross-references / 交叉引用

- The trait was introduced in [ADR-001: Multi-Platform Adapters](../adrs/adr-001-multi-platform-adapters.md).
- The router consumes events whose types are defined in
  [modules/acp § AcpEvent](acp.md#acp-events--事件).
- The output-directives parser is the canonical way for agents to influence
  adapter behaviour without OpenAB-side configuration changes — discussed in
  [concepts/output-directives](../concepts/output-directives.md).
