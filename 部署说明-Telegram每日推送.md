# TrendRadar 本地 Docker 部署 —— 每天早 7 点推送 Telegram

本仓库已**预配置好**，按下面 4 步即可在你自己的机器上跑起来。
全程不依赖 GitHub Actions，不受 7 天试用限制。

## 前提
一台能**长期开机联网**的机器（家用电脑/小服务器都行），并安装：
- **Docker** 和 **Docker Compose**
  - Windows / macOS：安装 [Docker Desktop](https://www.docker.com/products/docker-desktop/)
  - Linux：`curl -fsSL https://get.docker.com | sh`

## 第 1 步：克隆仓库
```bash
git clone https://github.com/123133331234/TrendRadar.git
cd TrendRadar
```

## 第 2 步：填入 Telegram 凭证（只在你本机操作，不要提交）
编辑 `docker/.env`，把这两行的 `=` 后面填上你的值：
```bash
TELEGRAM_BOT_TOKEN=你的BotToken
TELEGRAM_CHAT_ID=你的ChatID
```
> 定时已预设为 `CRON_SCHEDULE=0 7 * * *`（每天北京时间 7:00），无需改动。
> 安全提醒：token 不要提交到公开仓库；如曾泄露，去 @BotFather 用 `/revoke` 重置后再填。

## 第 3 步：启动
```bash
cd docker
docker compose up -d trendradar
```
启动时会**立刻跑一次**（`IMMEDIATE_RUN=true`），约 1 分钟内 Telegram 应收到第一条推送 ✅。
之后每天**北京时间 7:00** 自动推一条。

## 第 4 步：验证 / 排错
```bash
docker compose logs -f trendradar     # 看运行日志，确认推送成功
```
看到 Telegram 发送成功即完成。验证后可把 `docker/.env` 里的 `IMMEDIATE_RUN` 改为 `false`，再 `docker compose restart trendradar`。

---

## 已为你预设的配置
- `docker/.env`：`CRON_SCHEDULE=0 7 * * *`（每天 7:00）、`IMMEDIATE_RUN=true`、`RUN_MODE=cron`
- `config/config.yaml`：
  - `notification.enabled: true`（推送总开关）
  - `ai_analysis / ai_translation: false`（未配 AI Key，先关闭避免报错）
  - `rss: false`（关闭外文 RSS 源）
  - `report.mode: current`（早晨推当前在榜的匹配热点）

## 常用操作
- **改关心的关键词**：编辑 `config/frequency_words.txt`（每行一个词组）→ `docker compose restart trendradar`
- **改推送时间**：改 `docker/.env` 的 `CRON_SCHEDULE`（如 `0 9 * * *` = 每天 9:00）→ 重启
- **当日汇总模式**：把 `config/config.yaml` 的 `report.mode` 改为 `daily` → 重启
- **停止**：`docker compose down`
