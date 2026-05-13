# OpenAB + Claude Code + Discord — 本機 Docker 部署

這個資料夾是一份**最小可運行**的 OpenAB 部署:

- Agent backend:**Claude Code**(`@anthropic-ai/claude-code` + `claude-agent-acp` adapter)
- Platform:**Discord**(native adapter,outbound-only,不需要 public URL)
- Auth:Claude OAuth device flow(不需要 ANTHROPIC_API_KEY)
- 跑在:本機 Docker(`ghcr.io/openabdev/openab-claude:latest`)

## 檔案

| File | 用途 |
|---|---|
| `docker-compose.yml` | 一個 service:openab-claude container + 一個 named volume(`claude-data`)持久化 OAuth tokens |
| `config.toml` | OpenAB 設定:`[discord]` + `[agent]` (claude-agent-acp) + `[pool]` + `[reactions]` + `[markdown]` |
| `.env.example` | `DISCORD_BOT_TOKEN` 範本 — 複製成 `.env` 填入真實 token |
| `.gitignore` | 確保 `.env` 不會被 commit |

## 安裝步驟

詳細逐步流程見根目錄的 wiki:[`../wiki/`](../wiki/)。本資料夾的 README 是 cheat sheet。

### 1. 確認 Docker Desktop 正在跑

```bash
docker --version
docker info | head -5
```

### 2. 建立 Discord bot(在 Discord Developer Portal 手動完成)

依照 `../docs/discord.md` 走完:
- 建立 Application → Bot
- 開啟 Privileged Gateway Intents(Message Content + Server Members)
- Reset Token → 複製
- OAuth2 → URL Generator → bot scope + 必要權限 → 邀請到你的 server
- 在 Discord client 開「開發者模式」→ 右鍵頻道 → 複製頻道 ID

### 3. 填寫 `.env` 與 `config.toml`

```bash
cp .env.example .env
# 編輯 .env,把 paste-your-bot-token-here 改成真正的 bot token

# 編輯 config.toml,把 YOUR_CHANNEL_ID 改成你要 bot 進駐的頻道 ID
```

### 4. 啟動 container

```bash
docker compose up -d
docker compose logs -f openab   # 觀察啟動 log,Ctrl-C 離開
```

第一次啟動會看到 Claude OAuth 還沒設定 — 這是正常的,下一步處理。

### 5. 進去 container 跑 Claude OAuth

```bash
docker compose exec openab claude auth login
```

照螢幕指示複製一段 URL 到瀏覽器,登入 Claude.ai 並貼回 code。Token 會存到
`claude-data` volume,**重啟 container 不會掉**。

### 6. 重啟讓 OpenAB 讀新 token

```bash
docker compose restart openab
docker compose logs -f openab
```

看到 `discord: ready` 等字樣就成功了。

### 7. 在 Discord 測試

到你 allowlist 的頻道:

```
@YourBot hi
```

Bot 會建立一個 thread,在 thread 裡聊天就不用每次 @mention 了。

## 常用維運指令

| 用途 | 指令 |
|---|---|
| 看即時 log | `docker compose logs -f openab` |
| 重啟 | `docker compose restart openab` |
| 重 pull 最新 image | `docker compose pull && docker compose up -d` |
| 進去 shell | `docker compose exec openab bash` |
| 看 Claude 認證狀態 | `docker compose exec openab claude auth status` |
| 重新認證 Claude | `docker compose exec openab claude auth login` |
| 停掉(保留 volume) | `docker compose down` |
| 完全清掉(含 OAuth!) | `docker compose down -v` |

## 設定參考

完整 `config.toml` 欄位:[`../wiki/config/config-reference.md`](../wiki/config/config-reference.md)

Discord adapter 細節:[`../wiki/adapters/discord.md`](../wiki/adapters/discord.md)

Claude Code agent 細節:[`../wiki/agents/agents.md`](../wiki/agents/agents.md)
