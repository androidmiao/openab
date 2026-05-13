# SOP — OpenAB + Claude Code + Discord(本機 Docker)

Standard Operating Procedure for installing **OpenAB + Claude Code + Discord**
on macOS (Apple Silicon) using Docker Desktop. Verified end-to-end on
**2026-05-13** with `ghcr.io/openabdev/openab-claude:latest`,
`@agentclientprotocol/claude-agent-acp@0.29.2`,
`@anthropic-ai/claude-code@2.1.124`,Docker Compose v2。

> 本文同時收錄常踩坑與 troubleshooting 樹。第一次部署的人**請逐步照做**;
> 已熟悉的人可以從 `## Cheat Sheet` 直接複製整段。

---

## 0. 前提 / Prerequisites

| 需要 | 說明 |
|---|---|
| Mac(Apple Silicon 或 Intel) | 本 SOP 在 M-series 上驗證;Intel 流程相同 |
| 一個 Discord 帳號 | 必須有**至少一個 server 的 Manage Server 權限**(自己開的 server 就有) |
| 一個 Claude 帳號 | 用於 `claude auth login` device flow,**不需要** ANTHROPIC_API_KEY |
| ~1 GB 磁碟空間 | Docker image (~600 MiB) + Claude `~/.claude/` token store |
| 網路 | 能對外連 GHCR(image)、Discord Gateway、Claude OAuth、Claude API |

預估時間:**約 30 分鐘**(其中 Discord Developer Portal 設定佔大半)。

---

## 1. 安裝 Docker Desktop / Install Docker

### 1.1 檢查 Homebrew(沒有就先裝)

```bash
which brew
```

- 有輸出路徑 → 跳 1.2。
- 顯示 `not found` → 跑下面這行,5-10 分鐘:
  ```bash
  /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
  ```
  裝完依畫面 `Next steps` 兩行指令把 brew 加進 PATH,**開新的 Terminal 視窗**。

### 1.2 裝 Docker Desktop

```bash
brew install --cask docker
```

下載約 600 MiB。完成後:

```bash
open -a Docker
```

第一次跑會跳出 GUI 視窗:
1. 同意服務條款。
2. `Use recommended settings` → Accept(會要一次 Mac 密碼)。
3. 登入步驟可 Skip(不需要 Docker Hub 帳號)。
4. 教學可 Skip。

**等到 Mac 右上角狀態列鯨魚 🐳 icon 不再轉動才算 ready**。

### 1.3 驗證

```bash
docker --version
docker compose version
docker run --rm hello-world
```

預期 `Docker version 27.x` / `Docker Compose version v2.x` / `Hello from Docker!`。

如果 `Cannot connect to the Docker daemon` → Docker Desktop 還沒完全啟動,
等鯨魚 icon 穩定再重試。

### 1.4(選用)Apple Silicon 啟用 Rosetta 2

```bash
softwareupdate --install-rosetta --agree-to-license
```

`openab-claude` image 一般有 ARM64 manifest,但啟用 Rosetta 比較保險。

---

## 2. 建立 Discord Bot / Create the Bot

