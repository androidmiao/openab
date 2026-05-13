---
title: "`markdown` & `format` — rendering and splitting"
type: module
sources:
  - ../src/markdown.rs
  - ../src/format.rs
related: [adapter, ../adapters/discord, ../adapters/slack]
updated: 2026-05-12
---

兩個小但關鍵的模組:**`markdown.rs`** 把 agent 產生的 CommonMark 翻譯成
Discord-flavoured markdown(包括三種 table 渲染模式);**`format.rs`** 處理訊
息切段(Discord 2000、Slack ~40k、LINE 5000)與 thread 名稱縮短。

Two small but load-bearing modules: `markdown.rs` translates CommonMark from
the agent into Discord-flavoured markdown (with three table modes);
`format.rs` handles per-platform message splitting and thread-name
shortening.

---

## `markdown.rs` (349 LOC)

Rendering uses `pulldown-cmark` to parse, then re-emits in a Discord-friendly
form. Key transformations:

- **Tables**: GFM tables are not natively supported by Discord; rendered
  according to `TableMode`:
  - `code` (default) — wrap in a fenced code block with aligned columns
    (monospace).
  - `bullets` — convert each row into bullets (`• Header: Value`).
  - `off` — pass through unchanged (visible markdown table syntax).
- **Code fences**: preserved verbatim including language tags.
- **Headings**: Discord supports `#` / `##` / `###`; deeper levels are
  rendered as **bold**.
- **Links**: `[text](url)` is preserved; auto-link `https://...` URLs are
  left alone so Discord embeds them.
- **Inline images**: stripped to a fallback URL — Discord does not inline
  images from message text; only attachments embed.

`MarkdownConfig`:

```toml
[markdown]
tables = "code"  # "code" | "bullets" | "off"
```

---

## `format.rs` (327 LOC)

Two responsibilities:

### 1. Message splitting / 訊息切段

Each platform has a different cap; `ChatAdapter::message_limit()` returns it.
Common values:

| Platform | Limit |
|---|--:|
| Discord | 2000 |
| Slack | ~40000 (mrkdwn text block) |
| LINE | 5000 |
| Telegram | 4096 |
| Feishu | 30000 (varies by msg type) |

`format::split_for_limit(content, limit)` returns a `Vec<String>` of chunks
that:

- Prefer to **break at line / paragraph boundaries**.
- **Never split inside a fenced code block** — code fences are re-opened
  in the next chunk with the same language tag.
- Preserve list markers across boundaries (re-emit the marker at the start
  of continuation chunks).

When the agent's stream exceeds the per-message limit, the broker
edit-streams chunk 1 in place and starts chunk 2 as a new follow-up.

### 2. Thread name shortening / Thread 名稱縮短

Discord thread names cap at 100 chars. `format::shorten_thread_name(prompt)`:

- Strips markdown.
- Trims to ≤ 100 chars.
- Drops trailing partial words.
- Adds an ellipsis (`…`) if truncation happened.

Used by the auto-thread-creation path on Discord/Slack.

---

## Why these are not in `adapter.rs` / 為何獨立成兩個小模組

Both are **pure, platform-agnostic** transformations and are heavily unit-
tested. Keeping them separate from `adapter.rs` (which does the orchestration)
makes the test surface obvious and lets the markdown renderer evolve
independently from the trait surface.
