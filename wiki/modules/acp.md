---
title: "`acp` module — Protocol, Connection, Pool"
type: module
sources:
  - ../src/acp/mod.rs
  - ../src/acp/protocol.rs
  - ../src/acp/connection.rs
  - ../src/acp/pool.rs
related: [adapter, ../architecture/acp-protocol, ../concepts/session-pool]
updated: 2026-05-12
---

`src/acp/` 是 OpenAB 跟 agent CLI 對話的整個下半身。三個檔案各司其職:
**`protocol.rs`** 定義 ACP 訊息型別與分類規則(含 Kiro fallback);
**`connection.rs`** 啟動 agent subprocess、跑 JSON-RPC、處理權限自動回應;
**`pool.rs`** 用 thread_key 對應 `AcpConnection`、處理 LRU eviction 與
session resume。

`src/acp/` is the broker's lower half: `protocol.rs` defines the wire types
and classification logic, `connection.rs` spawns the agent CLI and runs
JSON-RPC, `pool.rs` is the `thread_key → AcpConnection` map.

The architectural overview lives in [acp-protocol](../architecture/acp-protocol.md);
this page is the **module-level** view.

---

## `protocol.rs` (385 LOC)

### Wire types

`JsonRpcRequest`, `JsonRpcResponse`, `JsonRpcMessage`, `JsonRpcError` — all
serde-deriving, all using `jsonrpc: "2.0"` literally.

### `ConfigOption` + Kiro fallback

`parse_config_options(result: &Value) -> Vec<ConfigOption>`:

- Standard path: read `configOptions[]` and deserialise.
- Kiro fallback: build a synthetic `ConfigOption { id: "model" }` from
  `models.{currentModelId, availableModels[]}` and a `ConfigOption { id: "agent" }`
  from `modes.{currentModeId, availableModes[]}`.
- Standard takes precedence (tested in `protocol.rs:347-364`).

### `AcpEvent` enum + `classify_notification`

Maps each `sessionUpdate` string to a typed event:

```
agent_message_chunk     → Text(text)
agent_thought_chunk     → Thinking
tool_call               → ToolStart { id, title }
tool_call_update        → ToolStart  (non-terminal) | ToolDone (completed | failed)
plan                    → Status
config_option_update    → ConfigUpdate { options }
(unknown)               → None
```

`toolCallId` is read from `update.toolCallId` and threaded through
`ToolStart`/`ToolDone` so the adapter can correlate the two phases of
claude-agent-acp's tool emission.

---

## `connection.rs` (901 LOC)

The heavy lifter. Responsibilities:

### 1. Spawn the agent CLI

- `env_clear()` then explicitly insert the allowlist (`HOME`, `PATH`,
  optionally `USER` / `USERPROFILE` / …).
- Apply `[agent].env` overrides (logged with prompt-injection warning).
- Apply `[agent].inherit_env` keys (named opt-in for K8s `envFrom` vars).
- Record `child_pgid` (process group ID on unix) so cleanup can SIGTERM the
  entire group on eviction.

### 2. Run JSON-RPC over stdio

- `next_id` (`AtomicU64`) gives request IDs.
- `pending: HashMap<u64, oneshot::Sender>` correlates response IDs to waiters.
- `notify_tx: Option<mpsc::UnboundedSender>` streams notifications to the
  `AdapterRouter`.
- A receive loop reads lines from the child's stdout, parses each as
  `JsonRpcMessage`, and:
    - if `id` matches a `pending` waiter → fulfil the request.
    - else if `method == "session/request_permission"` → call
      `build_permission_response` and `send_response`.
    - else → forward as a notification.

### 3. Permission auto-reply

`pick_best_option(options) → Option<String>`:

- Prefer `kind == "allow_always"` → `allow_once` → any non-reject option.
- Reject `reject_once`/`reject_always`.
- If nothing selectable, return `outcome: "cancelled"`.

This is intentionally **kind-based**, not metadata-based. See
[security-model § 5](../architecture/security-model.md#5-acp-permission-auto-reply--工具權限自動回應)
for the rationale (OpenClaw GHSA-7jx5-9fjg-hp4m).

### 4. Public state

`AcpConnection` exposes:

- `acp_session_id: Option<String>` — set after `session/new` or `session/load`.
- `supports_load_session: bool` — gleaned from `initialize` capabilities.
- `config_options: Vec<ConfigOption>` — updated on `config_option_update`.
- `session_prompt(blocks)`, `session_cancel`, `session_set_option`, … —
  high-level methods used by the dispatcher.

---

## `pool.rs` (490 LOC)

`SessionPool` is the **single owner** of all live `AcpConnection`s. Its
state is protected by **one `RwLock<PoolState>`** — lock ordering rule:
never await a per-connection mutex while holding `state`.

### `PoolState`

```rust
struct PoolState {
    active:          HashMap<String, Arc<Mutex<AcpConnection>>>,
    cancel_handles:  HashMap<String, (Arc<Mutex<ChildStdin>>, String)>,
    suspended:       HashMap<String, String>,   // thread_key → ACP sessionId
    creating:        HashMap<String, Arc<Mutex<()>>>, // serialise per-thread create
}
```

- `active` = currently-alive connections.
- `cancel_handles` = lock-free path for `/cancel` (just needs stdin + session ID).
- `suspended` = persisted across pod restarts via `~/.openab/thread_map.json`;
  the connection itself is gone but its ACP `sessionId` is remembered so the
  next request can `session/load`.
- `creating` = per-key gate that serialises concurrent
  "create or resume" requests for the same thread, preventing double `session/load`.

### Eviction

When `active.len() >= max_sessions`:

- Pick the **least-recently-used** active connection.
- If its agent supports `session/load`, move it to `suspended` (persist).
- Drop the child process (SIGTERM the process group).
- Remove from `active` / `cancel_handles`.

### Resume

On `with_connection(thread_key)`:

1. If active → reuse.
2. Else if suspended → `session/load(suspended_sessionId)`; on success, hoist
   into `active`.
3. Else → `session/new` fresh.

---

## Cross-references / 交叉引用

- Wire-level discussion: [architecture/acp-protocol](../architecture/acp-protocol.md).
- Pool semantics: [concepts/session-pool](../concepts/session-pool.md).
- Security choices around `env_clear()` + permission auto-reply:
  [architecture/security-model](../architecture/security-model.md).
- The dispatcher that wraps `session_prompt`: [modules/dispatch](dispatch.md).
