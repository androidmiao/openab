---
title: "`error_display` — User-Facing Error Formatting"
type: module
sources:
  - ../src/error_display.rs
related: [adapter, acp]
updated: 2026-05-12
---

`src/error_display.rs` (219 行) 把 agent / 平台 / 配置層產生的錯誤翻譯成
**對使用者友善**的 chat 訊息。錯誤分兩類:**coded error**(`format_coded_error`)
與 **user error**(`format_user_error`),前者保留代碼以便 oncall 查找,後者
完全人話。

`error_display.rs` (219 LOC) translates errors from the agent / platform /
config layers into user-friendly chat messages. Two formatters:
`format_coded_error` (keeps an error code for support / on-call lookup) and
`format_user_error` (purely human-readable).

---

## Where errors come from / 錯誤來源

| Source | Example |
|---|---|
| ACP agent | `JsonRpcError { code: -32603, message: "session not found" }` |
| Platform | `serenity::Error::Http(...)`, Slack rate-limit, gateway WS drop |
| Broker | bad config, pool full, missing OAuth token |
| Cron | invalid expression, channel not found |

Each callsite uses one of:

- `format_coded_error(code, message)` → `"❌ [E-1234] couldn't reach the agent (timeout). Please try again."`
- `format_user_error(message)` → `"❌ The thread isn't reachable anymore. Please open a new thread."`

---

## Style / 風格

- One short paragraph maximum.
- Lead with a clear emoji (❌ / ⚠️ / ⏳).
- **No stack traces** — those go to `tracing` at `error!` level for
  operators.
- Include actionable text where possible ("try again in a minute", "open a
  new thread", "check the bot has permission in this channel").

---

## Why a dedicated module / 為何獨立模組

Centralising error wording lets us keep tone consistent across platforms and
makes localisation / wording tweaks a single-file change. It also keeps
adapter code from accumulating `format!`-strings inline.
