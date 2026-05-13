---
title: "ADR-006 — Context-Aware Token for Agent-Initiated Platform Operations"
type: adr
sources:
  - ../docs/adr/context-aware-token.md
related: [../architecture/security-model, adr-002-custom-gateway]
updated: 2026-05-12
---

> **Status**: Proposed.
> **Date**: 2026-05-03 · **Author**: @chaodu-agent.
> **Related**: issue #339, PR #527 (superseded).

讓 agent 能**主動**和平台互動(更新 thread title、拉某筆歷史訊息、跨頻道 ping
其他 bot)的設計案。今天 agent 多半靠 steering-doc 的 `curl` hack 做到,這 ADR
要用一個**短期、agent-scoped、context-aware** 的 token 把它正名化,而**不**讓
agent 直接拿到平台級的長期憑證。

Lets agents **actively** interact with the platform (update thread title,
fetch a historical message, ping a bot in another channel) by issuing a
short-lived, agent-scoped, context-aware token — instead of either shipping
platform credentials to the agent or relying on `curl`-via-steering-doc
hacks.

---

## Why not PR #527 / 為什麼不直接前置 reply 內容

PR #527 proposed **always prepending** quoted message content at the OAB
transport layer. Rejected because:

- Always pays the cost (~500 tokens per reply) even when the agent already
  has it from history.
- Only solves one edge case (reply / quote context) out of the broader
  set of agent-initiated operations.
- Puts the decision in the wrong layer — OAB (transport) decides, instead
  of the agent.

A context-aware token lets the agent **pull on demand** when it
determines context is needed.

---

## Prior art / 前人

The ADR surveys OpenClaw and Hermes Agent:

| Project | Pattern | Lesson |
|---|---|---|
| **OpenClaw** | mediated 50-action message system; agent never gets API tokens; everything routes through Gateway + Channel Adapter | the mediated architecture **is** the security boundary |
| **Hermes Agent** | similar mediated tool-call interface | same — agents call tools, never APIs |

Three OpenClaw advisories inform the design constraints:

- GHSA-v3qc-wrwx-j3pw — LLM agent disabled exec approval via
  `config.patch`. Lesson: behavioural constraints alone are insufficient;
  agents can modify their own config.
- GHSA-2rqg-gjgv-84jm — workspace boundary override via attacker-
  controlled params. Lesson: the gateway must enforce boundaries
  regardless of caller overrides.
- GHSA-7jx5-9fjg-hp4m — ACP permission auto-approval bypass via untrusted
  metadata. Lesson: auto-approval heuristics that trust untrusted metadata
  are dangerous.

---

## Decision (proposed) / 決議(待定)

A **short-lived, agent-scoped, context-aware token** that:

1. Is issued by OpenAB (or the gateway) for each ACP session.
2. Encodes the agent's allowed operations and the current channel /
   thread / message context.
3. Is **mediated** by the same gateway boundary that handles inbound — the
   agent calls a generalised tool ("update_thread_title", "fetch_message",
   "ping_bot") and the gateway translates to the platform API.
4. Expires quickly and is **session-scoped** so a leaked token can't be
   reused outside the conversation.

This avoids:
- Long-lived platform credentials in the agent's reach.
- The "always prepend context" tax.
- Direct agent → platform API calls (which would break the
  outbound-only invariant on the OAB pod).

---

## Why this fits OAB's model / 為何符合 OAB 設計

- Mediation is what [ADR-002](adr-002-custom-gateway.md) already
  introduced for inbound; this ADR uses the same mediator for **outbound
  agent-initiated** calls.
- The OAB pod remains outbound-only (talks to the gateway and the agent
  CLI; nothing else).
- Agent credentials stay separate from platform credentials.

---

## Status / 狀態

Proposed. Not yet implemented at the time of this wiki bootstrap. The
absence of this primitive means agents currently work around the gap with
steering-doc `curl` hacks — which works but isn't auditable.

---

## Related / 相關

- The mediation pattern this builds on: [ADR-002](adr-002-custom-gateway.md).
- Why broker-side mediation is the right boundary:
  [architecture/security-model](../architecture/security-model.md).
