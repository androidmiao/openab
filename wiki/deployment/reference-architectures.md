---
title: "Reference Architectures — `docs/refarch/`"
type: deployment
sources:
  - ../docs/refarch/aws-ecs-fargate-spot.md
  - ../docs/refarch/cronjob_k8s_refarch.md
  - ../docs/refarch/gbrain.md
  - ../docs/refarch/remote-ssh-debugging.md
related: [helm-chart, k8s-manifests, ../modules/cron]
updated: 2026-05-12
---

`docs/refarch/` 收錄四份**參考架構**(refarch):AWS ECS Fargate Spot、K8s
CronJob、GBrain(group-mind agent deployment 模型)、Remote SSH Debugging。它們
不是「官方推薦」,而是已驗證過的範本,設計上是**喂給 agent 讀**讓 agent 替使用
者改寫到自己的環境。

`docs/refarch/` collects four reference architectures: AWS ECS Fargate Spot,
K8s CronJob screening, GBrain, Remote SSH Debugging. They're **not** the
canonical recommendation — they're verified templates designed to be fed to
a coding agent that adapts them to the user's environment.

---

## 1. AWS ECS Fargate Spot

`../docs/refarch/aws-ecs-fargate-spot.md`

> Deploy a single OpenAB bot on ECS Fargate Spot for **~$2.7/month** with
> persistent auth via S3.

Architecture:

```
+-- AWS -------------------------------------------------------+
|                                                              |
|  +-- ECS Fargate Spot Task -------------------------------+ |
|  |  openab container                                       | |
|  |    └─ kiro-cli acp                                      | |
|  +---------------------------------------------------------+ |
|                                                              |
|  S3 bucket: auth tokens, settings, openab thread-map         |
|                                                              |
+--------------------------------------------------------------+
```

Notable trade-offs:

- **Spot interruption** → task gets terminated; ECS reschedules; OAuth
  persistence on S3 prevents re-login.
- **No PVC** → use an init container to `aws s3 sync` settings down on
  start, plus a sidecar (or shutdown hook) to sync up on stop.
- **No Helm** → CloudFormation / Terraform / Copilot CLI / Click-Ops.

Intended usage (per the refarch's preamble):

> Prompt your AI agent with something like:
> `per https://github.com/openabdev/openab/blob/main/docs/refarch/aws-ecs-fargate-spot.md
>  deploy an openab on ECS Fargate Spot for me`
> and it will guide you through (or handle) the full setup.

---

## 2. K8s CronJob screening

`../docs/refarch/cronjob_k8s_refarch.md`

Companion to [ADR-004 Basic CronJob](../adrs/adr-004-basic-cronjob.md). For
**complex scheduled workflows** (claim-an-item, dispatch, post-back) where
in-OAB cron is insufficient.

```
GitHub Project Board (Incoming column)
        │  every 30 min
        ▼
Kubernetes CronJob
   concurrencyPolicy: Forbid
        │
        ▼
Ephemeral Job Pod
   image: ghcr.io/openabdev/openab-codex:latest
   command: bash /opt/openab-project-screening/screen_once.sh
     ├─ gh project item-list ...
     ├─ codex exec (one-shot agent run)
     ├─ post summary to Discord
     ├─ create Discord thread
     └─ post full report
        │
        ▼
GitHub Project Board (PR-Screening column)
        ▼
Human or agent follow-up
```

Key design points:

- The CronJob is a **separate workload** from the long-running OpenAB
  deployment. They share the Discord channel, not state.
- `concurrencyPolicy: Forbid` prevents overlap if a run runs long.
- Uses `codex exec` (one-shot mode) rather than ACP — the screening job
  doesn't need a persistent session.

---

## 3. GBrain

`../docs/refarch/gbrain.md`

A pattern for running OpenAB as the bridge in a **group-mind / multi-agent
collaboration** topology. Multiple agents share a channel and converge on
solutions via a structured protocol (see `../docs/multi-agent.md` and
[concepts/multi-agent](../concepts/multi-agent.md)).

OpenAB provides primitives (`allow_bot_messages`, `trusted_bot_ids`,
`max_bot_turns`); the GBrain refarch shows one way to compose them.

---

## 4. Remote SSH Debugging

`../docs/refarch/remote-ssh-debugging.md`

How to attach `kubectl exec` or `gh codespace ssh` into a running OpenAB
pod for live debugging — covering shell access, tailing `tracing` logs,
re-running CLI auth, and dumping `~/.openab/thread_map.json` for inspection.

---

## Common theme / 共通主題

All four refarchs:

1. **Pin a concrete topology** (not "here are 17 options").
2. **Assume an AI agent reads the doc** and adapts it to the user's
   environment, rather than walking the user through manually.
3. **Defer to Helm / Terraform / CloudFormation** for the actual
   instantiation — the refarch is the design, not the toolchain.

This matches the [AI-Native](../architecture/design-philosophy.md#3-ai-native--ai-原生操作)
philosophy.
