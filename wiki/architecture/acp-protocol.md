---
title: "ACP Protocol / ACP 協定"
type: architecture
sources:
  - ../src/acp/protocol.rs
  - ../src/acp/connection.rs
  - ../src/acp/pool.rs
related: [system-overview, message-flow, ../modules/acp]
updated: 2026-05-12
---

ACP (Agent Client Protocol) 是 [Anthropic 提出的 agent-CLI 對 broker 的標準介
面](https://github.com/anthropics/agent-protocol):**JSON-RPC over stdio**。
OpenAB 用 ACP 與每一個 agent CLI(Kiro、Claude Code、Codex、Gemini、OpenCode、
Copilot、Cursor)交談。本頁總結 OpenAB 用到的子集與分類規則。

ACP is the [Agent Client Protocol](https://github.com/anthropics/agent-protocol)
— **JSON-RPC over stdio** between a broker and an agent CLI. OpenAB
implements the broker side; every agent CLI it supports implements the agent
side (natively or via a thin adapter).

---

## Why JSON-RPC over stdio / 為什麼是 stdio JSON-RPC

From `../DESIGN.md`:

- No HTTP server in the agent — the agent is a **subprocess**, not a service.
- No network exposure — the channel is stdin/stdout pipes.
- Any ACP-compatible CLI works — swap agents with one config line.

對 OpenAB 來說這就等於:agent 永遠在 sandbox 裡跑、永遠不開 port、換 agent 只
需改 `[agent].command`。

---

## Message types / 訊息類型

Source: `../src/acp/protocol.rs`.

### Requests (OpenAB → agent)

```rust
pub struct JsonRpcRequest {
    pub jsonrpc: &'static str, // always "2.0"
    pub id: u64,
    pub method: String,
    pub params: Option<Value>,
}
```

Methods OpenAB issues today:

| Method | Purpose |
|---|---|
| `initialize` | handshake; receive `serverCapabilities` (`supportsLoadSession`, etc.) |
| `session/new` | open a session; receive `sessionId` |
| `session/load` | resume a suspended session by `sessionId` |
| `session/prompt` | send the user's turn as `Vec<ContentBlock>` |
| `session/cancel` | abort the in-flight turn |
| `session/setOption` | update a `configOption` (e.g. switch model / mode) |

### Notifications (agent → OpenAB)

The agent streams notifications back via stdout. `classify_notification`
(`../src/acp/protocol.rs:203-263`) maps each `sessionUpdate` into an `AcpEvent`:

```rust
pub enum AcpEvent {
    Text(String),                                  // agent_message_chunk
    Thinking,                                      // agent_thought_chunk
    ToolStart  { id: String, title: String },      // tool_call (or tool_call_update with non-terminal status)
    ToolDone   { id: String, title: String, status: String }, // tool_call_update with completed/failed
    ConfigUpdate { options: Vec<ConfigOption> },   // config_option_update
    Status,                                        // plan
}
```

Key design choices:

- **`toolCallId` is the stable identity** across `tool_call` → `tool_call_update`.
  Without it claude-agent-acp's two-phase emission would render twice (a
  placeholder + the refined version) — see the inline comment in `protocol.rs`.
- **Terminal status** (`completed` / `failed`) routes through `ToolDone`; any
  other status maps to `ToolStart` so the reaction FSM can keep ticking.
- **Plans** (`plan` updates) all collapse to `Status` — OpenAB doesn't render
  plan trees, it just keeps the reaction "active".

### Permission requests / 工具權限請求

When the agent wants to use a tool that needs operator confirmation, it
issues a JSON-RPC **request** (not a notification) with method
`session/request_permission`. OpenAB auto-replies with the most permissive
selectable option. Details: [security-model § 5](security-model.md#5-acp-permission-auto-reply--工具權限自動回應).

---

## `ConfigOption` / 設定選項

ACP defines a generic `configOptions` array on `session/new` and
`config_option_update`. OpenAB renders these as a Discord select-menu / Slack
modal so users can switch model / mode on the fly.

```rust
pub struct ConfigOption {
    pub id: String,           // e.g. "model"
    pub name: String,
    pub description: Option<String>,
    pub category: Option<String>,
    pub option_type: String,  // "enum"
    pub current_value: String,
    pub options: Vec<ConfigOptionValue>,
}
```

### Kiro-cli compatibility shim

`kiro-cli` does **not** emit `configOptions`; instead it emits an older
`models` / `modes` shape. `parse_config_options` (`protocol.rs:93-180`) has a
fallback that converts:

- `models.availableModels[]` → a synthetic `ConfigOption { id: "model" }`.
- `modes.availableModes[]` → a synthetic `ConfigOption { id: "agent" }`.

Standard `configOptions` takes precedence — if both are present, the legacy
shape is ignored (tested in `protocol.rs:347-364`).

---

## `ContentBlock` / 內容區塊

Source: `../src/acp/connection.rs`.

```rust
pub enum ContentBlock {
    Text  { text: String },
    Image { media_type: String, data: String }, // base64
}
```

Each ACP `session/prompt` carries `Vec<ContentBlock>`. A single user turn can
combine text + multiple images + transcribed voice. With ADR-005's
turn-boundary batching enabled, **N user messages** can be packed into one
`session/prompt`'s `Vec<ContentBlock>`.

---

## Connection lifecycle / 連線生命週期

For one `AcpConnection` (`../src/acp/connection.rs`):

```
spawn child (env_clear + allowlist)
  └─ stdin pipe, stdout pipe
  ▼
initialize
  └─ record supports_load_session
  ▼
session/new
  └─ record acp_session_id, config_options
  ▼
loop:
  session/prompt (one or more)
  notifications stream back (Text / Thinking / Tool* / Config*)
  optional session/cancel
  ▼
on eviction:
  if supports_load_session:
    record acp_session_id in ~/.openab/thread_map.json (suspended)
  drop child (SIGTERM via process-group)
```

On next message for the same `thread_key`, if a suspended `sessionId` exists
and the agent supports `session/load`, the broker resumes instead of starting
fresh. See [concepts/session-pool](../concepts/session-pool.md).

---

## Where to look in the code / 對應程式

| Concern | File |
|---|---|
| JSON-RPC types, `AcpEvent`, classification, `ConfigOption`, Kiro fallback | `../src/acp/protocol.rs` |
| Spawn CLI, env-clear, JSON-RPC plumbing, permission auto-reply | `../src/acp/connection.rs` |
| `thread_key → AcpConnection` map, eviction, suspended-session resume | `../src/acp/pool.rs` |
| Notification → adapter event routing | `../src/adapter.rs` |
