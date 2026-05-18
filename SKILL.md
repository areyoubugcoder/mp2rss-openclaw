---
name: mp2rss
description: |
  Mp2rss CLI Skill —— 微信公众号转 RSS 订阅与文章管理。

  **当以下情况使用此 Skill**：
  (1) 用户要订阅 / 取消订阅 / 列出 / 搜索公众号：「订阅这个公众号 https://mp.weixin.qq.com/s/...」「我订阅了哪些公众号」「列一下我的公众号 RSS」「搜一下我订阅的财经类公众号」「取消订阅 X」「把 X 从订阅里删了」
  (2) 用户要查公众号历史文章：「X 这个号最近发了什么」「拉一下 X 的文章」「<X 号> 的历史文章」
  (3) 用户要管理登录态：「登录公众号 RSS 服务」「登出公众号 RSS」「我的 Feed Key 是什么」「我在公众号订阅工具里登录了吗」「在 mp2rss 里登录」
  (4) 用户要安装 / 配置 Mp2rss CLI：「装 mp2rss」「升级 mp2rss」「mp2rss 配置在哪里」
version: 0.1.0
metadata:
  openclaw:
    requires:
      bins:
        - mp2rss
    envVars:
      - name: MP2RSS_FEED_KEY
        required: false
        description: Mp2rss Feed Key；也可由 `mp2rss auth login` 写入配置文件，二者任一即可。
      - name: MP2RSS_API_URL
        required: false
        description: Mp2rss API URL，默认 https://mp2rss.bugcode.dev。
    homepage: https://mp2rss.bugcode.dev
    emoji: "📡"
---

# Mp2rss Skill

通过 Mp2rss CLI（Go 二进制 `mp2rss`）将微信公众号转成 RSS / JSON Feed 并管理订阅。本 skill 是路由入口，按用户意图分发到 `references/` 子文档读取详细命令规格。

## ⚠️ Agent 必读约束

### 🔧 运行时前置

所有命令通过本地 `mp2rss` 二进制调用，**不直接打 HTTP**。Agent 不应自己拼 API 请求，统一走 CLI 子命令 + `-o json` 解析。

**Base URL 由 CLI 自动决定**（命令行 flag > 环境变量 > 配置文件 > 默认 `https://mp2rss.bugcode.dev`），Agent 无需关心。

### 🔑 凭证与登录态

调用任何 `mp2rss mp` 子命令前，**必须先确认登录态**：

```bash
mp2rss auth status -o json
```

返回 `{"loggedIn": false}` 时停止调用 mp 子命令，引导用户跑 `mp2rss auth login`，详见 [references/auth.md](references/auth.md)。

凭证优先级（高 → 低）：命令行 `--api-key` flag > `MP2RSS_FEED_KEY` 环境变量 > `~/.mp2rss/config.json` 配置文件。

### 🔢 mpId 是 int64

公众号 ID 是 64 位整数，超出 JavaScript `Number.MAX_SAFE_INTEGER`。Agent 在 JS 环境下解析 `mp list -o json` / `mp articles -o json` 输出时**始终把 mpId 当字符串处理**：

```javascript
const safe = text.replace(/"(mpId)"\s*:\s*(\d+)/g, '"$1":"$2"');
const data = JSON.parse(safe);
```

Python / Go / jq 原生支持大整数，无此问题。

### 🚫 反幻觉边界

- **禁止编造 mpId**：所有 mpId 必须来自 `mp list` / `mp search` 的响应，不得凭空构造
- **禁止跳过订阅参数校验**：`mp subscribe <url>` 的 `<url>` **必须**是 `https://mp.weixin.qq.com/s/...` 文章链接；不是公众号名、不是二维码、不是公众号主页。识别不到合法文章 URL 时**反问用户索要任意一篇文章链接**，不要直接尝试
- **禁止伪造执行结果**：不调用 CLI 不得告诉用户「已订阅」「已删除」
- **禁止忽略 exit code**：CLI 返回非 0 必须解析 stderr / JSON envelope 并报告用户

### 🔄 错误处理

CLI 错误统一 JSON envelope（见 [references/errors.md](references/errors.md)）：

```json
{"error": {"message": "...", "code": <int>}}
```

Exit code 速查：

| Code | 含义 | Agent 处理 |
|------|------|-----------|
| 0 | 成功 | 解析 stdout |
| 1 | 通用错误（网络） | 报告 + 建议稍后重试 |
| 2 | 参数错误 | 检查参数；若是 `mp subscribe` 检查 URL 格式 |
| 3 | 鉴权失败 | 引导跑 `mp2rss auth login`（见 [auth.md](references/auth.md)） |
| 4 | 资源不存在 | mpId 错或文章 URL 失效 |
| 5 | 上游不可用 | 报告 + 建议稍后重试 |

---

## 执行流程概览

