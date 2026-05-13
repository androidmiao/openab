# OpenAB Wiki — Log / 編輯紀錄

Chronological log of what happened to this wiki. New entries appended at the
bottom. Quick recent view: `grep '^## \[' log.md | tail -10`.

每筆編輯按時間順序記錄,新項目從底部追加。

---

## [2026-05-12] bootstrap | initial wiki creation

Created the wiki under `openab/wiki/` following the methodology in
[`../llm-wiki.md`](../llm-wiki.md). User requested bilingual (zh-TW + EN) at
medium depth (30–50 pages).

Bootstrap sweep covered:

- **Schema**: `CLAUDE.md` describes the three-layer model (raw sources / wiki /
  schema), the directory layout, language conventions, ingest/query/lint
  operations, source-of-truth precedence, and reference-anchor conventions.
- **Index**: `index.md` catalogues every page with one-line hooks, grouped by
  entry points, architecture, modules, adapters, agents, deployment,
  configuration, ADRs, and concepts.
- **Overview**: `overview.md` is the one-page project tour — what OpenAB is,
  what it isn't, the four design pillars, the two-tier adapter model.
- **Architecture pages**: system-overview, design-philosophy, message-flow,
  security-model, acp-protocol.
- **Module pages** (~11): one per Rust module in `src/`, plus a merged `acp`
  page covering `protocol.rs` + `connection.rs` + `pool.rs`.
- **Adapter pages** (9): Discord, Slack, gateway-client, gateway-service,
  Telegram, LINE, Feishu, Google Chat, Teams.
- **Agent backends**: a single comparison page covering Kiro, Claude Code,
  Codex, Gemini, OpenCode, Copilot, Cursor.
- **Deployment pages** (5): Helm chart, Docker images, k8s manifests,
  reference architectures, CI/CD.
- **Config reference**: one consolidated page mirroring `config.toml.example`.
- **ADR pages** (7): summaries of each ADR with status, decision, and
  consequences.
- **Concept pages** (5): session pool, thread detection, output directives,
  multi-agent collaboration, messaging model.

Sources consulted during bootstrap:

- `../README.md`, `../DESIGN.md`, `../AGENTS.md`, `../CONTRIBUTING.md`,
  `../RELEASING.md`
- `../Cargo.toml` (v0.8.3), `../gateway/Cargo.toml` (v0.4.0)
- `../src/main.rs`, `../src/adapter.rs`, `../src/acp/{protocol,connection,pool}.rs`,
  `../src/config.rs`, `../gateway/src/main.rs`, `../gateway/src/schema.rs`
- All seven ADRs under `../docs/adr/`
- `../docs/config-reference.md`, `../docs/multi-agent.md`,
  `../docs/messaging.md`, `../docs/cronjob.md`, and the per-platform/per-agent
  docs
- `../charts/openab/values.yaml`, `../charts/openab/README.md`
- The `../config.toml.example` skeleton

Outstanding follow-ups (deferred to later lint passes):

- Several wiki pages summarize ADRs whose status is "Proposed" — re-check
  whether those statuses have shifted (e.g. ADR-002 Custom Gateway, ADR-005
  Turn-Boundary Batching, ADR-006 Context-Aware Token).
- The Helm chart README references `agentsMd`, `secretEnv`, `extraInitContainers`,
  `extraContainers` — these warrant their own deployment-recipe page if usage
  patterns mature.
- The CI/CD page lists 22 workflows by name only; a full per-workflow page set
  is deferred unless someone needs it.

## [2026-05-13] ingest | cost-and-quota + observability concept pages

Added two new concept pages, distilling questions that came up during the
end-to-end install on 2026-05-13:

- **`concepts/cost-and-quota.md`** — Definitive answer to "if I leave the
  container running, will it burn tokens?" Walks through the three-layer
  cost profile (platform / broker / agent CLI), enumerates which of the
  seven inbound gates can absorb a message before any LLM call happens,
  and lists four silent-spend sources ranked by risk: open DM with empty
  `allowed_users`, over-aggressive cron, bot-message gating without
  `trusted_bot_ids`, and shared-channel exposure. Closes with a three-row
  hardening recipe (rectangle `allowed_users` / `allowed_channels` /
  `allow_dm = false`).
