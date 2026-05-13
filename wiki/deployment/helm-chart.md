---
title: "Helm Chart — `charts/openab/`"
type: deployment
sources:
  - ../charts/openab/Chart.yaml
  - ../charts/openab/values.yaml
  - ../charts/openab/README.md
  - ../charts/openab/templates/
  - ../AGENTS.md
  - ../docs/multi-agent.md
related: [docker-images, k8s-manifests, ../architecture/security-model, ../concepts/multi-agent]
updated: 2026-05-12
---

`charts/openab/` 是官方 Helm chart。設計圍繞「每個 agent 一份完整的
Deployment+Secret+ConfigMap+PVC」的多 agent 模型運轉,加上一個可選的 standalone
gateway Deployment。Helm template 有幾個 Go 模板的常見地雷,`../AGENTS.md` 把
它們列為硬規則。

The official Helm chart. Designed around the multi-agent model: each
`agents.<name>` key produces its own Deployment + Secret + ConfigMap + PVC,
plus an optional standalone gateway Deployment / Service.

---

## Chart layout / Chart 結構

```
charts/openab/
├── Chart.yaml
├── values.yaml
├── README.md
├── templates/
│   ├── _helpers.tpl
│   ├── configmap.yaml         ← config.toml per agent
│   ├── deployment.yaml         ← Deployment per agent
│   ├── secret.yaml             ← bot token / app token per agent
│   ├── pvc.yaml                ← persistent storage per agent (helm.sh/resource-policy: keep)
│   ├── gateway.yaml            ← optional standalone gateway Deployment + Service
│   ├── gateway-secret.yaml     ← optional gateway secrets
│   └── NOTES.txt               ← post-install hints
└── tests/
    ├── adapter-enablement_test.yaml
    ├── configmap_test.yaml
    ├── gateway_test.yaml
    ├── message-processing-mode_test.yaml
    ├── persistence_test.yaml
    ├── secretenv_deployment_test.yaml
    ├── secretenv_test.yaml
    └── tool-display_test.yaml
```

`tests/` are **helm-unittest** files — they exercise template rendering for
edge cases (especially the Go-template gotchas below).

---

## The `agents.<name>` map / Agent 對映表

The single most important convention. Top level of `values.yaml`:

```yaml
agents:
  kiro:                # ← one key per agent
    enabled: true
    discord:
      botToken: "..."
      allowedChannels: ["..."]
    workingDir: /home/agent
    pool:
      maxSessions: 10
      sessionTtlHours: 24
    reactions:
      enabled: true
      toolDisplay: "full"
    persistence:
      enabled: true
      size: 1Gi
    # ... lots more knobs

  claude:              # ← second agent
    enabled: true
    discord:
      botToken: "..."
      allowedChannels: ["..."]
    image: ghcr.io/openabdev/openab-claude
    command: claude-agent-acp
    workingDir: /home/node
```

Every agent becomes a **separate Kubernetes deployment** — separate pod,
separate bot token, separate PVC. They never share state. See
`../docs/multi-agent.md`.

---

## Security defaults / 安全預設

From `values.yaml`:

```yaml
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000
  seccompProfile: { type: RuntimeDefault }

containerSecurityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: [ALL]
```

These are the [Security Model](../architecture/security-model.md) §3
defaults baked into the chart.

---

## Go-template gotchas / Helm chart 必踩坑

From `../AGENTS.md` § Helm Chart Checklist — these are the rules every chart
change must satisfy before merge:

| Rule | Why |
|---|---|
| `{{ if hasKey .Values "field" }}` for booleans, **not** `{{ if .Values.field }}` | a value of `false` is "nil-y" in Go templates; without `hasKey` the check silently drops the field |
| Always use `--set-string` for channel/user IDs | Discord/Slack IDs are big enough that float64 loses precision |
| PVCs must carry `helm.sh/resource-policy: keep` | never delete user OAuth tokens on `helm uninstall` |
| New value → update `values.yaml` with a comment + update `docs/config-reference.md` | discoverability + agent-readable docs |

Validation commands:

```bash
helm template test charts/openab
helm template test charts/openab --set agents.kiro.enabled=false
```

The second one catches "what happens when the default agent is disabled and
only a custom agent is left" — a frequent breakage mode.

---

## `secretEnv` vs `env` / 機敏環境變數

The chart has two ways to pass environment variables to the agent:

| Field | Stored where | Use when |
|---|---|---|
| `agents.<name>.env` | ConfigMap (plaintext) | non-secret model knobs |
| `agents.<name>.secretEnv` | Secret (`valueFrom.secretKeyRef`) | API keys, tokens |
| `agents.<name>.envFrom` | references an existing Secret / ConfigMap | bulk injection |

⚠️ Do **not** list the same key in both `env` and `secretEnv`. The chart
auto-adds `secretEnv` keys to `inherit_env` in `config.toml`, but a duplicate
in `env` shadows it.

---

## Standalone gateway / Standalone gateway

To deploy the gateway through the same chart:

```yaml
agents:
  kiro:
    gateway:
      enabled: true       # OAB config.toml [gateway] block
      deploy: true        # deploy the standalone gateway as a separate workload
      url: "ws://openab-kiro-gateway:8080/ws"
      platform: "telegram"
      botUsername: "my_bot"
      telegram:
        botToken: "..."
        secretToken: "..."
```

`deploy: true` triggers `templates/gateway.yaml` to render. `deploy: false`
+ `enabled: true` is for sharing one gateway across multiple agents (set
`url` to point at the existing gateway service).

---

## Hooks / 鉤子

The chart doesn't use Helm hooks for migrations — schema changes are
backward-compatible by policy (`../AGENTS.md` § Backward-Compatible
Defaults). Upgrades are plain `helm upgrade`.

---

## Cross-references / 交叉引用

- Docker images that the chart pulls: [deployment/docker-images](docker-images.md).
- Plain k8s manifests if you don't want Helm: [deployment/k8s-manifests](k8s-manifests.md).
- Helm-publishing automation in CI: [deployment/ci-cd](ci-cd.md).
- README excerpt: [`charts/openab/README.md`](../../charts/openab/README.md).
