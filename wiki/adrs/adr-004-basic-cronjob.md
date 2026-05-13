---
title: "ADR-004 — Basic CronJob Support"
type: adr
sources:
  - ../docs/adr/basic-cronjob.md
  - ../docs/cronjob.md
  - ../docs/refarch/cronjob_k8s_refarch.md
related: [../modules/cron, ../deployment/reference-architectures]
updated: 2026-05-12
---

> **Status**: Proposed.
> **Date**: 2026-04-26 · **Author**: @chaodu-agent.

Phase 1 的內建定時器:operator 在 `config.toml` 用 `[[cron.jobs]]` 宣告 cron
表達式 + 頻道 + 訊息,OpenAB 每分鐘 tick 一次,符合就把訊息**當作有使用者輸
入**送進 agent。**不允許 agent 用自然語言設定自己的 cron**(借鏡 OpenClaw 的失
敗教訓:不透明、不可靠、不可預期)。複雜流程交給 K8s CronJob / GHA 等**外部**
scheduler。

Phase 1's in-process scheduler: operators declare `[[cron.jobs]]` in
`config.toml`; OpenAB ticks once per minute and sends each matching job
**as if a user typed it**. The agent **cannot** configure its own cron at
runtime — see Lessons from the OpenClaw in-app cronjob pattern.

---

## Design driver / 核心原則

> "If you already have a good landlord doing scheduling, you don't need
> your own alarm clock in the room."

OAB's scheduler is the **operator-controlled** option for the 80% case
("send this message at this time"). For anything more complex — claim an
item, retry, fan-out — use an external scheduler. See
[deployment/reference-architectures § K8s CronJob](../deployment/reference-architectures.md#2-k8s-cronjob-screening).

---

## Lessons from "agent configures its own cron" / OpenClaw 模式的問題

The OpenClaw "in-app cronjob" pattern (agent uses natural language to set
up cron) has known issues:

1. **Opacity** — operators can't easily inspect what the agent configured.
2. **Reliability** — continuous cron failures or too many in-flight jobs
   degrade the chat gateway.
3. **Unpredictability** — agent-generated payloads may not match user
   intent.

OpenAB explicitly rejects this pattern.

---

## Configuration shape / 設定形狀

```toml
[cron]
usercron_enabled = false                    # opt-in hot reload
usercron_path    = "cronjob.toml"           # external file path

[[cron.jobs]]
enabled     = true
schedule    = "0 9 * * 1-5"                 # cron expression
channel     = "123456789"                   # target channel/thread ID
message     = "summarize yesterday's PRs"   # prompt for the agent
platform    = "discord"                     # discord | slack
sender_name = "DailyOps"                    # attribution (default: openab-cron)
timezone    = "America/New_York"            # IANA timezone (default: UTC)
thread_id   = ""                            # post into existing thread (optional)
```

Two scopes:

- **Baseline `[cron.jobs]`** in `config.toml` — needs pod restart to
  change.
- **Usercron** — external `cronjob.toml`, hot-reloaded by file watcher.
  Must be **explicitly enabled** (`usercron_enabled = true`).

---

## Invariants / 不變式

1. **Operator-controlled, not agent-controlled.**
2. **No coupling between scheduler failures and gateway stability** —
   failed cron executions must not block or degrade normal chat traffic.
3. **Phase 1 only** — complex scheduling (conditional logic, retries,
   fan-out) belongs to external schedulers.

---

## Module page / 模組頁

Implementation summary: [modules/cron](../modules/cron.md).
