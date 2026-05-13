---
title: "`setup` — Interactive Setup Wizard"
type: module
sources:
  - ../src/setup/mod.rs
  - ../src/setup/wizard.rs
  - ../src/setup/config.rs
  - ../src/setup/validate.rs
related: [config, ../config/config-reference]
updated: 2026-05-12
---

`src/setup/` 是 `openab setup` 子命令的實作:互動式 CLI 流程,引導使用者一步
一步產生 `config.toml`。`wizard.rs` (667 行)是主流程,`validate.rs` 做欄位驗
證,`config.rs` 提供寫檔輔助。

`src/setup/` implements the `openab setup` subcommand — an interactive CLI
flow that walks the user through producing a `config.toml`. `wizard.rs`
(667 LOC) is the main flow, `validate.rs` checks each field, `config.rs`
formats and writes the final file.

---

## Entry point / 進入點

```bash
openab setup           # writes to ./config.toml
openab setup -o my.toml
```

The flow is launched from `main.rs` via the `Commands::Setup` subcommand
(`../src/main.rs`).

---

## Flow / 流程

```
1. Pick adapter(s)      → Discord, Slack, Gateway, or any combo
2. For each adapter:
     - bot token        → secret entry via `rpassword`
     - allowed channels → comma-separated IDs
     - allowed users    → comma-separated IDs
     - advanced opts    → optional bot-message gating, role IDs, DM
3. Pick agent backend   → kiro-cli / claude-agent-acp / codex-acp / gemini / opencode / copilot / cursor
4. Show resolved [agent] block
5. Pick pool sizing     → max_sessions, session_ttl_hours
6. Pick reactions       → on/off + tool-display verbosity
7. Optional STT setup   → provider + API key
8. Optional cron setup  → 0 or N jobs
9. Write config.toml
10. Print next steps     → "now run openab run -c config.toml"
```

---

## Validation / 驗證

`validate.rs` enforces:

- **Discord channel IDs** are 17–20-digit snowflakes.
- **Slack channel IDs** start with `C` (channel) or `G` (private group).
- **Slack user IDs** start with `U`.
- **Cron expressions** parse with the `cron` crate.
- **IANA timezones** are accepted via `chrono-tz`.

Invalid inputs re-prompt rather than aborting.

---

## Secrets handling / 機敏資料

Tokens are read with `rpassword` (no terminal echo). The wizard writes them
into the generated config either as **literal strings** or wrapped with
`${VAR_NAME}` expansion — the user picks. The default is `${VAR}` form so
the wizard can produce a "templated" config that picks up secrets from the
environment at runtime.

---

## Cross-references / 交叉引用

- Full config surface: [config/config-reference](../config/config-reference.md).
- The wizard does **not** support every config knob — power users can edit
  the produced file by hand. Adding a knob to the wizard is opt-in.
