---
title: "Observability — Logs, Reactions, and Health"
type: concept
sources:
  - ../src/main.rs
  - ../src/reactions.rs
  - ../Dockerfile.claude
related: [../modules/reactions, ../architecture/message-flow, ../adapters/discord, cost-and-quota]
updated: 2026-05-13
---

OpenAB 沒有 Prometheus、沒有 OpenTelemetry — 它只用 `tracing`(stdout 結構化
log)+ Docker `HEALTHCHECK` + **Discord 上的 reaction emoji** 三條觀察通道。三
條合起來就足以判斷 broker、Discord adapter、agent CLI 三層誰活著、誰卡住、誰
燒 token。

OpenAB ships with three observability channels — and only three:
**`tracing` logs on stdout**, **Docker `HEALTHCHECK`**, and **the reaction
emoji FSM on Discord messages**. Together they're enough to debug the
three-layer pipeline (broker / Discord adapter / agent CLI).

---

## 1. `tracing` logs

`../src/main.rs` 用 `tracing-subscriber` 預設的 fmt layer。重點屬性:

| 屬性 | 值 |
|---|---|
| 輸出位置 | stdout(Docker 會收) |
| 時間戳格式 | RFC3339,**UTC** with `Z` 後綴(`2026-05-13T12:27:59Z`) |
| 顏色 | 自動偵測;在 `docker compose logs` 內無色 |
| Filter env var | `RUST_LOG`,預設 `openab=info` |

時間戳是 UTC 寫死的 — `tracing-subscriber::fmt()` 沒指定 `.with_timer(...)`,
預設就是 UTC RFC3339。**台灣讀者 +8h** 才是本地時間:`12:27Z = 20:27 Asia/Taipei`。
看到「log 時間怎麼對不上」99% 是這個原因。

### 怎麼看 log

```bash
cd '<deployment-dir>'
docker compose logs -f openab            # 持續追
docker compose logs --tail=100 openab    # 最後 100 行
docker compose logs --since=30m openab   # 最近 30 分鐘
docker compose logs --since=12:00 openab # 從 UTC 12:00 起(! 是 UTC)
```

### 想要更多 / 更少細節

```yaml
# docker-compose.yml
services:
  openab:
    environment:
      RUST_LOG: "openab=debug"           # 全部 debug 級
      # 或更細:
      RUST_LOG: "openab=info,openab::acp=debug,openab::discord=debug"
```

`info`(預設)涵蓋啟動 / 連線 / session 生命週期。`debug` 才看得到「為什麼這
則訊息被擋」的細節(例如 `reason=user_not_allowed`)。

### 把 UTC 改成本地時間

短期 — pipe 過 `awk` 轉:

```bash
docker compose logs -f openab | awk '{
  if (match($0, /[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2}:[0-9]{2}:[0-9]{2}/)) {
    cmd = "TZ=Asia/Taipei date -j -f \"%Y-%m-%dT%H:%M:%SZ\" \""substr($0,RSTART,20)"Z\" +\"%H:%M:%S\"";
    cmd | getline t; close(cmd);
    print t substr($0, RSTART+27)
  } else { print $0 }
}'
```

長期 — 改 `main.rs` 加 `.with_timer(ChronoLocal::rfc_3339())` + 容器設 `TZ`
環境變數。需要 rebuild image。一般不建議,UTC 在跨時區 server log 是業界慣例。

---

## 2. Docker `HEALTHCHECK`

