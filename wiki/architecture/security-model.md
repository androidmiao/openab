---
title: "Security Model / 安全模型"
type: architecture
sources:
  - ../DESIGN.md
  - ../AGENTS.md
  - ../src/acp/connection.rs
  - ../charts/openab/values.yaml
  - ../docs/adr/context-aware-token.md
related: [design-philosophy, system-overview, ../adrs/adr-006-context-aware-token]
updated: 2026-05-12
---

OpenAB 的安全模型是「**生產環境必須跑在 pod 裡**」+「agent subprocess 從乾淨環
境啟動」+「OAB 永遠不持有 platform-side 憑證以外的東西」。Sandbox 不是選項,是
**唯一**選項;`env_clear()` 與最小 allowlist 是預設,不是 opt-in。

OpenAB's security model is "**production must run in a sandbox**" + "agent
subprocesses start from a clean environment" + "the broker holds no
credentials beyond its own platform token". The sandbox is the only option,
not an opt-in.

---

## 1. Outbound-only / 純對外

The OpenAB pod **never opens an inbound port** in default deployments:

- Discord: outbound WS via serenity Gateway.
- Slack: outbound WS via Socket Mode.
- Webhook platforms (Telegram / LINE / Feishu / Teams / Google Chat): the
  inbound webhook lands on the standalone **gateway** pod; OpenAB connects
  outbound to the gateway over WS.

⇒ Zero externally reachable attack surface on the OpenAB pod itself.

⇒ OpenAB pod **完全沒有對外可達的 attack surface**。

See [system-overview](system-overview.md) and [adrs/adr-002-custom-gateway](../adrs/adr-002-custom-gateway.md).

---

## 2. `env_clear()` allowlist / 啟動 child 時清環境

From `../AGENTS.md` § Critical Rules § Security and the spawn code in
`../src/acp/connection.rs`:

> Agent subprocesses start with `env_clear()`. The baseline env passed to the
> child is:
> - All platforms: `HOME`, `PATH`
> - Unix only: `USER`
> - Windows only: `USERPROFILE`, `USERNAME`, `SystemRoot`, `SystemDrive`
> - Plus any explicit `[agent].env` keys (logged with a prompt-injection warning)
>
> Never leak `DISCORD_BOT_TOKEN` or other OAB credentials to the agent.

Why this matters: a compromised or jailbroken agent **must not** be able to
read OAB's bot tokens, gateway tokens, or unrelated K8s env vars by spelunking
the inherited process environment. The only way an env var reaches the agent
is if the operator explicitly listed it in `[agent].env` (recorded in audit
logs with a prompt-injection warning) or in `[agent].inherit_env` (a named
opt-in).

每個有意義被傳入 agent 的環境變數,都會被 log 一筆 prompt-injection 警告,讓
operator 能稽核。

---

## 3. Pod-level isolation / Pod 級隔離

`../charts/openab/values.yaml` defaults:

```yaml
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  seccompProfile:
    type: RuntimeDefault

containerSecurityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
```

- Non-root user, no privilege escalation.
- Root filesystem is read-only (`readOnlyRootFilesystem: true`).
- All Linux capabilities dropped.
- Seccomp profile is `RuntimeDefault`.

Storage:

- Writable state lives **only** on a per-agent PVC mounted at `HOME` (e.g.
  `~/.kiro/`, `~/.local/share/kiro-cli/`, `~/.openab/thread_map.json`).
- `helm.sh/resource-policy: keep` annotation on PVCs prevents accidental
  deletion of OAuth tokens on `helm uninstall` (`../AGENTS.md` § Helm Chart
  Checklist).

每個 agent **一個獨立的** Deployment / Secret / PVC,憑證之間**互相看不到**。

---

## 4. Credential separation / 憑證隔離

| Credential | Where it lives | Who sees it |
|---|---|---|
| `DISCORD_BOT_TOKEN` / `SLACK_BOT_TOKEN` / `SLACK_APP_TOKEN` | K8s Secret per agent | only the OpenAB process; **never** passed to the agent subprocess |
| Gateway shared token | K8s Secret on the OpenAB side, plus the gateway's Secret | OpenAB + gateway only |
| Platform-side webhook secrets (LINE channel secret, Telegram secret token, Feishu app secret, Teams JWT keys, Google Chat HMAC) | K8s Secret(s) on the **gateway** pod | the gateway only — never the OpenAB pod |
| Agent OAuth tokens (Kiro, Claude, Codex, Gemini, Copilot, OpenCode, Cursor) | PVC on the agent pod | the agent CLI only |
| `[agent].env` opt-ins (e.g. `ANTHROPIC_API_KEY`) | K8s Secret + explicit allowlist | the agent subprocess (logged with warning) |

The split between **OpenAB pod (broker credentials)** and **gateway pod
(platform credentials)** is the central architectural lever for limiting
blast radius. See [adrs/adr-002-custom-gateway](../adrs/adr-002-custom-gateway.md)
§ Credential Store Security Risks for the gateway's own concentration risk.

---

## 5. ACP permission auto-reply / 工具權限自動回應

OpenAB receives `session/request_permission` notifications from the agent
when it wants to call a tool. The default behaviour in `../src/acp/connection.rs`
(`build_permission_response` / `pick_best_option`):

- Prefer `allow_always` ≥ `allow_once` ≥ any non-reject option.
- Reject `reject_once` / `reject_always` options.
- If no selectable option exists, return `outcome: "cancelled"`.

This is **trust-on-first-use** behaviour, matching `kiro-cli`'s
`--trust-all-tools` semantics in default deployments. Agents that prefer
stricter gating should disable trust-all flags upstream.

Known caveat (from `../docs/adr/context-aware-token.md` § Prior Art): trust
heuristics derived from agent-supplied metadata are dangerous — see OpenClaw
advisory GHSA-7jx5-9fjg-hp4m. OpenAB intentionally keeps its auto-reply
heuristic naive (kind-based, not metadata-based) for this reason.

---

## 6. Local-dev escape hatch / 本機開發例外

Running `openab run -c config.toml` directly on a Mac/Linux/Windows host is
**allowed for local development and testing**. The security model assumes
pod-level isolation; without it the operator is responsible for the host's
isolation properties.

This is the **only** exception to the sandbox requirement, and it's intended
for `cargo run` debug loops — not production.

---

## 7. What's still unsolved / 還沒解決

`../docs/adr/context-aware-token.md` proposes a **short-lived, agent-scoped
token** so agents can do active platform operations (update thread title,
fetch historical messages, ping bots in other channels) **without** OpenAB
shipping platform tokens to the agent. Currently agents do this via
steering-doc `curl` hacks, which is exactly what the ADR aims to replace.

Status (per the ADR): Proposed. Implementation: not landed at the time of
this wiki bootstrap. See [adrs/adr-006-context-aware-token](../adrs/adr-006-context-aware-token.md).
