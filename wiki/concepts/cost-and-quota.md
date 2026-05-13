---
title: "Cost & Quota — When does OpenAB actually consume LLM tokens?"
type: concept
sources:
  - ../src/adapter.rs
  - ../src/dispatch.rs
  - ../src/acp/pool.rs
  - ../src/cron.rs
  - ../AGENTS.md
  - ../DESIGN.md
related: [../architecture/message-flow, ../architecture/design-philosophy, session-pool, multi-agent, ../modules/bot-turns, ../modules/cron]
updated: 2026-05-13
---

**最短答案:OpenAB 自己不打 LLM API。Tokens 只在某筆訊息通過全部 7 道濾鏡並
被 dispatch 到 agent CLI 時消耗。** 也就是說,**container 開著但沒人發訊息 =
0 tokens**;只要把 `allowed_users` 鎖死,即使 24/7 跑也不會被陌生人燒。

> Bottom line: OpenAB the broker calls no LLM API itself. Tokens are
> consumed **only** when a message survives all seven inbound gates and the
> dispatcher hands it to the agent CLI. **Idle container = $0.**

---

## 三層分清楚 / Three Layers, Three Cost Profiles

```
                        cost profile
─────────────────────   ──────────────────────────────
Discord / Gateway         heartbeats only, no $
       ↕ WebSocket
OpenAB broker process     compute on your host, no LLM $
       ↕ stdio JSON-RPC
Agent CLI subprocess      LLM $ ← *only this layer*
   (claude-agent-acp,
    kiro-cli, codex, …)
```

| 層 | Idle | 收到觸發訊息 |
|---|---|---|
| Discord Gateway / Gateway WS | 心跳封包 (KB/分) | 不變 |
| OpenAB broker process | RAM ~50 MB,CPU ~0% | 短暫 CPU 尖峰 |
| Agent CLI subprocess | 0(沒被 spawn) / 0(spawn 但 idle) | **LLM API call** |

