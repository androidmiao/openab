# OpenAB — Agent Guidelines

You are contributing to OpenAB, a lightweight Rust-based ACP harness bridging Discord, Slack, and webhook platforms to coding CLIs over stdio JSON-RPC.

## Architecture

```
src/
├── main.rs           # entrypoint, multi-adapter startup, graceful shutdown
├── adapter.rs        # ChatAdapter trait, AdapterRouter (platform-agnostic)
├── config.rs         # TOML config + ${ENV_VAR} expansion
├── discord.rs        # DiscordAdapter (serenity EventHandler)
├── slack.rs          # SlackAdapter (Socket Mode)
├── gateway.rs        # Custom Gateway WebSocket adapter
├── cron.rs           # Config-driven + usercron scheduler
├── media.rs          # Image resize/compress + STT download
├── format.rs         # Message splitting, thread name shortening
├── markdown.rs       # Discord markdown rendering
├── reactions.rs      # Emoji status controller (debounce, stall detection)
├── bot_turns.rs      # Bot turn tracking
├── error_display.rs  # User-facing error formatting
├── stt.rs            # Speech-to-text (Whisper API)
├── setup/            # Interactive setup wizard
│   ├── config.rs     # Setup config helpers
│   ├── validate.rs   # Input validation
│   └── wizard.rs     # CLI setup flow
└── acp/
    ├── protocol.rs   # JSON-RPC types + ACP event classification
    ├── connection.rs # Spawn CLI, stdio JSON-RPC, env_clear whitelist
    └── pool.rs       # Session key → AcpConnection map + lifecycle
```

**Helm chart:** `charts/openab/` — Go templates, values.yaml, NOTES.txt
**Gateway:** `gateway/` — standalone Rust binary for Telegram/LINE/Teams
**Docs:** `docs/` — user-facing guides, ADRs, config reference

## Message Flow

```
Discord/Slack event
  → channel allowlist check
  → user allowlist check (bot messages bypass this)
  → bot message filter (off/mentions/all)
  → thread detection (thread_metadata = thread, NOT parent_id)
  → bot ownership check (bot_owns = thread creator matches bot user ID)
  → multibot detection (allowUserMessages mode)
  → handle_message → pool.get_or_create → ACP JSON-RPC
```

Understand this pipeline before modifying any adapter code.

## Critical Rules

### 1. Backward-Compatible Defaults

New config fields MUST default to the previous behavior. Never change what existing deployments experience without an explicit opt-in.

### 2. Thread Detection

The definitive rules — do NOT reinvent this:
- A channel is a thread if `thread_metadata` is present (NOT `parent_id`)
- `bot_owns` = thread's `owner_id` matches the bot's user ID
- Use `ChannelType` enum, not structural heuristics
- Forum posts are threads with `parent_id` pointing to a forum channel

### 3. Security — Child Process Environment

Agent subprocesses start with `env_clear()`. The baseline env passed to the child is:
- **All platforms:** `HOME`, `PATH`
- **Unix only:** `USER`
- **Windows only:** `USERPROFILE`, `USERNAME`, `SystemRoot`, `SystemDrive`
- Plus any explicit `[agent].env` keys (logged with a prompt-injection warning)

Never leak `DISCORD_BOT_TOKEN` or other OAB credentials to the agent.

### 4. Dockerfile Discipline

There are 7 Dockerfiles: `Dockerfile`, `Dockerfile.claude`, `Dockerfile.codex`, `Dockerfile.copilot`, `Dockerfile.cursor`, `Dockerfile.gemini`, `Dockerfile.opencode`.

A change to one MUST be evaluated against ALL. Common layers (base image, openab binary, tini) are shared — update all or explain why not.

### 5. Cross-Platform

Gate platform-specific code with `#[cfg(target_os = "...")]`. After adding Unix-only calls (libc, signals), verify:
```bash
cargo check --target x86_64-pc-windows-gnu
```

### 6. Discord API Limits

- Select menu: max 25 options (truncate with count, don't crash)
- Message content: max 2000 chars (use `format.rs` splitting)
- Rate limits: respect serenity's built-in ratelimiter

## Helm Chart Checklist

Before merging any chart change:

```bash
helm template test charts/openab
helm template test charts/openab --set agents.kiro.enabled=false
```

- Boolean fields: use `{{ if hasKey .Values "field" }}` not `{{ if .Values.field }}` (Go nil trap)
- Channel/user IDs: always `--set-string` (float64 precision loss)
- PVCs: set `"helm.sh/resource-policy": keep` — never delete user data on uninstall
- New values: add to `values.yaml` with comment + update `docs/config-reference.md`

## Breaking Changes

If your change alters existing behavior:

1. Add `!` to commit type: `feat(cron)!: description`
2. Include migration steps in PR body
3. Update relevant docs with migration callout
4. Ensure release note has ⚠️ Breaking Change section

## Build & Verify

```bash
cargo fmt
cargo clippy -- -D warnings
cargo test
```

All three must pass before pushing. For cross-platform changes, add:
```bash
cargo check --target x86_64-pc-windows-gnu
```

## PR Standards

- One logical change per PR
- Commit message: `type(scope): description` — types: `feat`, `fix`, `docs`, `refactor`, `test`, `ci`, `build`
- Include regression tests for bug fixes
- Reference issues: `Closes #123` or `Fixes #456`

## ADRs

Read `docs/adr/` before implementing features in these areas:
- Cron scheduler → `docs/adr/basic-cronjob.md`
- Custom Gateway → `docs/adr/custom-gateway.md`
- LINE adapter → `docs/adr/line-adapter.md`
- Multi-platform adapters → `docs/adr/multi-platform-adapters.md`

Write a new ADR if your feature introduces a new subsystem.

## Imported Claude Cowork project instructions