**這一步只能用瀏覽器手動操作**,在 [Discord Developer Portal](https://discord.com/developers/applications) 完成。

### 2.1 建立 Application

1. **New Application** → 取名(例如 `OpenAB-Claude`)→ Create。
2. 左側 **Bot** tab(自動建好,不用手動「Add Bot」)。

### 2.2 ⚠️ 開啟 Privileged Gateway Intents(最常踩坑)

仍在 **Bot** tab,滾到 **Privileged Gateway Intents** 區塊:

- ✅ **Message Content Intent** — 沒開的話 bot **看不到訊息內容**,只有 metadata
- ✅ **Server Members Intent** — 用於解析 `@user` 顯示名稱

按 **Save Changes**。

### 2.3 取得 Bot Token

仍在 **Bot** tab → **Reset Token** → 複製。

> ⚠️ Token **只會出現一次**。先貼到備忘錄,等下一步用。
> 如果不小心關掉沒複製,可以再 Reset 一次(舊的就失效)。

### 2.4 邀請 Bot 進你的 Discord Server

左側 **OAuth2 → URL Generator**:

- **Scopes** 勾:`bot`
- **Bot Permissions** 勾:
  - `Send Messages`(傳送訊息)
  - `Embed Links`(嵌入連結)
  - `Attach Files`(附加檔案)
  - `Read Message History`(閱讀訊息記錄)
  - `Add Reactions`(新增表情符號)
  - `Create Public Threads`(建立公開討論串)
  - `Send Messages in Threads`(在討論串中傳送訊息)
  - `Manage Threads`(管理討論串)

頁面底端產生 URL → 用瀏覽器打開 → 選你的 server → 繼續 → 授權。

Bot 加入後在成員列表會**顯示離線**(因為 OpenAB 還沒跑),正常。

---

## 3. 在 Discord App 取得 Channel ID / Get Channel ID

### 3.1 開啟開發者模式

在 Discord **桌面 / 網頁 app**(不是 Developer Portal):

1. 左下角頭像旁的齒輪 ⚙️ →「使用者設定」。
2. 左側捲到底 →「**進階 / Advanced**」。
3. 把「**開發者模式 / Developer Mode**」切換為 **開啟**。

### 3.2 複製頻道 ID

1. 進入剛剛邀請 bot 的 server。
2. 在你要 bot 進駐的**文字頻道**名稱上**右鍵**(Mac trackpad 兩指點擊)。
3. 選單最底端 → **複製頻道 ID / Copy Channel ID**。

ID 是 18-19 位純數字,例如 `1503814913171394741`。

> 確認複製的是**頻道 ID**,不是「複製連結」、不是伺服器 ID。

---

## 4. 配置本機部署檔案 / Configure Files

### 4.1 找到 deploy folder

本 SOP 假設你的 deploy folder 是:
```
/Users/<you>/.../openab/deploy-claude-discord/
```

裡面已經有 `docker-compose.yml`、`config.toml`、`.env.example`、`.gitignore`。
如果沒有,從 wiki 或先前的部署 commit 取得。

### 4.2 建立 `.env` 並填入 token

```bash
cd '/Users/<you>/.../openab/deploy-claude-discord'

cp .env.example .env
chmod 600 .env

open -e .env
```

把第 4 行的 `paste-your-bot-token-here` 整段換成步驟 2.3 拿到的 token,存檔:

```
DISCORD_BOT_TOKEN=MTIzN...你的真實 token...XYZ
```

> `.env` 已經在 `.gitignore` 內,不會被 commit。

### 4.3 編輯 `config.toml` 填入 channel ID

```bash
open -e config.toml
```

把:

```toml
allowed_channels = ["YOUR_CHANNEL_ID"]
```

換成你步驟 3.2 複製的 ID:

```toml
allowed_channels = ["1503814913171394741"]
```

> **雙引號保留** — 即使是數字也要當字串,避免 float64 精度損失
> (Discord ID 大到 JSON Number 無法完整表示)。

### 4.4(選擇)決定 DM 還是 Channel-only

| 用法 | `config.toml` 設定 |
|---|---|
| 只在 server 頻道用 | `allow_dm = false`(預設) |
| 也要在 DM 用(快速測試最方便) | `allow_dm = true` |

DM 模式下,訊息被當作隱式 @mention,**不需要 @bot**。`allowed_channels` 在 DM
不適用;但 `allowed_users` 仍會生效(空陣列 = 允許所有人)。

### 4.5 Preflight check

```bash
cd '/Users/<you>/.../openab/deploy-claude-discord'

# 預期都回傳 0(無 placeholder 殘留):
grep -c 'paste-your-bot-token-here' .env
grep -c 'YOUR_CHANNEL_ID' config.toml

# .env 權限應為 600
stat -f '%Lp' .env   # macOS 用 -f,Linux 用 -c
```

---

## 5. 首次啟動 + Claude OAuth / First Boot

### 5.1 拉 image(首次 2-5 分鐘)

```bash
cd '/Users/<you>/.../openab/deploy-claude-discord'
docker compose pull
```

如果失敗(`Cannot connect to the Docker daemon`)→ 回 1.2 確認 Docker Desktop
是否就緒。

### 5.2 啟動 container

```bash
docker compose up -d
```

預期看到:
```
✔ Volume "deploy-claude-discord_claude-data"  Created
✔ Container openab-claude                     Started
```

### 5.3 看 log 確認 Discord adapter 連線

```bash
docker compose logs --tail=50 openab
```

正常 log:
```
INFO openab: config loaded agent_cmd=claude-agent-acp pool_max=5 discord=true
INFO openab: starting discord adapter ... allow_dm=true/false ...
INFO openab: discord bot running
INFO openab::discord: discord bot connected user=AmazingCat
INFO openab::discord: registered global slash commands
INFO openab::discord: registered guild slash commands guild_id=<server_id>
```

此時 bot 在 Discord 上會變綠燈 **但發訊息會卡** — 因為 Claude 還沒登入。

### 5.4 ⚠️ 執行 Claude OAuth device flow

```bash
docker compose exec openab claude auth login
```

畫面會印:
```
Open this URL in your browser:
  https://claude.ai/oauth/device?code=ABCD-1234
Then paste the code shown on that page here:
```

操作:
1. **複製 URL** 貼到瀏覽器。
2. 用 Claude 帳號登入(如果還沒登入)。
3. 網頁出現一段確認 code → **複製**。
4. 回 Terminal,**貼上 code,Enter**。
5. 看到 `Login successful` 或類似訊息。

Token 持久化到 `claude-data` named volume → `docker compose down` 不掉,
`docker compose down -v` 才會掉。

### 5.5 重啟讓 OpenAB 載入 OAuth

```bash
docker compose restart openab
docker compose logs -f openab
```

`-f` 持續追 log,等下測試時保留這個視窗。

---

## 6. 在 Discord 驗證 / Verify

### 6.1 觸發測試

**Channel 模式(`allow_dm = false`)**:在你 `allowed_channels` 內的頻道
,輸入訊息時 `@` 後從**下拉選單選 bot**(讓它變藍色 highlight),別純打字:

```
@AmazingCat 你好
```

**DM 模式(`allow_dm = true`)**:直接 DM 訊息:

```
你好,介紹一下你自己
```

### 6.2 預期行為

| 階段 | 觀察點 |
|---|---|
| ① 訊息被接收 | 你的訊息上出現 👀 emoji(reaction FSM 第一階段) |
| ② Agent 啟動 | log:`spawning agent cmd="claude-agent-acp"` + `session created session_id=...` |
| ③ Claude 思考 | 訊息 emoji 變 🤔 |
| ④ 工具使用(可選) | emoji 變 🔥 / 👨‍💻 / ⚡ |
| ⑤ 完成 | emoji 變 🆗 + 隨機 mood emoji(😌、😆 等) |
| ⑥ Bot 回覆 | Channel:建一個 thread 在裡面回。DM:直接回覆 |
| ⑦ Edit-streaming | 回覆訊息右下角會有「(已編輯)」字樣 — token 是邊收邊渲染 |

### 6.3 後續對話

Channel 模式:**進入 thread 後**,後續訊息**不用 @mention**
(`allow_user_messages = "involved"` 預設行為)。

DM 模式:DM 永遠不需要 @mention,每則訊息都會被當隱式 mention 處理。

---

## 7. Troubleshooting / 常踩坑與排查

### 7.1 Bot 在 Discord 顯示離線

| 可能原因 | 確認 / 修正 |
|---|---|
| Container 沒跑 | `docker compose ps` 看 status |
| Bot token 錯誤 | log 會有 `Authentication failed`;檢查 `.env` 是否完整(沒換行錯位) |
| 沒邀請進 server | 重跑 2.4 的 OAuth2 URL |

### 7.2 Bot 線上但完全沒反應(連 emoji 都沒出現)

最常見原因依優先序:

1. **Privileged Intents 沒開** → 回 2.2,**兩個都要開**,然後 `docker compose restart openab`。
2. **不是真正的 @mention** → 在 Discord 打 `@` 後**從下拉選單選 bot**,文字會變藍色 highlight。純文字 `@AmazingCat` 不算 mention。
3. **頻道不在 allowlist** → 右鍵頻道複製 ID,跟 `config.toml` 的 `allowed_channels` 比對。
4. **在 DM 但 `allow_dm = false`** → 把 `allow_dm` 改 `true`,重啟。或改去 server 頻道測試。

### 7.3 Bot 收到訊息(log 有顯示)但卡在 🤔 不回覆

```bash
docker compose logs openab | grep -E 'spawn|session|error|claude'
```

依輸出判斷:

| Log 字樣 | 解法 |
|---|---|
| `auth required` / `not logged in` | 重做 `docker compose exec openab claude auth login`,然後 restart |
| `rate limit` / `429` | Claude API 配額用完,等下次 reset 或檢查帳號限制 |
| `claude-agent-acp: not found` | image 損壞,`docker compose pull` 重抓 |
| 完全沒新 log | 訊息根本沒進來,回 7.2 |

### 7.4 改了 config.toml 但行為沒變

OpenAB 啟動時讀一次 config,改完**必須重啟**:

```bash
docker compose restart openab
docker compose logs --tail=20 openab
```

從 log 確認新值有生效(例如 `allow_dm=true`)。

### 7.5 想完全重來

```bash
docker compose down -v   # ⚠️ -v 會清掉 OAuth volume,要重新 login
rm -f .env               # 清掉 token
# 從 step 4 重新做
```

---

## 8. 日常維運 / Day-2 Operations

```bash
cd '/Users/<you>/.../openab/deploy-claude-discord'

# 看即時 log
docker compose logs -f openab

# 看 container 狀態
docker compose ps

# 確認 Claude OAuth 還有效
docker compose exec openab claude auth status

# 重啟(改 config 後必做)
docker compose restart openab

# 升級到最新 image
docker compose pull && docker compose up -d

# 進 shell 看內部
docker compose exec openab bash

# 停掉(保留 OAuth volume)
docker compose down

# 完全清掉(含 OAuth!)
docker compose down -v
```

### 8.1 備份 Claude OAuth token

OAuth 在 `claude-data` named volume 內;volume 被刪等於要重做 device flow:

```bash
docker run --rm \
  -v deploy-claude-discord_claude-data:/data \
  -v "$(pwd):/backup" \
  alpine \
  tar czf /backup/claude-data-backup-$(date +%Y%m%d).tgz -C /data .
```

復原:

```bash
docker compose down
docker volume rm deploy-claude-discord_claude-data
docker volume create deploy-claude-discord_claude-data
docker run --rm \
  -v deploy-claude-discord_claude-data:/data \
  -v "$(pwd):/backup" \
  alpine \
  tar xzf /backup/claude-data-backup-YYYYMMDD.tgz -C /data
docker compose up -d
```

### 8.2 升級 OpenAB / Claude Code 版本

預設 image tag 是 `latest`,所以 `docker compose pull` 就會更新:

```bash
docker compose pull
docker compose up -d
```

想釘版本 → 改 `docker-compose.yml` 的 `image:` 行(例如
`ghcr.io/openabdev/openab-claude:0.8.3`)。

### 8.3 觀察 Discord 流量

```bash
docker compose logs -f openab | grep -E 'discord|adapter'
```

---

## 9. 進階變化 / Advanced Variations

### 9.1 同時跑多個 agent(Kiro + Claude)

複製 `docker-compose.yml`,新增第二個 service(用 `ghcr.io/openabdev/openab:latest`
跑 Kiro 預設 image),用**第二個 Discord bot token**,共用同一個 server。
詳見 `../wiki/concepts/multi-agent.md`。

### 9.2 讓 Claude 看到你的 repo

把 Mac 上的 repo mount 進去:

```yaml
# docker-compose.yml
services:
  openab:
    volumes:
      - ./config.toml:/etc/openab/config.toml:ro
      - claude-data:/home/node
      - /Users/<you>/.../my-project:/workspace      # ← 新增
```

然後改 `config.toml`:

```toml
[agent]
working_dir = "/workspace"
```

⚠️ Claude 可以**讀也可以寫** mount 進去的內容,跑之前確認 git 狀態乾淨,
最好先 `git status` + commit。

### 9.3 改成只走 server 頻道(關 DM)

回 4.4,把 `allow_dm = true` 改回 `false`,重啟。

### 9.4 加 STT(語音訊息轉文字)

`config.toml` 加區塊:

```toml
[stt]
enabled = true
api_key = "${GROQ_API_KEY}"   # 或 OpenAI / 本機 Whisper
model = "whisper-large-v3-turbo"
base_url = "https://api.groq.com/openai/v1"
echo_transcript = true
```

把 `GROQ_API_KEY` 加進 `.env`,重啟。詳見 `../wiki/modules/media-stt.md`。

---

## 10. 解除安裝 / Uninstall

完全清掉本機部署:

```bash
cd '/Users/<you>/.../openab/deploy-claude-discord'

docker compose down -v          # 停掉並清 volume(包含 OAuth)
docker image rm ghcr.io/openabdev/openab-claude:latest   # 移除 image
```

Discord 端清掉(可選):

1. Developer Portal → 你的 Application → 右上 Delete App。
2. Server 管理員把 bot 從成員列表移除。

如果保留 Docker Desktop 給其他用途就不用解除安裝;否則:

```bash
brew uninstall --cask docker
rm -rf ~/Library/Containers/com.docker.docker
rm -rf ~/.docker
```

---

## Cheat Sheet / 速查

把 `<token>`、`<channel_id>` 換成你的值;從 `cd` 到 `compose up -d` 一氣呵成:

```bash
# 0. Docker Desktop ready,確認:
docker --version && docker run --rm hello-world

# 1. 部署資料夾
cd '/Users/<you>/.../openab/deploy-claude-discord'

# 2. 設定
cp .env.example .env && chmod 600 .env
sed -i '' "s|paste-your-bot-token-here|<token>|" .env
sed -i '' 's|"YOUR_CHANNEL_ID"|"<channel_id>"|' config.toml
# 視需要,啟用 DM:
sed -i '' 's|allow_dm = false|allow_dm = true|' config.toml

# 3. 啟動
docker compose pull
docker compose up -d

# 4. Claude OAuth(互動)
docker compose exec openab claude auth login
docker compose restart openab

# 5. 觀察
docker compose logs -f openab
```

---

## 對應的 Wiki 頁面 / Cross-References

| 主題 | Wiki 連結 |
|---|---|
| 全景 | [`../wiki/overview.md`](../wiki/overview.md) |
| 系統架構 | [`../wiki/architecture/system-overview.md`](../wiki/architecture/system-overview.md) |
| 訊息流(7 道濾鏡) | [`../wiki/architecture/message-flow.md`](../wiki/architecture/message-flow.md) |
| 安全模型(`env_clear` 等) | [`../wiki/architecture/security-model.md`](../wiki/architecture/security-model.md) |
| Discord adapter | [`../wiki/adapters/discord.md`](../wiki/adapters/discord.md) |
| Claude Code 細節 | [`../wiki/agents/agents.md`](../wiki/agents/agents.md) |
| Reaction FSM | [`../wiki/modules/reactions.md`](../wiki/modules/reactions.md) |
| Session pool | [`../wiki/concepts/session-pool.md`](../wiki/concepts/session-pool.md) |
| 多 agent 拓撲 | [`../wiki/concepts/multi-agent.md`](../wiki/concepts/multi-agent.md) |
| 訊息模式(DM / Channel / Thread / Multi-bot) | [`../wiki/concepts/messaging-model.md`](../wiki/concepts/messaging-model.md) |
| 完整 `config.toml` 參考 | [`../wiki/config/config-reference.md`](../wiki/config/config-reference.md) |

---

## 版本紀錄 / Changelog

| Date | Change |
|---|---|
| 2026-05-13 | Initial SOP. Verified end-to-end on macOS Apple Silicon + Docker Desktop. |