OpenAB 的 [thin-bridge 設計哲學](../architecture/design-philosophy.md#1-thin-bridge--薄橋接)
明文寫:broker 不持有 model、不 inject prompts、不 govern outputs。所以它就是
**轉介層**,自己不會花錢。

---

## 哪些事件**會**觸發 token 消耗 / What Burns Tokens

依照 [message-flow](../architecture/message-flow.md) 的 7 道濾鏡,**全過**才會
dispatch:

```
1. channel allowlist
2. user allowlist     ← 最強的「鎖人」槓桿
3. bot message filter
4. thread detection
5. bot_owns
6. multibot detection
7. @mention / DM / role trigger
       ↓
   dispatcher submit
       ↓
   session_prompt → agent CLI ← *LLM API call here*
```

**任一關擋下 → 0 tokens**。Log 會看到:

```
DEBUG openab::discord: message ignored reason=channel_not_allowed
DEBUG openab::discord: message ignored reason=user_not_allowed
DEBUG openab::discord: message ignored reason=bot_message_filtered
```

---

## 哪些情境**會在你沒注意時**燒 token / Silent Spend Sources

依風險高低排:

### 1. `allowed_users = []` 且 `allow_dm = true`

DM 模式下,**任何**找得到 bot 名字的 Discord 用戶都能觸發 Claude 回覆。Discord
bot 名稱可被搜尋,**不算機密**。預設新部署最大的洞。

降風險:
```toml
[discord]
allowed_users = ["你的_discord_user_id"]
```
把使用者鎖死。即使 `allow_dm = true`,陌生人 DM 會在 gate ② 被擋。

### 2. `[[cron.jobs]]` 排得太頻繁

`../src/cron.rs` 的 [config-driven cron](../modules/cron.md) 每符合就送一筆 prompt
過去。`* * * * *`(每分鐘)會讓你一天叫 1440 次 LLM。預設 `[cron]` block 是空的
(`config.toml.example`)→ 不會自己跑。要加之前先想清楚頻率。

ADR-004 故意**不允許 agent 用自然語言設定自己的 cron**就是為了避免這種事 —
詳見 [adrs/adr-004-basic-cronjob](../adrs/adr-004-basic-cronjob.md)。

### 3. `allow_bot_messages` 開到 `mentions` / `all` 沒配 `trusted_bot_ids`

任何其他 bot 都能觸發你的 agent。即使沒人惡意,單純兩個 bot 互相 @ 也可能跑出
chain。

防護網:
- 把每個 bot 都列進對方的 `trusted_bot_ids`(白名單)。
- [`max_bot_turns`](../modules/bot-turns.md) 預設 100 是最後的 hard cap。
  人類訊息一進來就 reset 到 0。

### 4. 多人共用一個 channel

`allowed_channels = ["C..."]` 開放整個頻道。如果該頻道有 N 個成員,N 人都能 @
bot。沒有 user 白名單時無上限。

降風險:把 `allowed_users` 也設好,或關閉頻道權限。

---

## 哪些事**不**燒 token / What Doesn't Cost

| 行為 | 為什麼不收費 |
|---|---|
| Container 開著但沒人發訊息 | Agent CLI 可能根本沒 spawn |
| Session pool 有 active connection 但 idle | Anthropic 按 request 收費,sub-process 開著 ≠ in-flight request |
| Reaction FSM 在跑(👀→🤔)| 純本地 timer + Discord API,沒 LLM call |
| Discord Gateway 心跳 | TCP keepalive,不關 LLM |
| Claude OAuth token refresh | Auth 層,不算用量 |
| `docker compose logs -f` | 不會喚醒 agent |
| `docker compose exec openab claude auth status` | 純本地查 token 狀態 |
| Session 過 TTL 被 evict | 只是關 child process,不是 API call |
| Session/load 重接已暫存的 session | Anthropic 端有狀態檔但**沒**算錢給 broker 端的 load 操作 |

---

## Claude OAuth vs API Key 計費差異 / Auth Path Affects Billing

如果 `[agent].command = "claude-agent-acp"`(本部署的設定),OAuth 路徑使用
**你的 Claude.ai 訂閱配額**:

| 訂閱 | 行為 |
|---|---|
| 免費 | 額度小,很快撞限制 |
| Pro | 5-hour rolling window |
| Max | 較高 quota + 更多 model |

切到 `env = { ANTHROPIC_API_KEY = "${KEY}" }` 則走 **API key 按量計費**,沒月配
額但每筆 token 都算錢。**OAuth 是預設 + 推薦**:不會超出訂閱;額外不會有意外帳單。

查看用量:
```bash
docker compose exec openab claude auth status
# 或登 https://claude.ai 看 usage dashboard
```

---

## 給 operator 的三條決策樹 / Three Decision Trees

### 「我會不會半夜被人燒 token?」

```
allowed_users 非空? ── yes ──→ 不會,gate ② 擋住所有非白名單用戶
       │
      no
       │
       ▼
allow_dm == true? ── yes ──→ ⚠️ 任何 DM 都會觸發 Claude
       │
      no
       │
       ▼
allowed_channels 是私人 server? ── yes ──→ 只有 server 成員能用
       │
      no
       │
       ▼
公開 server channel + 任何人都可 @ → 有風險
```

### 「停用一段時間,要不要 down container?」

| 期間 | 建議 |
|---|---|
| 數小時(午餐 / 開會) | 留著,沒 LLM cost,只多用幾 KB 心跳 |
| 過夜 / 週末 | 留著也行;OAuth volume 不受影響 |
| 數日以上不用 | `docker compose down` 比較乾淨。OAuth volume 保留,`up -d` 接著用,不用重 login |
| 想徹底清掉 | `docker compose down -v` — ⚠️ OAuth 也會掉,要重做 `claude auth login` |

### 「我要做大量自動化,會不會爆 quota?」

- Pro/Max 的 5-hour window 大致夠正常使用;如果是**自動化批次**(例如 cron 每
  分鐘問一次 + 開到 24h),很容易把 window 用光。
- 建議:加 `max_bot_turns`、限制 cron 頻率、必要時切 API key 路徑用按量計費。
- 觀察方法:`docker compose logs -f openab | grep -E 'session_prompt|prompt_hard_timeout'`
  數一天的 prompt 次數。

---

## 三條最便宜的安全做法 / Three Cheap Hardening Steps

```toml
[discord]
allowed_users = ["你的_user_id"]   # gate ② 最強的鎖
allowed_channels = ["你的_channel_id"]  # gate ① 限制範圍
allow_dm = false                     # 除非真的需要 DM
allow_bot_messages = "off"            # 預設,不要動
trusted_bot_ids = []                  # 沒有要多 agent 互動就維持空
max_bot_turns = 100                   # 預設,當最後保險
```

這四行落地後,**只有你、在你指定的 channel、且 @mention 才會觸發 LLM**。Idle =
$0,被觸發 = 你自己負責。

---

## 交叉引用 / Cross-References

- 七道濾鏡細節:[architecture/message-flow](../architecture/message-flow.md)
- Bot 互動迴圈防護:[modules/bot-turns](../modules/bot-turns.md)
- Cron 啟動條件:[modules/cron](../modules/cron.md) + [adrs/adr-004-basic-cronjob](../adrs/adr-004-basic-cronjob.md)
- Session pool TTL / eviction:[concepts/session-pool](session-pool.md)
- 多 agent 場景:[concepts/multi-agent](multi-agent.md)
- 安裝 SOP 的「停掉/重啟」維運段:[`../../deploy-claude-discord/SOP.md`](../../deploy-claude-discord/SOP.md#8-日常維運--day-2-operations)
