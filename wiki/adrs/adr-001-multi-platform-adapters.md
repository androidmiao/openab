---
title: "ADR-001 — Multi-Platform Adapter Architecture"
type: adr
sources:
  - ../docs/adr/multi-platform-adapters.md
related: [../modules/adapter, ../adapters/discord, ../adapters/slack, adr-002-custom-gateway]
updated: 2026-05-12
---

> **Status**: Partially Implemented — Phase 1 + 3 landed via #259
> (Slack adapter).
> **Date**: 2026-04-06 · **Author**: @chaodu-agent
> **Tracking**: issues #86 (Telegram), #93 (Slack).

把 OpenAB 從「只能跑 Discord」改成「同一個 process 同時跑 Discord + Slack
(+ 將來其他平台)」的設計案。決議:**Contract B — 同 process 同時多平台**,
共用一個 `SessionPool`,session key 內含平台前綴。Contract A(一個部署一個平
台)自動成立(設定檔只放一個 platform section 就行)。

Defines the platform-agnostic `ChatAdapter` trait so OpenAB can host
Discord + Slack (+ future adapters) in **one process** with a shared
`SessionPool`. Contract A (single-platform per deployment) is a free subset
of Contract B (multi-platform simultaneously).

---

## Decision / 決議

Adopt a `ChatAdapter` trait and an `AdapterRouter` shim. Session keys are
prefixed with the platform name so two platforms' threads can never
collide.

```rust
#[async_trait]
pub trait ChatAdapter: Send + Sync + 'static {
    fn platform(&self) -> &'static str;
    fn message_limit(&self) -> usize;
    async fn run(&self, router: Arc<AdapterRouter>) -> Result<()>;
    async fn send_message(...) -> Result<MessageRef>;
    async fn edit_message(...) -> Result<()>;
    async fn create_thread(...) -> Result<ChannelRef>;
    async fn add_reaction(...) -> Result<()>;
    async fn remove_reaction(...) -> Result<()>;
}
```

Implementations:
- `DiscordAdapter` (`../src/discord.rs`)
- `SlackAdapter` (`../src/slack.rs`)
- `GatewayAdapter` (`../src/gateway.rs`)

---

## Motivation / 動機

- OpenAB was originally hard-wired to Discord via `serenity`.
- Telegram (#86) and Slack (#93) both needed similar abstractions.
- Without a shared trait each adapter would duplicate session routing,
  message splitting, reaction handling, and streaming logic.
- A clean trait boundary enables **simultaneous multi-adapter** runs
  (Discord + Slack in one process, validated by #259).

---

## Architecture / 架構

```
                ┌─────────────────┐
                │  Discord Users  │
                └────────┬────────┘
                         │ Gateway WS (serenity)
                         ▼
                ┌─────────────────┐  impl ChatAdapter
                │ DiscordAdapter  │──┐
                └─────────────────┘  │
                                     │     ┌─────────────────┐
┌─────────────────┐                  ▼     │   AdapterRouter │     ┌──────────────┐
│  Slack Users    │            ┌─────────────────┐           │────►│ SessionPool  │
└────────┬────────┘            │  AdapterRouter  │           │     │              │
         │ Socket Mode (WS)    │   - route msg   │           │     └──────┬───────┘
         ▼                      │   - manage thr  │           │            │ ACP stdio
┌─────────────────┐             │   - stream edits│           │     ┌──────▼───────┐
│  SlackAdapter   │────────────►└─────────────────┘           │     │ AcpConnection│
└─────────────────┘                                            │     └──────────────┘
                                ┌─────────────────┐            │
                                │ TelegramAdapter │            │  (future / now via gateway)
                                └─────────────────┘
```

---

## Consequences / 後果

Positive:
- New platforms become **drop-in implementations** — see Slack landing in
  one PR.
- Per-platform tests don't have to mock the rest of OpenAB; just mock the
  trait.

Negative:
- Trait methods accumulate as platforms diverge (e.g. reactions on Telegram
  but not LINE → `add_reaction` no-ops on LINE).
- The trait can't fully hide every platform quirk; `discord.rs` is still
  the largest file at 2308 LOC because of Discord-specific concerns
  (privileged intents, forum channels, role mentions, DM allow-listing).

---

## Status today / 目前狀態

- Discord, Slack: native adapters using the trait.
- Telegram, LINE, Feishu, Teams, Google Chat: handled by **one**
  `GatewayAdapter` (`src/gateway.rs`) talking to the standalone gateway —
  see [ADR-002](adr-002-custom-gateway.md).
- The "Telegram in OAB" line in the ADR's architecture diagram is **not**
  the current direction; the modern path is the gateway.

---

## Related / 相關

- The Custom Gateway successor for non-native platforms: [ADR-002](adr-002-custom-gateway.md).
- Module page for the trait: [modules/adapter](../modules/adapter.md).
- Discord-specific concerns: [adapters/discord](../adapters/discord.md).
- Slack-specific concerns: [adapters/slack](../adapters/slack.md).