`../Dockerfile.claude` 第 37-38 行:

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD pgrep -x openab || exit 1
```

純 process 存活檢查 — 只要 `openab` 二進位活著就健康。**不檢查**:Discord 連
線狀態、Claude OAuth 有效性、Session pool 飽和度。

看 health 狀態:

```bash
docker compose ps                                 # STATUS 欄會顯示 healthy / unhealthy
docker inspect openab-claude --format='{{.State.Health.Status}}'
```

如果 `unhealthy`:`openab` 已經 crash 或被 kill。看 log 找 panic / SIGTERM:

```bash
docker compose logs --tail=200 openab | grep -E 'panic|SIGTERM|shutdown|error'
```

---

## 3. Reaction FSM 當 observability

最被低估的觀察手段。`../src/reactions.rs` 的 `StatusReactionController` 把
agent 內部狀態**直接**渲染到觸發那則 user 訊息上,**人眼**就能讀:

| Emoji | 狀態 | 你能推論的事 |
|---|---|---|
| 👀 | queued | broker 收到訊息、過了所有 gate、準備 dispatch |
| 🤔 | thinking | agent 開始跑,正在生 token 但還沒呼叫工具 |
| 🔥 / 👨‍💻 / ⚡ | tool active | agent 正在用工具(generic / coding / web) |
| 🆗 | done | turn 結束,完整回應已渲染 |
| 😱 | error | turn 失敗(會配合錯誤訊息) |
| (隨機 mood) | 完成後附帶 | 純 vibes;不代表狀態 |

時間語意(可在 `[reactions.timing]` 調整):

| 區間 | 預設 | 意義 |
|---|---|---|
| `debounce_ms` | 700 | 狀態變化要連續這麼久才會 commit |
| `stall_soft_ms` | 10000 | 沒事件超過 10s,靜默切「仍在思考」 |
| `stall_hard_ms` | 30000 | 超過 30s 顯式向使用者表達 stall |
| `done_hold_ms` | 1500 | 🆗 顯示多久才被清(如果 remove_after_reply) |
| `error_hold_ms` | 2500 | 😱 顯示多久 |

實戰判斷:

- **沒出現 👀** → 訊息根本沒進 OpenAB(Privileged Intents 沒開?頻道沒對?DM 沒
  開?)→ 看 [message-flow gates](../architecture/message-flow.md)。
- **👀 之後卡死沒下一步** → broker 接到了但沒 dispatch,可能 dispatcher 對該
  thread 已有 in-flight。或 ACP 子行程 spawn 失敗。看 log。
- **🤔 之後卡很久** → agent CLI 在思考。如果超過 `stall_hard_ms`(30s)還沒動,
  可能 Claude API 慢或卡 OAuth。`docker compose exec openab claude auth status`。
- **🔥 / 👨‍💻 之後卡** → agent 在跑工具。如果工具是 shell 命令而那命令卡住
  (例如等 stdin)就會表現成這樣。
- **🆗 出現但訊息空白** → 罕見,可能是 markdown 渲染問題;檢查 `tables` 設定。

詳細的狀態機:[`../modules/reactions`](../modules/reactions.md)。

---

## 三條合起來怎麼判斷 / Triage Cheat Sheet

```
                       reaction          log                  health
                       ─────────         ─────                ──────
Container 沒跑          (沒進 Discord)    (沒 log)              unhealthy
Discord 沒連上          (沒進 Discord)    "discord bot running" 但無 "connected" healthy
Gate 擋掉              (完全無 emoji)    reason=*_not_allowed   healthy
Dispatcher 卡           👀 不動           spawn 訊息缺           healthy
Agent CLI 卡            🤔 不動           無 agent_message_chunk healthy
Claude API 問題         🤔→🔥→久          rate limit / 401      healthy
Markdown 渲染問題       🆗 但訊息怪       無 error level         healthy
```

最快的單一觀察點:**在 Discord 上看 emoji 變化**。比 `docker compose logs -f`
門檻低、能讓你瞬間定位卡哪一層。

---

## 沒有的東西 / What's Intentionally Missing

| 沒有 | 為什麼 |
|---|---|
| Prometheus / metrics endpoint | OpenAB 是 outbound-only,不開 port — 加 `/metrics` 等於破壞核心不變式 |
| OpenTelemetry tracing | tracing crate 有,但 OTLP exporter 沒接;不阻擋你自己加 |
| 結構化 JSON log | `tracing-subscriber` 支援(`features=["json"]` 已啟用),只是 `main.rs` 沒用。可改 |
| Dashboard | 屬於 [Design Philosophy thin-bridge](../architecture/design-philosophy.md):觀察留給上層 |

要做企業級觀測 → 把 OpenAB stdout pipe 到 Loki / CloudWatch / Stackdriver。
Container log 已經是結構化的 `key=value` pairs,grep 友善;進 Loki 後 LogQL
可寫 alert(例如 5 分鐘內 `reason=user_not_allowed` 超過 100 次 → 告警)。

---

## 交叉引用 / Cross-References

- Reaction 狀態機完整描述:[modules/reactions](../modules/reactions.md)
- 7 道 gate 的失敗訊息:[architecture/message-flow](../architecture/message-flow.md)
- 燒 token 的真實時機:[concepts/cost-and-quota](cost-and-quota.md)
- Discord adapter 細節:[adapters/discord](../adapters/discord.md)
- SOP 內的 troubleshooting tree:[`../../deploy-claude-discord/SOP.md`](../../deploy-claude-discord/SOP.md#7-troubleshooting--常踩坑與排查)
