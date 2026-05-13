---
title: "`cron` module — Scheduled Messages"
type: module
sources:
  - ../src/cron.rs
  - ../docs/cronjob.md
  - ../docs/adr/basic-cronjob.md
  - ../docs/refarch/cronjob_k8s_refarch.md
related: [config, ../adrs/adr-004-basic-cronjob, ../deployment/reference-architectures]
updated: 2026-05-12
---

`src/cron.rs` (1272 行) 實作 ADR-004:**config-driven** 定時訊息調度器。
operator 在 `config.toml` 用 `[[cron.jobs]]` 宣告 cron 表達式 + 頻道 + 訊息,
OpenAB 內部每分鐘評估一次,符合就把訊息**當作有使用者輸入**塞進 agent。可選
開啟 `usercron_*` 來支援外部 `cronjob.toml` 熱重載。

`cron.rs` (1272 LOC) implements ADR-004. Operators declare `[[cron.jobs]]` in
`config.toml`; OpenAB evaluates cron expressions once per minute and sends
each matching job as if a user typed it. Optional `usercron_*` enables
hot-reload of an external `cronjob.toml`.

---

## Two scopes / 兩種來源

| Source | Where it lives | Hot-reload? |
|---|---|---|
| `[cron.jobs]` baseline jobs | `config.toml` (with the rest of config) | no — needs pod restart |
| `usercron` jobs | external `cronjob.toml` whose path is `cron.usercron_path` | yes — file watcher reloads on change |

`usercron_enabled` must be **explicitly** set to `true` (default `false`),
matching the "backward-compatible default" rule.

---

## Job shape / Job 結構

From `../docs/cronjob.md`:

```toml
[[cron.jobs]]
enabled    = true                          # optional, default: true
schedule   = "0 9 * * 1-5"                 # cron expression
channel    = "123456789"                   # target channel/thread ID
message    = "summarize yesterday's PRs"   # prompt sent to the agent
platform   = "discord"                     # "discord" or "slack"
sender_name = "DailyOps"                   # attribution (default: "openab-cron")
timezone   = "America/New_York"            # IANA timezone (default: "UTC")
thread_id  = ""                            # optional: post into an existing thread
```

The job is executed **as if a user typed `message`** in `channel`. It goes
through the dispatcher and the ACP pool the same way real user messages do.

---

## Two timezones / 兩種時區

Cron expressions are interpreted in either UTC (default) or an IANA zone via
`timezone`. The scheduler uses `chrono-tz` for the conversion. The
once-per-minute tick is anchored on the OS clock; DST transitions are
handled by `chrono-tz`.

---

## What this is NOT / 不是什麼

ADR-004 explicitly **does not** support:

- **In-app cronjob** pattern (agent configures its own cron via natural
  language) — the OpenClaw pattern with known opacity / reliability /
  unpredictability issues.
- **Conditional or retry logic** — that belongs to an external scheduler
  (K8s CronJob, GitHub Actions cron). See `../docs/refarch/cronjob_k8s_refarch.md`
  for the recommended K8s pattern.
- **Agent-modifiable cron entries at runtime** — explicitly out of scope.

The principle from ADR-004:

> If you already have a good landlord doing scheduling, you don't need your
> own alarm clock in the room.

---

## Failure isolation / 失敗不影響本職

ADR-004 § 1 invariant:

> No coupling between scheduler failures and gateway stability — failed cron
> executions must not block or degrade normal chat traffic.

The cron loop runs on its own task; failures (bad expression, missing
channel, agent error) log and continue. Cron does not hold any locks that
the chat path needs.

---

## Cross-references / 交叉引用

- ADR: [adrs/adr-004-basic-cronjob](../adrs/adr-004-basic-cronjob.md).
- For complex / external scheduling: [K8s CronJob refarch](../deployment/reference-architectures.md).
- The reminder / `@later` subsystem (different from cron) lives in
  `../src/remind.rs` and is summarised in [`remind` notes below](#sibling-remind-module--親戚-remind).

---

## Sibling: `remind` module / 親戚:`remind` 模組

`src/remind.rs` (399 行) is a related but distinct subsystem: **one-shot
reminders** created from inside a conversation (e.g. agent says "remind me
in 1 hour"). It does not use cron expressions and is not config-driven;
state is per-thread and lives only in memory.

The two share a tick loop concept but are otherwise independent.
