---
title: "CI / CD — `.github/workflows/`"
type: deployment
sources:
  - ../.github/workflows/
  - ../RELEASING.md
related: [helm-chart, docker-images]
updated: 2026-05-12
---

`.github/workflows/` 共 **22 個** GitHub Actions workflow,分四類:**build &
test**(每 PR 跑)、**release**(切版時自動化)、**issue / PR triage**(社群
維運)、**stale 自動關閉**。Helm publishing 與 release PR automation 是其中比
較吃重的兩個工作流。

22 GitHub Actions workflows in four buckets: build & test (per PR), release
automation (on tag / cut-release), issue/PR triage automation (community ops),
and stale-bot housekeeping.

---

## Build & test / 建置與測試

| Workflow | When | What |
|---|---|---|
| `ci.yml` | every PR + push to main | `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo test` |
| `build.yml` | every PR | `cargo build` for the main `openab` binary across targets |
| `build-binaries.yml` | every PR | builds release binaries (cross-platform) |
| `build-gateway.yml` | every PR | builds the `openab-gateway` binary |
| `chart-test.yml` | every PR | `helm template` + `helm-unittest` against `charts/openab/tests/` |
| `docker-smoke-test.yml` | every PR | smoke-builds each of the seven Dockerfiles |

The full set is the gate for merging. From `../AGENTS.md` § Build & Verify:

> `cargo fmt && cargo clippy -- -D warnings && cargo test` — all three must
> pass before pushing.

---

## Release / 發版

| Workflow | When | What |
|---|---|---|
| `release.yml` | git tag `v*` | builds + publishes Docker images to GHCR; creates GitHub Release with notes |
| `release-pr.yml` | manual dispatch | opens a PR with version bumps for the main crate |
| `gateway-release-pr.yml` | manual dispatch | same for the gateway crate (independent versioning, currently `0.4.0` vs main `0.8.3`) |
| `tag-on-merge.yml` | merge to main | auto-creates a tag after a release PR merges |

Two-crate / two-cadence model: `openab` (main) and `openab-gateway` are
released **independently**. `RELEASING.md` is the human-readable runbook.

---

## Helm publishing / Helm 發行

`charts/openab/` is published to `https://openabdev.github.io/openab` via
GitHub Pages on each release. The publishing CI lives in this workflow set
(see `../docs/helm-publishing.md` for the published-repo flow).

---

## Issue / PR triage / Issue 與 PR 自動分流

| Workflow | What |
|---|---|
| `issue-triage.yml` | label + route new issues by template |
| `issue-check.yml` | sanity-check issue body for required fields |
| `issue-pending-maintainer.yml` | nudge issues that need maintainer attention |
| `pending-maintainer.yml` | similar for PRs |
| `pending-screening.yml` | for PRs awaiting initial screening |
| `pr-discussion-check.yml` | enforce ADR-007 PR-template sections |
| `pr-preview.yml` | render PR preview (chart / docs) |
| `needs-rebase.yml` | label PRs that need rebase against main |

The PR-template enforcement (ADR-007) is the one that catches "PR has no
problem statement / no prior-art research". See
[adrs/adr-007-pr-contribution-guidelines](../adrs/adr-007-pr-contribution-guidelines.md).

---

## Stale / 過時自動處理

| Workflow | What |
|---|---|
| `close-stale-issues.yml` | warn → close stale issues |
| `close-stale-prs.yml` | warn → close stale PRs |
| `stale-issue.yml` | mark stale |
| `stale-contributor.yml` | nudge contributors who went quiet |

Standard `actions/stale` configuration.

---

## Required checks / 必過檢查

For a PR to be mergeable:

1. `ci.yml` (fmt + clippy + test)
2. `build.yml` (cross-platform — note `cargo check --target x86_64-pc-windows-gnu`
   from `../AGENTS.md` § Cross-Platform)
3. `chart-test.yml`
4. `docker-smoke-test.yml`
5. PR-template compliance (ADR-007)
6. CODEOWNERS sign-off (from `.github/CODEOWNERS`)

---

## Cross-references / 交叉引用

- Per-PR contribution standard: [adrs/adr-007-pr-contribution-guidelines](../adrs/adr-007-pr-contribution-guidelines.md).
- Release runbook: `../RELEASING.md`.
- Helm publishing notes: `../docs/helm-publishing.md`.
