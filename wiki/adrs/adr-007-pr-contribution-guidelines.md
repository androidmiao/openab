---
title: "ADR-007 — PR Contribution Guidelines"
type: adr
sources:
  - ../docs/adr/pr-contribution-guidelines.md
  - ../CONTRIBUTING.md
  - ../.github/pull_request_template.md
related: [../deployment/ci-cd]
updated: 2026-05-12
---

> **Status**: Accepted.
> **Date**: 2026-04-13 · **Author**: @chaodu-agent.

OpenAB 的 PR 標準。每個 PR 都得交代「**這解決什麼問題、學了哪些前例、為什麼選
這條路、有哪些 trade-off**」。架構/runtime/agent/scheduling/delivery/persistence
類的改動**強制**做 prior-art research(最少 OpenClaw + Hermes Agent)。docs /
chore / CI / release / trivial fix 走輕量流程。目的:把研究成本放在最了解問題
的人(貢獻者)身上,而不是 reviewer。

OpenAB's PR standard. Every PR must explain **what problem this solves,
what prior art exists, why this approach over alternatives, what tradeoffs
remain**. Architectural / runtime / agent / scheduling / delivery /
persistence changes require **prior-art research** against at least
OpenClaw + Hermes Agent. Docs / chore / CI / release / trivial fixes use a
lighter process.

---

## Required PR sections / 必填段落

The `../.github/pull_request_template.md` enforces these:

| # | Section | Purpose |
|---|---|---|
| 0 | Discord Discussion URL | strongly recommended — link prior discussion |
| 1 | What problem does this solve? | plain-language pain point + linked issue |
| 2 | At a Glance | ASCII diagram of the change |
| 3 | Prior Art & Industry Research | required for architectural changes |
| 4 | Proposed Solution | technical approach |
| 5 | Why This Approach | why over alternatives |
| 6 | Alternatives Considered | what was evaluated and rejected |
| 7 | Validation | rust / helm / ci / docs checks |

---

## Tiered prior-art / 分級的前人研究

| Tier | Trigger | Requirement |
|---|---|---|
| Heavy | architectural, runtime, agent, scheduling, delivery, persistence | research **at minimum** OpenClaw + Hermes Agent, document with links |
| Light | docs-only, chore, CI, release, trivial bug fix | write "Not applicable — \<reason\>" in the prior-art section |

The two reference projects:

- **[OpenClaw](https://github.com/openclaw/openclaw)** — largest OSS AI
  agent gateway. Plugin architecture across 7+ platforms. Mature patterns
  for media, security, session management.
- **[Hermes Agent](https://github.com/NousResearch/hermes-agent)** — Nous
  Research's self-hosted agent. Gateway across 17+ platforms. Strong prior
  art on messaging, tool integration, service management.

For each, document:
1. How they solve the same problem (with source links).
2. Key architectural decisions.
3. What we can learn from their approach.

If neither addresses the problem, state explicitly with evidence.

---

## Good example / 好範例

The ADR cites Issue #224 / PR #525 (voice-message STT) as the model: a
thorough prior-art investigation comparing OpenClaw's
`audio-transcription-runner.ts` preflight pipeline with Hermes Agent's
`transcription_tools.py` local-first approach, with a comparison table and
explicit reasoning for OpenAB's simpler OpenAI-compatible endpoint design.

---

## Why this matters / 為何如此

> Without a clear PR standard, we see PRs that:
> - Jump straight to implementation without explaining the problem.
> - Don't research how existing projects solve the same problem.
> - Don't justify why a particular approach was chosen over alternatives.
> - Make review harder because reviewers must do the research themselves.
>
> This front-loads research cost onto the contributor (who understands
> the problem best) rather than distributing it across reviewers.

---

## Enforcement / 強制執行

- `.github/pull_request_template.md` ships the form.
- `pr-discussion-check.yml` GitHub Action validates required sections.
- CODEOWNERS sign-off is required for the relevant area.

---

## Cross-references / 交叉引用

- The CI workflows: [deployment/ci-cd](../deployment/ci-cd.md).
- `CONTRIBUTING.md` is the contributor-facing summary of this ADR.
