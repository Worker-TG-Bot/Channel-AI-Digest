

<div align="center">
  <img src="./logo.png" alt="RSS Telegram Bot Logo" width="400">
  <h1> 📡 Channel AI Digest</h1>
  <p>一个基于 Cloudflare Workers 的 Telegram 频道消息自动摘要机器人。它能监听授权频道的消息，利用 Cloudflare Workers AI 自动生成中文总结与智能标签，并直接编辑原消息追加摘要内容。</p>
</div>

<br/>

---

## ✨ 功能特性

| 功能 | 说明 |
|------|------|
| 🤖 **AI 自动总结** | 监听频道新消息和编辑消息，自动生成中文摘要 |
| 🏷️ **智能标签** | 自动生成 1-3 个高相关度标签，遵循"互斥且穷尽"原则 |
| 🔗 **链接解析** | 通过 Jina Reader API 抓取 URL 内容参与总结 |
| 📋 **标签记忆** | 标签持久化到 KV，避免同义词堆砌（如 #科技 和 #技术 不会同时出现） |
| 🛡️ **授权管理** | 通过 Bot 的 Inline Keyboard 管理授权频道列表 |
| 🔄 **编辑检测** | 智能区分用户编辑和 Bot 自身编辑，避免无限循环 |
| 🧹 **聊天整洁** | 私聊消息自动删除，仅保留 Bot 的管理面板 |
| 📷 **媒体过滤** | 仅处理文本消息，纯图片/视频/贴纸等自动忽略 |

---

## 🏗️ 架构

```
Telegram Channel
       │
       ▼
Cloudflare Worker (webhook)
       │
       ├── 频道消息 ──► 授权检查 ──► 媒体过滤 ──► 提取标签/链接
       │                                              │
       │                                    Jina Reader (抓取链接)
       │                                              │
       │                                     Workers AI (总结)
       │                                              │
       │                                     KV (标签去重)
       │                                              │
       │                                   编辑原消息(追加摘要)
       │
       └── 私聊消息 ──► 管理员验证 ──► Inline Keyboard 管理面板
```

---

## 🚀 部署指南

### 方式一：一键部署（推荐）

点击下方按钮，按照引导完成部署：

