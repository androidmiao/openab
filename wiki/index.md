# OpenAB Wiki — Index / 索引

The content-oriented catalog of this wiki. The schema (how this wiki is
maintained) lives in [`CLAUDE.md`](CLAUDE.md). The timeline of edits lives in
[`log.md`](log.md). Start with [`overview.md`](overview.md) for a tour.

本 wiki 的內容索引。維護規則見 [`CLAUDE.md`](CLAUDE.md);編輯紀錄見
[`log.md`](log.md);專案巡禮見 [`overview.md`](overview.md)。

---

## Project entry points / 專案入口

- [Overview / 專案概覽](overview.md) — one-page tour of OpenAB: what it is, what it isn't, the four pillars.

## Architecture / 架構

- [System Overview / 系統總覽](architecture/system-overview.md) — the two-tier adapter model + ACP bridge + session pool layout.
- [Design Philosophy / 設計哲學](architecture/design-philosophy.md) — Thin Bridge, Multi-Bot Ready, AI-Native, Security by Design.
- [Message Flow / 訊息流](architecture/message-flow.md) — from inbound event to ACP turn to outbound reply.
- [Security Model / 安全模型](architecture/security-model.md) — `env_clear()`, pod isolation, no inbound ports, credential separation.
- [ACP Protocol / ACP 協定](architecture/acp-protocol.md) — JSON-RPC over stdio, session lifecycle, classify_notification.

## Rust Modules / Rust 模組

- [`adapter` — ChatAdapter trait](modules/adapter.md) — platform-agnostic adapter abstraction, `AdapterRouter`, output directives.
- [`acp` — ACP subsystem](modules/acp.md) — `protocol.rs`, `connection.rs`, `pool.rs`; ACP types, JSON-RPC plumbing, session pool.
- [`config` — TOML config](modules/config.md) — full config tree, `${VAR}` expansion, `MessageProcessingMode`, `AllowBots`, `AllowUsers`.
- [`dispatch` — Turn-boundary batching](modules/dispatch.md) — per-thread/lane buffer that batches messages arriving during an in-flight turn.
- [`cron` — Scheduler](modules/cron.md) — config-driven cron jobs + optional usercron hot-reload.
- [`reactions` — Status emoji controller](modules/reactions.md) — 👀→🤔→🔥→🆗 debounce/stall state machine.
- [`markdown` & `format` — Rendering](modules/markdown-format.md) — Discord-flavoured markdown, message splitting, thread name shortening.
- [`media` & `stt` — Attachments & STT](modules/media-stt.md) — image resize/compress + Whisper-compatible speech-to-text.
- [`bot_turns` — Bot loop guard](modules/bot-turns.md) — counts consecutive bot turns per thread for `max_bot_turns`.
- [`setup` — Interactive setup wizard](modules/setup-wizard.md) — CLI `openab setup` that generates `config.toml`.
- [`error_display` — User-facing errors](modules/error-display.md) — formats agent errors for chat clients.

## Platform Adapters / 平台適配器

- [Discord adapter (native)](adapters/discord.md) — serenity Gateway, thread detection, `bot_owns` logic.
- [Slack adapter (native)](adapters/slack.md) — Socket Mode, thread_ts handling, MPDM behaviour.
- [Gateway client](adapters/gateway-client.md) — OAB-side `src/gateway.rs` WebSocket client.
- [Gateway service (standalone)](adapters/gateway-service.md) — the `gateway/` binary that fans webhooks in to OAB.
- [Telegram](adapters/telegram.md) — webhook → gateway, no threads.
- [LINE](adapters/line.md) — webhook + reply-token cache, 1:1 vs group trade-offs.
- [Feishu / Lark](adapters/feishu.md) — long-connection WebSocket + webhook fallback, encrypted payloads.
- [Google Chat](adapters/googlechat.md) — webhook, threading via `thread.name`.
- [MS Teams](adapters/teams.md) — Bot Framework + service_url cache.

## Agent Backends / AI 代理後端

- [Agent backends](agents/agents.md) — comparison + per-agent setup notes for Kiro CLI, Claude Code, Codex, Gemini, OpenCode, Copilot CLI, Cursor.

## Deployment / 部署

- [Helm chart](deployment/helm-chart.md) — `charts/openab/` structure, per-agent resources, Go-template gotchas.
- [Docker images](deployment/docker-images.md) — the seven Dockerfiles (default + 6 agent variants).
- [Kubernetes manifests](deployment/k8s-manifests.md) — `k8s/` plain manifests for non-Helm deployments.
- [Reference architectures](deployment/reference-architectures.md) — ECS Fargate Spot, K8s CronJob, GBrain, remote SSH debugging.
- [CI / CD](deployment/ci-cd.md) — the 22 GitHub Actions workflows: build, release, issue triage, stale-issue automation.

## Runbooks / 安裝 SOP

- [SOP: Claude Code + Discord on macOS Docker](../deploy-claude-discord/SOP.md) — end-to-end verified procedure: install Docker Desktop → create Discord bot → configure files → `claude auth login` → verify in Discord; with troubleshooting tree.

## Configuration / 設定

- [Config reference](config/config-reference.md) — full `config.toml` schema with defaults, including `[discord]`, `[slack]`, `[gateway]`, `[agent]`, `[pool]`, `[reactions]`, `[stt]`, `[markdown]`, `[cron]`.

## ADRs / 架構決策紀錄

- [ADR-001: Multi-Platform Adapters](adrs/adr-001-multi-platform-adapters.md) — `ChatAdapter` trait, shared `SessionPool`, simultaneous multi-platform contract.
- [ADR-002: Custom Gateway](adrs/adr-002-custom-gateway.md) — standalone service for webhook platforms, OAB stays outbound-only.
- [ADR-003: LINE Adapter](adrs/adr-003-line-adapter.md) — original in-OAB LINE adapter (superseded for v2 by ADR-002).
- [ADR-004: Basic CronJob](adrs/adr-004-basic-cronjob.md) — config-driven scheduled messages, decoupled from agent.
- [ADR-005: Turn-Boundary Batching](adrs/adr-005-turn-boundary-batching.md) — per-thread bounded `mpsc` queue, N-messages → 1-turn.
- [ADR-006: Context-Aware Token](adrs/adr-006-context-aware-token.md) — short-lived agent-scoped token for active platform operations.
- [ADR-007: PR Contribution Guidelines](adrs/adr-007-pr-contribution-guidelines.md) — required PR sections + tiered prior-art research.

## Concepts / 概念

- [Session Pool](concepts/session-pool.md) — one CLI process per thread, eviction, suspension/resume via `session/load`.
- [Thread Detection](concepts/thread-detection.md) — definitive rules; `thread_metadata` not `parent_id`; `bot_owns`.
- [Output Directives](concepts/output-directives.md) — `[[reply_to:id]]` and the directive header grammar.
- [Multi-Agent Collaboration](concepts/multi-agent.md) — `allow_bot_messages`, `trusted_bot_ids`, bot-turn caps, A2A patterns.
- [Messaging Model](concepts/messaging-model.md) — the five messaging patterns: DM, channel, thread, multi-bot thread, bot-to-bot.
- [Cost & Quota](concepts/cost-and-quota.md) — when OpenAB actually consumes LLM tokens, silent spend sources, hardening recipe.
- [Observability](concepts/observability.md) — `tracing` logs (UTC), Docker `HEALTHCHECK`, and the reaction-emoji FSM as a triage tool.
