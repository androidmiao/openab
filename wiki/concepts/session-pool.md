---
title: "Session Pool — one CLI per thread"
type: concept
sources:
  - ../src/acp/pool.rs
  - ../src/acp/connection.rs
related: [../architecture/acp-protocol, ../modules/acp]
updated: 2026-05-12
---

OpenAB 的 session pool 把「**一個 thread = 一個 agent CLI 子行程**」做成一級
primitive。`thread_key` (e.g. `"discord:1234567"`、`"slack:Cxxx:tsxxx"`、
`"gateway:line:Uxxx:_"`) 是唯一的對應鍵。Pool 同時支援:LRU eviction、
session 暫存後回復(`session/load`)、per-thread 序列化建立(避免重複 load
資料競爭)、與 pod 重啟後的 session ID 持久化。

OpenAB's session pool makes "**one thread → one agent CLI subprocess**" a
first-class primitive. The `thread_key` is the only mapping key. The pool
supports LRU eviction, suspended-session resume via `session/load`,
per-thread create-or-resume serialisation (preventing duplicate-load races),
and session-ID persistence across pod restarts.

---

## Why per-thread / 為何按 thread 切

The alternative — one agent process per user, or one global pool — fails
on two axes:

- **Context isolation** — two threads have different conversations. One
  agent process serving both would mix contexts.
- **Cost** — one process per **user** explodes for popular channels;
  threads are a natural conversation boundary.

Per-thread balances both: the agent sees a coherent conversation in each
process; users don't get private silos.

---

## `thread_key` construction / Key 構成

| Platform | Key shape |
|---|---|
| Discord | `discord:<channel_id_or_thread_id>` (DMs use the DM channel ID) |
| Slack | `slack:<channel>:<thread_ts>` (top-level message uses its own `ts`) |
| Gateway | `gateway:<platform>:<channel.id>:<thread_id_or_'_'>` |

The platform prefix prevents collisions between different platforms' ID
spaces (e.g. a Slack channel `C12345` vs a hypothetical Discord ID `12345`).

---

## Pool states / 池狀態

```rust
struct PoolState {
    active:          HashMap<String, Arc<Mutex<AcpConnection>>>,
    cancel_handles:  HashMap<String, (Arc<Mutex<ChildStdin>>, String)>,
    suspended:       HashMap<String, String>,   // thread_key → ACP sessionId
    creating:        HashMap<String, Arc<Mutex<()>>>,
}
```

| Set | Meaning |
|---|---|
| `active` | live `AcpConnection` per thread |
| `cancel_handles` | lock-free `/cancel` path (just needs stdin + session ID) |
| `suspended` | thread is dormant but its ACP `sessionId` is remembered for resume |
| `creating` | per-key gate that serialises concurrent create/resume requests |

The `creating` gate is the key to preventing **double `session/load`**:
two near-simultaneous user messages on the same thread would otherwise both
trigger resume.

---

## Lifecycle / 生命週期

```
First message on thread T:
  └─ creating[T].lock()
       active[T] absent + suspended[T] absent  → session/new
       active[T] absent + suspended[T] present → session/load
                                                  (move suspended→active)
       active[T] present                        → reuse

Subsequent messages on T:
  └─ pool.with_connection(T, |conn| conn.session_prompt(...))

LRU eviction (when active.len >= max_sessions):
  └─ pick least-recently-used active connection
     ├─ if supports_load_session: move to suspended (persist)
     └─ SIGTERM process group, drop from active

Pod restart:
  └─ load ~/.openab/thread_map.json → suspended
```

`max_sessions` defaults to 10; `session_ttl_hours` defaults to 24 (idle
eviction).

---

## Persistence / 持久化

`~/.openab/thread_map.json` stores `thread_key → ACP sessionId` for
suspended connections. Survives pod restart so users don't lose
conversation context when their pod gets rescheduled.

**Not** persisted: the agent's own conversation state (that lives in the
agent's own files — `~/.kiro/sessions/`, etc., on the PVC).

---

## Cancel paths / 取消路徑

Two flavours:

- **In-flight prompt cancel** (`session/cancel`) — uses `cancel_handles`
  to write the cancel request to the child's stdin **without** taking the
  per-connection mutex (the mutex is held by the in-flight `with_connection`
  call).
- **Full pool drain** on SIGTERM — `main.rs`'s shutdown signal handler
  triggers the pool to suspend all active connections and persist the map
  before exit.

---

## Cross-references / 交叉引用

- ACP wire-level: [architecture/acp-protocol](../architecture/acp-protocol.md).
- Module page: [modules/acp](../modules/acp.md).
- Config knobs: [config/config-reference § pool block](../config/config-reference.md#pool-block).
