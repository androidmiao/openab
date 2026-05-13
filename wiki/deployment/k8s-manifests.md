---
title: "Kubernetes Manifests — `k8s/`"
type: deployment
sources:
  - ../k8s/configmap.yaml
  - ../k8s/deployment.yaml
  - ../k8s/pvc.yaml
  - ../k8s/secret.yaml
  - ../README.md
related: [helm-chart, ../architecture/security-model]
updated: 2026-05-12
---

`k8s/` 是不想用 Helm 時的 plain manifest 起手式:一個 Deployment、一個 PVC、
一個 ConfigMap、一個 Secret。功能對標 Helm chart 的「一個 agent + Discord」最小
配置;多 agent 或 gateway 整合請改用 Helm。

`k8s/` is the plain-YAML starter pack for deployments that don't use Helm.
Functionally equivalent to a single-agent Discord-only Helm install with
defaults; for multi-agent or gateway setups, use the Helm chart instead.

---

## The four manifests / 四個 manifest

| File | Purpose |
|---|---|
| `k8s/deployment.yaml` | single-container pod with config + data volume mounts |
| `k8s/configmap.yaml` | `config.toml` mounted at `/etc/openab/` |
| `k8s/secret.yaml` | `DISCORD_BOT_TOKEN` (and friends) injected as env vars |
| `k8s/pvc.yaml` | persistent storage for auth + settings (Kiro tokens, openab thread-map) |

---

## Quick deploy / 快速部署

From `../README.md`:

```bash
kubectl create secret generic openab-secret \
  --from-literal=discord-bot-token="your-token"

kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/pvc.yaml
kubectl apply -f k8s/deployment.yaml
```

`k8s/secret.yaml` is in the repo as a **template**; the real Secret should
be created from your token via `kubectl create secret` so no plaintext
token ever lands in git.

---

## What's in the Deployment / Deployment 內容

Roughly:

```yaml
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
        - name: openab
          image: ghcr.io/openabdev/openab:latest
          args: ["run", "-c", "/etc/openab/config.toml"]
          envFrom:
            - secretRef: { name: openab-secret }
          securityContext:
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities: { drop: [ALL] }
          volumeMounts:
            - { mountPath: /etc/openab, name: config, readOnly: true }
            - { mountPath: /data,       name: data }
      volumes:
        - { name: config, configMap: { name: openab-config } }
        - { name: data,   persistentVolumeClaim: { claimName: openab-data } }
```

The security defaults mirror [Security Model § 3](../architecture/security-model.md).

---

## When to prefer manifests over Helm / 何時用 manifest 而不是 Helm

- One agent, single platform, no gateway.
- Air-gapped environment without Helm.
- Want every line of YAML visible in git review.

For everything else — multi-agent, gateway, multi-environment — Helm is
strongly preferred. See [deployment/helm-chart](helm-chart.md).

---

## Storage / 儲存

The PVC holds:

- `~/.kiro/` — Kiro CLI settings, prior sessions.
- `~/.local/share/kiro-cli/` — OAuth tokens (do **not** delete on
  uninstall — annotate the PVC with `helm.sh/resource-policy: keep` if
  managed by Helm; for plain manifests, just don't `kubectl delete pvc`).
- `~/.openab/thread_map.json` — suspended ACP session IDs for resume after
  pod restart (see [concepts/session-pool](../concepts/session-pool.md)).

---

## Reference architectures / 參考架構

For non-Kubernetes setups, see [deployment/reference-architectures](reference-architectures.md):

- AWS ECS Fargate Spot (~$2.7/month for a single bot).
- K8s CronJob screening pattern.
- Remote SSH debugging into the pod.
