---
title: "Configuration Reference — `config.toml`"
type: config
sources:
  - ../config.toml.example
  - ../docs/config-reference.md
  - ../src/config.rs
related: [../modules/config, ../adapters/discord, ../adapters/slack, ../adapters/gateway-client, ../modules/cron, ../modules/reactions, ../modules/media-stt]
updated: 2026-05-12
---

整棵 `config.toml` 的權威參考。所有欄位 → 預設值 → 對應的 Helm value。**至少一
個 adapter section(`[discord]`、`[slack]`、`[gateway]`)是必要的**;`[agent]`
**永遠**必要。完整的 RFC-grade reference 在 `../docs/config-reference.md`,本
頁是 wiki 整合版。

The authoritative reference for `config.toml`. Every field → default → its
Helm-values mapping. **At least one adapter section** (`[discord]`,
`[slack]`, `[gateway]`) is required; `[agent]` is **always** required. The
RFC-grade reference lives in `../docs/config-reference.md`; this page is the
wiki-aggregated view.

---

## Loading / 載入

```bash
openab run -c config.toml                     # local
openab run -c https://example.com/config.toml # remote (HTTPS recommended)
openab run -c http://internal/config.toml     # warned — config holds secrets
```

Remote: HTTP GET, 10-second timeout, 1 MiB response size cap. `${VAR}`
expansion works identically on local and remote content.

---

## `[discord]` block

```toml
[discord]
bot_token              = "${DISCORD_BOT_TOKEN}"
allow_all_channels     = ?     # bool | omitted → auto-detect from list
allowed_channels       = []    # string[]
allow_all_users        = ?     # bool | omitted → auto-detect from list
allowed_users          = []    # string[]
allow_bot_messages     = "off" # off | mentions | all
trusted_bot_ids        = []    # string[]
allow_user_messages    = "involved"      # involved | mentions
max_bot_turns          = 100   # u32
allowed_role_ids       = []    # string[]
allow_dm               = false # bool
message_processing_mode = "per-message"  # per-message | per-thread | per-lane
max_buffered_messages  = 10    # usize
max_batch_tokens       = 24000 # usize
```

Auto-detect logic for `allow_all_channels` / `allow_all_users`: when
**omitted**, non-empty list → `false`, empty list → `true`. Explicit values
always win.

Adapter page: [adapters/discord](../adapters/discord.md).

## `[slack]` block

```toml
[slack]
bot_token            = "${SLACK_BOT_TOKEN}"     # xoxb-...
app_token            = "${SLACK_APP_TOKEN}"     # xapp-... (Socket Mode)
allow_all_channels   = ?  # same auto-detect as discord
allowed_channels     = []
allow_all_users      = ?
allowed_users        = []
allow_bot_messages   = "off"      # off | mentions | all
trusted_bot_ids      = []
allow_user_messages  = "involved" # involved | mentions | multibot-mentions
max_bot_turns        = 100
```

Adapter page: [adapters/slack](../adapters/slack.md). The
`multibot-mentions` policy is Slack-only.

## `[gateway]` block

```toml
[gateway]
url               = "ws://openab-gateway:8080/ws"
platform          = "telegram"   # telegram | line | feishu | googlechat | teams
token             = "${GATEWAY_TOKEN}"     # WS-level auth
bot_username      = "my_bot"               # for @mention gating
allow_all_channels = ?
allowed_channels  = []
allow_all_users   = ?
allowed_users     = []
```

Adapter pages: [adapters/gateway-client](../adapters/gateway-client.md),
[adapters/gateway-service](../adapters/gateway-service.md), and the per-
platform pages.

## `[agent]` block

```toml
[agent]
command      = "kiro-cli"
args         = ["acp", "--trust-all-tools"]
working_dir  = "/home/agent"
env          = {}                       # ⚠️ visible to the agent
inherit_env  = []                       # named opt-ins for K8s envFrom vars
```

See [agents/agents](../agents/agents.md) for the seven supported backends.

## `[pool]` block

```toml
[pool]
max_sessions            = 10
session_ttl_hours       = 24
prompt_hard_timeout_secs = 1800   # 30 min hard ceiling per prompt (#732)
liveness_check_secs     = 30      # recv-loop liveness cadence
```

See [concepts/session-pool](../concepts/session-pool.md).

## `[reactions]` block

```toml
[reactions]
enabled            = true
remove_after_reply = false

[reactions.emojis]
queued   = "👀"
thinking = "🤔"
tool     = "🔥"
coding   = "👨💻"
web      = "⚡"
done     = "🆗"
error    = "😱"

[reactions.timing]
debounce_ms     = 700
stall_soft_ms   = 10000
stall_hard_ms   = 30000
done_hold_ms    = 1500
error_hold_ms   = 2500
```

See [modules/reactions](../modules/reactions.md).

## `[stt]` block

```toml
[stt]
enabled         = false
api_key         = "${STT_API_KEY}"
model           = "whisper-large-v3-turbo"
base_url        = "https://api.groq.com/openai/v1"
echo_transcript = false
```

See [modules/media-stt § stt](../modules/media-stt.md#sttrs).

## `[markdown]` block

```toml
[markdown]
tables = "code"   # code | bullets | off
```

See [modules/markdown-format](../modules/markdown-format.md).

## `[cron]` block

```toml
[cron]
usercron_enabled = false                # must be explicit true to enable hot-reload
usercron_path    = "cronjob.toml"       # external file, relative to $HOME/.openab

[[cron.jobs]]
enabled     = true
schedule    = "0 9 * * 1-5"
channel     = "123456789"
message     = "summarize yesterday's merged PRs"
platform    = "discord"
sender_name = "DailyOps"
timezone    = "America/New_York"
thread_id   = ""                        # optional, post into existing thread
```

See [modules/cron](../modules/cron.md) and
[adrs/adr-004-basic-cronjob](../adrs/adr-004-basic-cronjob.md).

---

## Helm values mapping / Helm 對映

Every `config.toml` field has a `agents.<name>.<field>` counterpart in the
Helm chart. The chart's README (`../charts/openab/README.md`) has the full
table. Highlights:

| `config.toml` | Helm value |
|---|---|
| `[discord].bot_token` | `agents.<name>.discord.botToken` |
| `[discord].allowed_channels` | `agents.<name>.discord.allowedChannels` |
| `[slack].bot_token` | `agents.<name>.slack.botToken` |
| `[agent].command` | `agents.<name>.command` |
| `[pool].max_sessions` | `agents.<name>.pool.maxSessions` |
| `[reactions].enabled` | `agents.<name>.reactions.enabled` |
| `[stt].enabled` | `agents.<name>.stt.enabled` |
| `[[cron.jobs]]` | `agents.<name>.cronjobs[]` |

⚠️ Always use `--set-string` for channel/user IDs — see
[deployment/helm-chart § Go-template gotchas](../deployment/helm-chart.md#go-template-gotchas--helm-chart-必踩坑).

---

## Cross-references / 交叉引用

- Source-of-truth code: [modules/config](../modules/config.md).
- Example file: `../config.toml.example`.
- Rigorous reference: `../docs/config-reference.md`.
