---
title: "Multi-Agent Collaboration"
type: concept
sources:
  - ../docs/multi-agent.md
  - ../config.toml.example
  - ../src/config.rs
  - ../src/bot_turns.rs
related: [../architecture/design-philosophy, ../modules/bot-turns, ../adapters/discord, ../adapters/slack, output-directives, messaging-model]
updated: 2026-05-12
---

OpenAB 從第一天就為「同 channel 多個 agent」設計,但**從不替使用者決定如何協
作**。它只提供 primitives —`allow_bot_messages`、`trusted_bot_ids`、
`max_bot_turns`、`allow_user_messages` 的 `multibot-mentions` 模式,以及
agent 端的 `[[reply_to:]]` 指令——你可以用它們組出 sequential handoff、parallel
review、human-in-the-loop、agent-to-agent discussion 等多種拓撲。

OpenAB was designed from day one for multiple agents to coexist in one
channel — but it never picks the orchestration strategy. It exposes a small
set of primitives; you compose them into the topology you want.

---

## The primitives / 五個積木

| Primitive | Effect |
|---|---|
| `allow_bot_messages` (`off` / `mentions` / `all`) | gate whether bot-originated messages count at all |
| `trusted_bot_ids` | further restrict which bots count |
| `max_bot_turns` | hard cap on consecutive bot turns (default 100) |
| `allow_user_messages` (`involved` / `mentions` / `multibot-mentions`) | when does the bot need an @mention to reply |
| `[[reply_to:<msg_id>]]` directive | agent picks which message it's responding to |

Each agent runs in its own pod; the primitives are **per-agent** config —
agent A can be open to bot-talk while agent B isn't.

---

## Topologies / 常見拓撲

### Sequential handoff / 依序接棒

```
user → review-bot → "this PR looks good, [[reply_to:user_msg]] @deploy-bot please deploy"
                  → deploy-bot picks it up because @mention-gated to it
```

Setup:
- both bots `allow_bot_messages = "mentions"` and add each other to
  `trusted_bot_ids`.

### Parallel collaboration / 平行作答

```
user @analyzer-bot @critic-bot "what's wrong with this code"
  └─ analyzer-bot replies
  └─ critic-bot replies (independently, same prompt)
```

Each bot is mentioned directly; no bot-to-bot routing.

### Human-in-the-loop / 人為仲裁

```
agents propose ──► human reviews ──► human decides ──► one agent acts
```

Set bots to `allow_bot_messages = "off"` for the proposal phase, then
human triggers the action with a direct @mention.

### Agent-to-agent discussion (A2A) / Agent 互相討論

```
agent-A proposes → agent-B critiques → agent-A revises → agent-B accepts
```

`allow_bot_messages = "all"` + a sensible `max_bot_turns` cap (the
[modules/bot-turns](../modules/bot-turns.md) safety net catches runaway
loops).

---

## Slack `multibot-mentions` / Slack 的混合模式

A practical sub-case from `allow_user_messages`:

- One bot in the thread → `involved` semantics (no @mention needed for
  follow-ups).
- Second bot posts → automatically upgrades to `mentions` semantics
  (every follow-up needs @mention to disambiguate).

This works well for "I usually want quick follow-ups, but if a co-pilot
joins, I'll @-pin who I want."

---

## What OpenAB does **not** do / 刻意不做

From `../DESIGN.md`'s "What OpenAB Is Not":

- ❌ Decide which agent handles which message.
- ❌ Provide a memory layer for cross-agent context sharing.
- ❌ Translate between agents' tool schemas.
- ❌ Manage an A2A "judge" that picks a winner.

All of those are **above** the broker — they live in steering documents,
agent prompts, or external orchestration tools.

---

## Helm layout / Helm 對映

Each agent is a separate Deployment in the chart:

```yaml
agents:
  reviewer:
    enabled: true
    discord:
      botToken: "..."
      allowBotMessages: "mentions"
      trustedBotIds: ["<deployer_bot_id>"]
  deployer:
    enabled: true
    discord:
      botToken: "..."
      allowBotMessages: "mentions"
      trustedBotIds: ["<reviewer_bot_id>"]
```

See [deployment/helm-chart](../deployment/helm-chart.md) and
`../docs/multi-agent.md`.

---

## Cross-references / 交叉引用

- Design pillar #2: [architecture/design-philosophy § Multi-Bot Ready](../architecture/design-philosophy.md#2-multi-bot-ready--多-bot-共存).
- Loop guard: [modules/bot-turns](../modules/bot-turns.md).
- The `[[reply_to:]]` directive used by agents to thread their replies:
  [concepts/output-directives](output-directives.md).
- The five messaging patterns: [concepts/messaging-model](messaging-model.md).
