# TALKCRAB Kepong Operation Agent

给 kepong-operation dashboard 加一个 AI 助理：查数据、登记 petty cash、每天自动预警。

## 1. 装依赖

在你的 Vercel 项目根目录：

```bash
npm install @anthropic-ai/sdk googleapis
```

## 2. 环境变量（Vercel 项目 Settings → Environment Variables）

| 变量名 | 说明 |
|---|---|
| `ANTHROPIC_API_KEY` | 你的 Anthropic API key（console.anthropic.com 申请） |
| `KEPONG_SHEET_ID` | 你的 kepong-operation Google Sheet 的 ID（网址中 `/d/` 和 `/edit` 之间那串） |
| `GOOGLE_SERVICE_ACCOUNT_JSON` | Google Service Account 的完整 JSON key（整个塞进去，一行） |
| `TELEGRAM_BOT_TOKEN` | 跟 @BotFather 建 bot 拿到的 token（用于预警推送） |
| `TELEGRAM_CHAT_ID` | 你自己/群组的 chat id |
| `CRON_SECRET` | 随便设一串密码，防止 cron 端点被外部乱调用 |

### Google Service Account 怎么建（5 分钟）
1. console.cloud.google.com → 建一个 project（或用现有的）
2. 开启 Google Sheets API
3. IAM & Admin → Service Accounts → Create → 下载 JSON key
4. 打开你的 Google Sheet，"共享" 里加入这个 service account 的 email，给 Editor 权限
5. 把整份 JSON 内容贴进 `GOOGLE_SERVICE_ACCOUNT_JSON` 环境变量

### Telegram Bot 怎么建（2 分钟）
1. Telegram 搜 @BotFather → `/newbot` → 拿到 token
2. 跟你的新 bot 说句话，然后开 `https://api.telegram.org/bot<TOKEN>/getUpdates` 找到你的 `chat.id`

## 3. 改 lib/tools.js 里的 range

现在写的是假设你的 sheet 有 `Sales`、`PettyCash`、`FoodCost` 三个 tab，栏位是我猜的，
你要对照你实际 kepong-operation 表格的 tab 名字和栏位顺序改一下 `readRange`/`appendRow` 里的 range 字符串。

## 4. 部署

```bash
vercel --prod
```

Cron 会自动照 `vercel.json` 里的排程跑（目前设的是 UTC 15:00 = 马来西亚时间晚上 11 点，
如果想改时间，去改 `vercel.json` 里的 cron schedule，注意 Vercel cron 用的是 UTC，
马来西亚是 UTC+8，比如想晚上 10 点跑就设 `"0 14 * * *"`）。

## 5. 嵌入前端

在你 kepong-operation dashboard 的 HTML 里加一行：

```html
<script src="/agent-widget.js"></script>
```

右下角会出现一个 🦀 浮动按钮，点开就是聊天框。

## 6. Dashboard 主版面预警面板

除了 Telegram 推播，也可以在 dashboard 主页面直接显示预警状态：

1. 在你的 Google Sheet 新增一个 tab，命名为 `AlertsLog`，第一行留空或加表头 `timestamp | message | status`
2. cron 每次检查完，除了发 Telegram，也会把结果写进这个 tab
3. 把 `public/alerts-panel.html` 整块内容贴进你 kepong-operation dashboard 主页面 HTML 里想要的位置
4. 面板每 60 秒自动读取 `/api/get-alerts` 刷新一次，红色⚠️是异常、绿色✅是"检查过、正常"

这样效果是：Telegram 负责你人不在电脑前也能收到通知，dashboard 面板负责你打开网页时一眼看到最新状态，两边数据同源（都来自 `AlertsLog` tab），不会对不上。

## 7. 之后可以扩充的工具

现在给了 4 个工具（查sales/查petty cash/登记petty cash/查food cost），
之后想加"查库存"、"改排班"之类，照着 `lib/tools.js` 的格式加 schema + `executeTool` 里加一个 case 就行，
不用动 `api/agent.js` 的逻辑。

## 安全提醒

- 这个 widget 现在没有登入验证，谁能开你的 dashboard 网页就能用这个 agent 改数据。
  如果 dashboard 本身已经有 PIN 验证，建议把 `/api/agent` 也加上同样的 PIN 检查（从 request header 传 PIN，
  在 `api/agent.js` 开头验证），不然等于绕过了你原本的权限控制。
- Google Service Account JSON、Anthropic API key 都不要 commit 进 git，只放在 Vercel 环境变量。