- **`concepts/observability.md`** — The three observability channels:
  `tracing` stdout logs, Docker `HEALTHCHECK`, and the reaction-emoji
  FSM on Discord messages. Documents that the broker's log timestamps
  are RFC3339 UTC (`Z` suffix), explains the `+8h` mental conversion for
  Asia/Taipei readers, provides an `awk`-based local-time pipe, and maps
  each reaction-FSM state to a debug interpretation (no 👀 → message
  never arrived; 🤔 stuck → agent stalled; etc.).

Both pages anchor against existing wiki concepts:
- 7-gate enumeration comes from `architecture/message-flow.md`.
- Reaction state machine references `modules/reactions.md`.
- Session-pool TTL claim references `concepts/session-pool.md`.
- Cron caveats reference `adrs/adr-004-basic-cronjob.md`.

Lessons that triggered these additions:

- **"Idle container = $0" is non-obvious to new operators.** The thin-bridge
  design philosophy implies it but doesn't state it. New page makes it
  explicit and ties it to the seven gates as the mechanism.
- **`allowed_users = []` + `allow_dm = true` is the biggest silent-spend
  trap.** A naive default deployment exposes the bot to any Discord user
  who can find its handle. Both new pages flag this as the #1 hardening
  step and link to the user-allowlist gate.
- **Log timezone is the #1 false-positive "is it broken?" trigger.** UTC
  is correct but routinely confuses users on +8 timezones. Observability
  page makes this canonical so future readers don't re-ask.
- **The reaction FSM is genuinely the cheapest triage signal.** No
  `docker compose logs` needed to know which layer is stuck — just look
  at the emoji on the triggering message. Surfaced as a triage cheat
  sheet in the observability page.

Both pages linked from `index.md` under the existing "Concepts" section.
No structural changes elsewhere; existing pages were not modified.

---

## [2026-05-13] ingest | Claude Code + Discord SOP (verified end-to-end)

Wrote `../deploy-claude-discord/SOP.md` — a complete runbook for installing
OpenAB + Claude Code + Discord on macOS Apple Silicon with Docker Desktop.
Procedure was verified end-to-end on this date with `openab-claude:latest`,
`@agentclientprotocol/claude-agent-acp@0.29.2`,
`@anthropic-ai/claude-code@2.1.124`.

Sections: prerequisites, Docker Desktop install (Homebrew path), Discord
Developer Portal walkthrough (including the two Privileged Gateway Intents
that are the #1 failure mode), Channel ID extraction, file configuration,
first-boot + `claude auth login` device flow, in-Discord verification,
troubleshooting tree (Bot offline / no reaction / stuck at 🤔 / config
not reloaded), day-2 ops (backup + restore of the OAuth volume, version
pinning), and advanced variations (multi-agent, repo mount, STT).

Lessons learned during the actual install (driving the troubleshooting tree):

- **DM vs channel confusion**: the user's first test was in a DM but
  `allow_dm = false`, so the message was dropped at gate ① with no log.
  Flagged in the troubleshooting table and in §4.4. Resolution: either
  set `allow_dm = true` or test in a server channel matching
  `allowed_channels`.
- **Channel ID as string, not number**: `allowed_channels = ["123..."]`
  with quotes preserved — otherwise TOML / float64 round-trip loses
  precision on the 18-19 digit IDs.
- **Privileged Intents are the #1 silent failure**: bot looks online,
  receives no message content. Both intents (Message Content + Server
  Members) must be enabled in the Developer Portal before the bot goes
  live.
- **Reaction FSM as observability**: the `👀 → 🤔 → 🔥 → 🆗` progression
  on the user's triggering message is the simplest "did the broker
  receive it / did Claude start / did Claude finish" indicator without
  reading logs.

Linked the SOP from `index.md` under a new "Runbooks / 安裝 SOP" section.

---

## [2026-05-12] lint | post-bootstrap link / orphan check

Ran an automated lint sweep over the freshly-created wiki:

- **48 wiki pages total**, all under `wiki/`.
- **Index coverage**: every page on disk is referenced from `index.md`
  (excluding index/log/CLAUDE themselves). No orphans.
- **Wiki-internal links**: zero genuine broken links. The three flagged
  hits are all inside fenced code blocks teaching markdown syntax
  (`[Page Title](path/to/page.md)` in CLAUDE.md, `[text](url)` example in
  `modules/markdown-format.md`).
- **Raw-source references (`../src/...`, `../docs/...`, `../charts/...`,
  `../gateway/...`)**: every relative path resolves to an existing file
  on disk. No stale code paths.

The wiki is internally consistent and ready for normal use. Next ingest /
query operations can append below this entry.
