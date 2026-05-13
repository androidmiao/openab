---
title: "`config` module — TOML config + env expansion"
type: module
sources:
  - ../src/config.rs
  - ../config.toml.example
  - ../docs/config-reference.md
related: [../config/config-reference, ../concepts/messaging-model]
updated: 2026-05-12
---

`src/config.rs` (925 行) 定義整棵 `config.toml` 的型別與 serde 解析,負責
`${VAR}` 環境變數展開、policy 列舉(`AllowBots`、`AllowUsers`、
`MessageProcessingMode`)、以及所有 backward-compatible default 邏輯
(如 `allow_all_channels` 自動偵測)。

`src/config.rs` (925 LOC) defines the entire `config.toml` tree, handles
`${VAR}` env expansion, and encodes the policy enums (`AllowBots`,
`AllowUsers`, `MessageProcessingMode`). The full surface — every knob,
default, and example — lives in [config/config-reference](../config/config-reference.md);
this page is the module-level summary.

---

## Top-level shape / 頂層結構

```rust
pub struct Config {
    pub discord:   Option<DiscordConfig>,
    pub slack:     Option<SlackConfig>,
    pub gateway:   Option<GatewayConfig>,
    pub agent:     AgentConfig,
    #[serde(default)] pub pool:     PoolConfig,
    #[serde(default)] pub reactions: ReactionsConfig,
    #[serde(default)] pub stt:       SttConfig,
    #[serde(default)] pub markdown:  MarkdownConfig,
    #[serde(default)] pub cron:      CronConfig,
}
```

At least one of `[discord]` / `[slack]` / `[gateway]` is required. `[agent]`
is **always** required.

---

## Env-var expansion / 環境變數展開

`${VAR_NAME}` syntax. Implemented via regex in `config.rs`. Applied to every
string-valued field after TOML parse but before deserialisation into the
config types. Missing variables expand to the empty string (caller is
responsible for validation).

`bot_token = "${DISCORD_BOT_TOKEN}"` is the canonical pattern. Avoid hard-
coding secrets in config — see the security best-practice note in
`../docs/config-reference.md`.

---

## Policy enums / 政策列舉

### `MessageProcessingMode`

```
per-message  (default)  → 1 message = 1 ACP turn (v0.8.2-beta.1 behaviour)
per-thread              → all senders share one buffer per thread
per-lane                → one buffer per (thread, sender) — no silent drops
```

Drives the dispatcher (ADR-005 / [modules/dispatch](dispatch.md)).

### `AllowBots`

```
off       (default)  → ignore all bot messages
mentions             → only bot messages that @mention this bot
all                  → all bot messages, capped at max_bot_turns
```

Combined with `trusted_bot_ids` (list to further restrict).

### `AllowUsers`

```
involved        (default)  → reply in threads bot has participated in
mentions                   → always require @mention
multibot-mentions          → involved until a second bot posts; then mentions
```

These policy enums are the **primary** behavioural knobs for multi-bot
channels — see [concepts/multi-agent](../concepts/multi-agent.md).

---

## Backward-compatibility defaults / 向後相容預設

From `../AGENTS.md` § Critical Rules § Backward-Compatible Defaults:

> New config fields MUST default to the previous behavior. Never change what
> existing deployments experience without an explicit opt-in.

Two concrete idioms:

1. **`allow_all_channels: Option<bool>`** — if unset, auto-detect:
   non-empty `allowed_channels` → `false`; empty → `true`. Existing
   deployments that listed channels get the safer restriction; deployments
   without lists keep the open-access default.
2. **All `#[serde(default)]`** sub-tables — adding a new sub-table (e.g.
   `[stt]`, `[markdown]`, `[cron]`) does not break configs that omit it.

The Helm chart enforces the same discipline via Go-template `hasKey` checks
(see `../AGENTS.md` § Helm Chart Checklist).

---

## Sub-config types / 子設定

| Type | Purpose | Page |
|---|---|---|
| `DiscordConfig` | bot token, allowlists, bot-message gating, role IDs, DM, batching | [adapters/discord](../adapters/discord.md) |
| `SlackConfig` | bot/app tokens, allowlists, MPDM behaviour | [adapters/slack](../adapters/slack.md) |
| `GatewayConfig` | gateway WS URL, platform, token, bot username, allowlists | [adapters/gateway-client](../adapters/gateway-client.md) |
| `AgentConfig` | `command`, `args`, `working_dir`, `env`, `inherit_env` | [agents/agents](../agents/agents.md) |
| `PoolConfig` | `max_sessions`, `session_ttl_hours`, `prompt_hard_timeout_secs`, `liveness_check_secs` | [concepts/session-pool](../concepts/session-pool.md) |
| `ReactionsConfig` | enabled, remove-after-reply, emoji set, timing | [modules/reactions](reactions.md) |
| `SttConfig` | enabled, api_key, model, base_url, echo_transcript | [modules/media-stt § stt](media-stt.md) |
| `MarkdownConfig` | `tables = "code"  | "bullets" | "off"` | [modules/markdown-format](markdown-format.md) |
| `CronConfig` | `usercron_*` + `[[cron.jobs]]` | [modules/cron](cron.md) |

---

## Where to look in the code / 對應程式

| Concern | Symbol |
|---|---|
| Top-level parse + `${VAR}` expansion | `Config::load_from_str` / `expand_env_vars` |
| Auto-detect `allow_all_channels` / `allow_all_users` | `resolve_allow_all_*` helpers |
| `AllowBots`, `AllowUsers`, `MessageProcessingMode` deserializers | custom `Deserialize` impls in `config.rs` |
