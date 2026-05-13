---
title: "Output Directives — `[[reply_to:id]]`"
type: concept
sources:
  - ../src/adapter.rs
  - ../docs/output-directives.md
related: [../modules/adapter, ../concepts/multi-agent]
updated: 2026-05-12
---

Agent 在輸出 **開頭**用 `[[key:value]]` 行指示 adapter 做平台層動作。目前唯一
被解析的指令是 `[[reply_to:<message_id>]]`,讓 agent 主動指定要回覆哪一筆上游
訊息(Discord `message_reference`、Slack thread_ts 覆寫)。語法刻意嚴格:**只
有開頭連續的 `[[key:value]]` 行**會被當作指令;其他都是內容。

Agents emit `[[key:value]]` lines at the **very start** of their output to
tell the adapter to perform a platform-level action. The only directive
currently parsed is `[[reply_to:<message_id>]]` — it lets the agent pick
which specific message it's replying to (Discord `message_reference`, Slack
`thread_ts` override). The grammar is intentionally strict.

---

## Grammar / 文法

Implemented in `parse_output_directives` (`../src/adapter.rs:26-97`):

```
output         := directive_header? content
directive_header := (directive_line "\n")+
directive_line := "[[" key ":" value "]]"        (only at the very start)
key            := identifier (case-insensitive, currently just "reply_to")
value          := 1..64 chars from [A-Za-z0-9._-]
```

Rules:
- Only **consecutive** `[[key:value]]` lines at the very beginning count.
  Any other content stops the parser; everything after is content.
- Trailing text on the same line as a closing `]]` ends the directive header
  **but is preserved** as the first character of the content. This handles
  the case where an agent emits `[[reply_to:123]] Sure, here's the fix.`
  on one line.
- Invalid values (empty, > 64 chars, illegal chars) silently drop the
  directive — the line is treated as content.
- Unknown directive keys log at `debug!` and are stripped (don't appear
  in the final content).

---

## `[[reply_to:<id>]]`

Effect: the next outbound message is sent as a **reply** to the platform
message identified by `<id>`.

| Platform | Mapped to |
|---|---|
| Discord | `message_reference` (creates a quoted reply) |
| Slack | `thread_ts` override (reply lands in that thread) |
| Gateway | passed through in `GatewayReply.reply_to`-adjacent field |
| LINE | best-effort reply token lookup (see [adapters/line](../adapters/line.md)) |

---

## Why directives, not config / 為何用指令而不是設定

Two reasons:

1. **Per-message decisions, not per-deployment.** Reply-to context changes
   every turn; you can't pre-configure it.
2. **AI-Native** — keeping it in-band means the agent decides; the adapter
   doesn't need to be taught new rules. Adding a new directive is **agent-
   side** plus a tiny parser case on the broker.

---

## Use cases / 使用情境

The motivating use case (`../docs/output-directives.md`) is **multi-bot
channels**: when bot A is being addressed but bot B was the last to speak,
bot A can emit `[[reply_to:<bot_B_message_id>]]` to make clear which earlier
message it's responding to. This keeps multi-agent conversations
threadable on Discord.

It also helps when:

- An agent skipped over an intermediate human message because it was
  irrelevant — reply directly to the earlier relevant one.
- The agent is responding to a quoted message rather than the latest one.

---

## Cross-references / 交叉引用

- Parser implementation: [modules/adapter § output directives](../modules/adapter.md#output-directives--agent-端輸出指令).
- Motivating use case (multi-agent): [concepts/multi-agent](multi-agent.md).