```
用户意图 → 路由匹配 → 读对应 references/xxx.md → 构造 mp2rss 子命令 → 执行 → 解析 JSON / 验证 exit code → 返回结果
                                                                                      ↓
                                                                                  exit ≠ 0 → 按错误码分支处理
```

**关键原则**

- **CLI 输出是真理**：所有状态以 `mp2rss xxx -o json` 返回为准，不依赖上下文记忆
- **`-o json` 优先**：批量 / 结构化场景统一加 `-o json`（除 `auth login` 不支持外）
- **mpId 字符串化**：JS 环境解析前先正则替换为字符串

---

## 指令路由表

| 用户意图 / 自然语言 | 角色 | 详细文档 |
|------|------|---------|
| 「装 mp2rss」「升级 mp2rss」「mp2rss 配置在哪」 | ⚙️ 安装/配置 | [references/install.md](references/install.md) |
| 「登录」「登出」「我的 Feed Key 是什么」「我登录了吗」 | 🔐 认证 | [references/auth.md](references/auth.md) |
| 「订阅这个公众号 https://mp.weixin.qq.com/s/...」 | 📡 订阅 | [references/mp.md](references/mp.md#subscribe) |
| 「我订阅了哪些公众号」「列一下我的 RSS」 | 📋 列表 | [references/mp.md](references/mp.md#list) |
| 「搜一下我订阅的财经类公众号」 | 🔍 搜索 | [references/mp.md](references/mp.md#search) |
| 「取消订阅 X」「把 X 从订阅里删了」 | 🗑️ 取消订阅 | [references/mp.md](references/mp.md#remove) |
| 「X 这个号最近发了什么」「拉一下 X 的文章」 | 📰 文章 | [references/mp.md](references/mp.md#articles) |
| 错误码 / JSON envelope / exit code 处理 | 🚨 错误 | [references/errors.md](references/errors.md) |

---

## 自然语言路由（细分 trigger）

```
包含 mp.weixin.qq.com/s/ URL          → mp subscribe（参数 = 该 URL）
「订阅」+ 任何 URL 形态                → 先校验是不是 mp.weixin.qq.com/s/...，否则反问
「订阅了哪些」「列一下」「我的公众号」  → mp list
「搜」「找一下」「带 关键词 的」        → mp list -q <kw> 或 mp search <kw>
「取消订阅」「删了」「不想要 X 了」     → 先 mp list -q X 确认 mpId，再 mp remove <mpId>
「最近的文章」「历史文章」「发了什么」  → mp articles <mpId>（先列表确认 mpId）
「登录」「授权」                       → auth login（参考 auth.md 选三种登录模式之一）
「我的 Key」「登录了吗」「状态」        → auth status -o json
「登出」「退出登录」                   → auth logout
```

**决策原则**：

- 有 `mp.weixin.qq.com/s/` URL → 直接 `mp subscribe`
- 用户给的是公众号名而非 URL → **反问索要文章 URL**，不要先 `mp search`（搜索是用于在已订阅源里找，不会订阅新号）
- 取消订阅前**先用 list/search 拿到准确 mpId**，避免误删

---

## API 路由（CLI 子命令总表）

| 子命令 | 用途 | 详细文档 |
|--------|------|---------|
| `mp2rss auth login [-k <key>] [--no-browser]` | 登录（三种模式） | [auth.md](references/auth.md#log-in) |
| `mp2rss auth status [-o json]` | 查询登录态 | [auth.md](references/auth.md#check-status) |
| `mp2rss auth logout` | 登出 | [auth.md](references/auth.md#log-out) |
| `mp2rss mp subscribe <article-url> [-o json]` | 订阅公众号 | [mp.md](references/mp.md#subscribe) |
| `mp2rss mp list [-q <kw>] [-p <page>] [--page-size <n>] [-o json]` | 列出订阅 | [mp.md](references/mp.md#list) |
| `mp2rss mp search <keyword> [-o json]` | 搜索已订阅源 | [mp.md](references/mp.md#search) |
| `mp2rss mp remove <mpId> [-y] [-o json]` | 取消订阅 | [mp.md](references/mp.md#remove) |
| `mp2rss mp articles <mpId> [-p <page>] [--page-size <n>] [-o json]` | 查公众号文章 | [mp.md](references/mp.md#articles) |
| `mp2rss update [--check]` | 升级 CLI | [install.md](references/install.md#升级) |

---

## 全局 flag

所有子命令都支持：

| Flag | 等价环境变量 | 说明 |
|------|-------------|------|
| `-o, --output <table\|json>` | — | 输出格式，默认 `table` |
| `--api-key <feed-key>` | `MP2RSS_FEED_KEY` | 覆盖 Feed Key |
| `--api-url <url>` | `MP2RSS_API_URL` | 覆盖 API 地址 |

优先级（高 → 低）：CLI flag > env > 配置文件 > 默认。
