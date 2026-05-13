---
title: "Docker Images — The Seven Dockerfiles"
type: deployment
sources:
  - ../Dockerfile
  - ../Dockerfile.claude
  - ../Dockerfile.codex
  - ../Dockerfile.copilot
  - ../Dockerfile.cursor
  - ../Dockerfile.gemini
  - ../Dockerfile.opencode
  - ../gateway/Dockerfile
  - ../AGENTS.md
related: [helm-chart, ../agents/agents]
updated: 2026-05-12
---

OpenAB 維護 **七支 Dockerfile**:預設 `Dockerfile`(內附 kiro-cli)加上六個
agent-specific 變體(`.claude`、`.codex`、`.copilot`、`.cursor`、`.gemini`、
`.opencode`)。Gateway 另有一支獨立的 `gateway/Dockerfile`。`../AGENTS.md` 把
「動了一支就要評估全部七支」列為 Critical Rule #4。

OpenAB ships **seven Dockerfiles**: the default `Dockerfile` (with `kiro-cli`
bundled) plus six agent-specific variants. The standalone gateway has its
own `gateway/Dockerfile`. `../AGENTS.md` Critical Rule #4 makes "touch one →
evaluate all" a hard constraint.

---

## Why so many / 為什麼分這麼多

Each agent CLI has its own runtime stack:

| Image | Bundles | Reason |
|---|---|---|
| `Dockerfile` (default) | `kiro-cli` (Go binary) | Kiro is the default; ships in the main image. |
| `Dockerfile.claude` | Node 20 + `@agentclientprotocol/claude-agent-acp` + `@anthropic-ai/claude-code` | Claude Code is npm-distributed. |
| `Dockerfile.codex` | Node + `@zed-industries/codex-acp` + OpenAI Codex CLI | Same npm runtime. |
| `Dockerfile.copilot` | `gh` + Copilot CLI extension | gh CLI is the auth surface. |
| `Dockerfile.cursor` | `cursor-agent` binary | Vendor-provided binary. |
| `Dockerfile.gemini` | Node + Gemini CLI | npm. |
| `Dockerfile.opencode` | Node + OpenCode | npm. |
| `gateway/Dockerfile` | `openab-gateway` binary | standalone; no agent CLI inside. |

Mixing them into one mega-image would carry every agent's runtime (Go +
Node + Cursor binary + gh + …) on every pod — wasteful and a wider attack
surface. Per-agent images keep each pod minimal.

---

## Common base layers / 共用層

All seven OpenAB images share:

1. A **Debian-slim base** for glibc compatibility with the agent CLIs.
2. **`tini`** as PID 1 baseliner (signal forwarding to the openab process).
3. The **`openab` binary** built from `Cargo.toml` v0.8.3.
4. Non-root user (UID 1000).
5. The **agent-specific** layer (kiro-cli / npm install / vendor binary).

`../AGENTS.md` Critical Rule #4:

> A change to one Dockerfile MUST be evaluated against ALL. Common layers
> (base image, openab binary, tini) are shared — update all or explain why
> not.

This is what keeps the seven images consistent without a build-matrix tool.
The CI workflows under `.github/workflows/build-binaries.yml` run a smoke
test against each variant on every PR.

---

## Build commands / 建置命令

```bash
# default (Kiro)
docker build -t openab:latest .

# Claude Code
docker build -f Dockerfile.claude -t openab-claude:latest .

# Codex
docker build -f Dockerfile.codex -t openab-codex:latest .

# … similar for copilot / cursor / gemini / opencode

# Gateway
docker build -f gateway/Dockerfile -t openab-gateway:latest gateway/
```

Released images appear at `ghcr.io/openabdev/openab[-<agent>]:<version>`.

---

## Image size profile / 鏡像大小輪廓

Rough order of magnitude (at v0.8.3 / v0.4.0):

```
openab           ~50-80 MiB   (Kiro is Go-static; tiny)
openab-claude    ~250 MiB     (Node runtime + npm packages)
openab-codex     ~250 MiB     (same shape as claude)
openab-gemini    ~250 MiB     (npm)
openab-opencode  ~250 MiB     (npm)
openab-copilot   ~150 MiB     (gh + extension)
openab-cursor    ~120 MiB     (single binary)
openab-gateway   ~50 MiB      (Rust-static; no agent inside)
```

Numbers approximate; check the GHCR registry for current values.

---

## Smoke testing / 冒煙測試

`docker-smoke-test.yml` (`.github/workflows/`) builds each variant in CI and
runs a minimal "binary boots and responds to `--version`" check. This is the
cheapest catch for regressions where one Dockerfile breaks while the others
still build.

---

## Cross-references / 交叉引用

- Per-agent setup that maps to each image: [agents/agents](../agents/agents.md).
- CI workflows that build/release them: [deployment/ci-cd](ci-cd.md).
- Helm `image:` field mapping: [deployment/helm-chart](helm-chart.md).