[![Deploy to Cloudflare Workers](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/xiaobaiweinuli/channel-digest-ai)

部署完成后，还需要手动配置环境变量和 KV 绑定（见下方配置章节）。

### 方式二：手动部署

#### 1. 前置准备

- [Cloudflare 账号](https://dash.cloudflare.com/sign-up)
- [Node.js](https://nodejs.org/) >= 18
- 一个 [Telegram Bot Token](https://t.me/BotFather)

#### 2. 克隆仓库

```bash
git clone https://github.com/xiaobaiweinuli/channel-digest-ai.git
cd channel-digest-ai
```

#### 3. 安装依赖

```bash
npm install wrangler --save-dev
```

#### 4. 创建 KV 命名空间

```bash
npx wrangler kv namespace create KV
```

将输出的 `id` 填入 `wrangler.toml`：

```toml
[[kv_namespaces]]
binding = "KV"
id = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

#### 5. 配置环境变量

使用 `wrangler secret` 设置敏感变量（推荐）：

```bash
npx wrangler secret put BOT_TOKEN
npx wrangler secret put WEBHOOK_SECRET
npx wrangler secret put ADMIN_ID
npx wrangler secret put SETUP_PASSWORD
# 可选
npx wrangler secret put JINA_API_KEY
```

#### 6. 部署

```bash
npx wrangler deploy
```

#### 7. 设置 Webhook

部署成功后，使用 curl 一键设置 Bot 的 Webhook 和命令菜单：

```bash
curl https://your-worker.your-subdomain.workers.dev/setup/你的SETUP_PASSWORD
```

---

## ⚙️ 环境变量

| 变量名 | 必填 | 说明 |
|--------|------|------|
| `BOT_TOKEN` | ✅ | Telegram Bot Token，从 [@BotFather](https://t.me/BotFather) 获取 |
| `WEBHOOK_SECRET` | ✅ | Webhook 密钥，用于验证请求来源，随意设置一个复杂字符串 |
| `ADMIN_ID` | ✅ | 超级管理员的 Telegram 用户 ID（纯数字） |
| `SETUP_PASSWORD` | ✅ | 设置接口密码，用于 curl 初始化 |
| `JINA_API_KEY` | ❌ | [Jina Reader API](https://jina.ai/reader/) 密钥，不设置则使用免费 API |
| `AI_MODEL` | ❌ | Workers AI 模型名称，默认 `@cf/qwen/qwen2.5-coder-32b-instruct` |

> 💡 **获取你的 Telegram 用户 ID**：向 [@userinfobot](https://t.me/userinfobot) 发送任意消息即可获取。

---

## 📑 KV 绑定

本项目需要一个 Workers KV 命名空间，绑定名称必须为 `KV`。

**用途：**
- `channels` — 存储授权频道列表
- `tags` — 存储全局标签库（用于去重）
- `bot_edit:{chat_id}:{message_id}` — Bot 编辑标记（自动过期）
- `state:{user_id}` — 用户交互状态（自动过期）

在 Cloudflare Dashboard 中创建：
> Workers & Pages → KV → 创建命名空间 → 绑定到 Worker

---

## 🎮 使用方法

### 1. 管理授权频道

1. 向 Bot 发送 `/start` 或任意消息
2. 点击「📋 授权管理」按钮
3. 点击「➕ 添加授权」
4. 发送频道 ID（格式：`-100xxxxxxxxxx`）
5. Bot 会自动验证是否为频道，验证通过后添加到授权列表

> ⚠️ **注意**：添加前请确保已将 Bot 添加为频道管理员，并授予「编辑消息」权限。

### 2. 获取频道 ID

将频道消息转发给 [@userinfobot](https://t.me/userinfobot) 或使用类似的 ID 查询 Bot。

### 3. 自动总结

配置完成后，在授权频道中发送的文本消息将自动被 AI 总结。Bot 会编辑原消息，在末尾追加：

```
[原始消息内容]

---
📝 总结：[AI 生成的中文摘要]

🏷️ #标签1 #标签2
```

---

## 🔍 处理逻辑

```
收到频道消息
    │
    ├── 非频道类型？ ──► 忽略
    ├── 未授权频道？ ──► 忽略
    ├── 纯媒体消息？ ──► 忽略（图片/视频/贴纸/语音等）
    ├── Bot 自身编辑？ ──► 忽略（通过 KV 标记检测）
    │
    ▼
  有文本内容
    │
    ├── 提取已有 #标签 并移除
    ├── 分离原始内容（去掉之前的摘要部分）
    ├── 提取 URL → Jina Reader 抓取内容
    │
    ▼
  构建 AI Prompt
    │
    ├── 注入 KV 中的历史标签 + 原文标签
    ├── 调用 Workers AI 生成总结
    │
    ▼
  处理 AI 输出
    │
    ├── 提取新标签 → 去重合并 → 写入 KV
    ├── 组装最终消息（原文 + 摘要 + 标签）
    ├── 设置 Bot 编辑标记到 KV
    └── 调用 Telegram API 编辑原消息
```

---

## ❓ 常见问题

<details>
<summary><b>Bot 没有反应？</b></summary>

1. 确认已执行 `curl .../setup/密码` 设置 Webhook
2. 检查 Worker 日志中是否有报错
3. 确认 `BOT_TOKEN` 环境变量正确
</details>

<details>
<summary><b>频道消息没有被总结？</b></summary>

1. 确认 Bot 已被添加为频道管理员
2. 确认 Bot 拥有「编辑消息」权限
3. 确认频道已在授权列表中
4. 确认消息包含文本内容（纯媒体不处理）
</details>

<details>
<summary><b>出现无限编辑循环？</b></summary>

正常情况下 Bot 会通过 KV 标记检测自身编辑并跳过。如果出现循环：
1. 检查 KV 绑定是否正常
2. 查看 Worker 日志排查问题
</details>

<details>
<summary><b>如何更换 AI 模型？</b></summary>

设置环境变量 `AI_MODEL` 为其他 Cloudflare Workers AI 支持的模型名称，例如：
- `@cf/meta/llama-3.1-70b-instruct`
- `@cf/qwen/qwen2.5-72b-instruct`

完整列表见 [Workers AI 模型目录](https://developers.cloudflare.com/workers-ai/models/)。
</details>

---

## 📄 License

[MIT](LICENSE)

---

## 🙏 致谢

- [Cloudflare Workers](https://workers.cloudflare.com/) — Serverless 运行平台
- [Cloudflare Workers AI](https://ai.cloudflare.com/) — AI 推理服务
- [Jina Reader API](https://jina.ai/reader/) — 网页内容抓取
- [Telegram Bot API](https://core.telegram.org/bots/api) — Bot 接口
