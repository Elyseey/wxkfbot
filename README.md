# 微信客服 AI 助手 (WxKF Bot)

基于 Cloudflare Worker 构建的微信客服智能机器人，集成 OpenAI 兼容 API，支持多轮对话、管理后台、关键词 Webhook 触发等功能。

[![Deploy to Cloudflare Workers](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/bestk/wxkfbot)

![管理后台预览](images/preview.png)

## 功能特点

- 基于 Cloudflare Worker，无需服务器，全球边缘部署
- 集成 OpenAI 兼容 API（支持 GPT、DeepSeek、Claude 等任何兼容接口）
- 微信客服消息接收与自动回复，支持多轮对话上下文
- 内置消息加解密，完整对接企业微信回调
- 使用 Cloudflare KV 存储会话历史和配置
- 可视化管理后台（客服帐号、会话、消息、统计、系统设置）
- 关键词匹配触发 Webhook（支持精确/包含/正则，自定义请求体和变量替换）
- AI 模型/Base URL/API Key 可在管理后台动态配置，无需重新部署

## 前置准备

| 项目 | 说明 |
| --- | --- |
| Cloudflare 账号 | 最好已绑定可用域名（免费的 eu.org 也行） |
| GitHub 账号 | 用于一键部署，需关联上述 Cloudflare 账号 |
| 企业微信 | 你是该企业的管理员（无需认证） |
| OpenAI 兼容接口 | 支持 `/v1/chat/completions` 的任意服务（官方、第三方、new-api 均可） |
| Deno 账号（可选） | 用于部署加解密服务，也可直接使用公共服务 |

## 部署流程总览

```
1. 部署加解密服务（Deno Deploy） → 拿到 CRYPTO_SERVICE_URL
2. 一键部署到 Cloudflare Workers → 配置环境变量与 KV
3. 企业微信获取 CORP_ID、生成 Token 和 EncodingAESKey → 回填到 Workers
4. 企业微信完成回调 URL 验证 → 获取 Secret → 回填到 Workers
5. 验证对话功能
```

## 详细部署步骤

### 第一步：部署加解密服务（Deno）

> 如果不想自己部署，可直接使用公共服务：`https://wecom-crypto.deno.dev`

1. 打开项目中的 [`wecom_crypto_deno.ts`](./wecom_crypto_deno.ts)，复制全部内容
2. 打开 [Deno Deploy 控制台](https://dash.deno.com)
3. 点击 **New Playground**，将脚本内容粘贴进去，点击部署
4. 记录右上角的 Deno URL（例如 `https://your-name.deno.dev`）
5. 访问该 URL 确认可达（返回 `Method not allowed` 即表示正常）

### 第二步：部署到 Cloudflare Workers

#### 方式一：一键部署（推荐）

1. 点击本页顶部的 **Deploy to Cloudflare Workers** 按钮
2. 登录 GitHub 和 Cloudflare 账号
3. 在部署参数页配置环境变量和 KV 绑定（详见下方参数说明）
4. 部署完成后获得两个域名：
   - Workers 自带域名：`xxx.workers.dev`（国内访问不稳定，不建议作为主入口）
   - **自定义域名**（强烈建议绑定，例如 `kf.yourdomain.com`）

#### 方式二：手动部署

```bash
git clone https://github.com/bestK/wxkfbot.git
cd wxkfbot
npm install
```

创建 KV 命名空间：

```bash
wrangler kv:namespace create "CONVERSATIONS"
wrangler kv:namespace create "MESSAGE_TRACKER"
```

复制 `wrangler.toml.example` 为 `wrangler.toml`，填入配置后部署：

```bash
wrangler deploy
```

### 第三步：配置企业微信

#### 3.1 获取企业 ID

手机企业微信 → 工作台 → 管理企业 → 企业信息 → 复制企业 ID

#### 3.2 生成 Token 与 EncodingAESKey

1. 打开[微信客服管理后台](https://kf.weixin.qq.com)
2. 左侧进入「开发配置 → API 配置」
3. 点击生成 **Token** 和 **EncodingAESKey**
4. 复制这两个值，回到 Cloudflare Workers 填入对应变量

#### 3.3 设置回调 URL

1. 回调地址填写你的**自定义域名** + `/callback`
   - 例如：`https://kf.yourdomain.com/callback`
   - ⚠️ 不建议使用 `workers.dev` 域名，国内验证大概率失败
2. 点击保存/验证
3. 验证成功后会显示 **Secret**，将其填入 Workers 的 `WECHAT_KF_SECRET` 变量
4. **重新部署 Workers**

### 第四步：开始聊天

部署完成后，有两种方式发起对话：

**微信扫码体验**：客服后台 → 开始接入 → 在微信内其他场景接入 → 扫码体验

**外部链接接入**：客服后台 → 在微信外 App/网页中接入 → 复制客服链接，在微信中打开

## 环境变量详解

| 变量名 | 说明 | 必填 | 示例 |
| --- | --- | --- | --- |
| WECHAT_CORP_ID | 企业微信企业 ID | 是 | `wwxxxxxxxxx` |
| WECHAT_KF_SECRET | 客服 Secret（回调验证通过后获取） | 是 | |
| WECHAT_KF_TOKEN | 消息校验 Token | 是 | |
| WECHAT_KF_ENCODING_AES_KEY | 消息加解密 Key | 是 | |
| OPENAI_API_KEY | API 密钥 | 是 | `sk-xxx` |
| OPENAI_BASE_URL | API 地址，**只填到根路径** | 否 | `https://api.openai.com` |
| OPENAI_MODEL | 模型名称 | 否 | `gpt-4o` |
| OPENAI_TIMEOUT | 请求超时 ms | 否 | `30000` |
| SYSTEM_PROMPT | AI 系统提示词 | 否 | |
| CRYPTO_SERVICE_URL | 加解密服务地址 | 是 | `https://wecom-crypto.deno.dev` |
| ADMIN_KEY | 管理后台访问密钥 | 否 | |

> **注意**：`OPENAI_BASE_URL` 只填到根路径，不要带 `/v1/chat/completions`。例如 API 请求地址是 `https://api.example.com/v1/chat/completions`，这里就填 `https://api.example.com`。

> AI 模型、Base URL、API Key、系统提示词均可在管理后台动态修改，KV 中的配置优先于环境变量。

## KV 命名空间

| 绑定名称 | 用途 | 说明 |
| --- | --- | --- |
| CONVERSATIONS | 会话存储 | 存储对话历史和动态配置 |
| MESSAGE_TRACKER | 消息去重/追踪 | 防止重复处理消息 |

> 两个 KV 命名空间必须是不同的实例，不能共用。

## 管理后台

部署后访问 `https://your-domain.com/admin` 即可使用。

功能模块：

- **客服帐号** — 查看、添加、修改、删除微信客服帐号，获取客服链接
- **会话管理** — 实时查看会话列表，聊天窗口支持富媒体消息（图片/语音/视频/文件/链接/位置），支持手动回复和消息撤回
- **消息同步** — 同步微信客服历史消息
- **统计数据** — 按客服帐号查看统计
- **系统设置** — AI 模型配置、系统提示词、关键词 Webhook 规则

## 关键词 Webhook

在管理后台「系统设置」中配置关键词触发规则：

- 匹配方式：精确匹配、包含、正则表达式
- 请求方法：GET / POST
- Content-Type：JSON / Form
- 自定义请求体模板，支持变量替换

可用变量：

| 变量 | 说明 |
| --- | --- |
| `{{content}}` | 用户消息内容 |
| `{{external_userid}}` | 用户 ID |
| `{{open_kfid}}` | 客服 ID |
| `{{msgid}}` | 消息 ID |
| `{{keyword}}` | 匹配的关键词 |
| `{{timestamp}}` | 触发时间戳 |

GET 请求时变量可用于 URL 参数，POST 请求时用于请求体模板。

## 常见问题

### 企业微信回调验证失败

- **Workers 日志无请求记录**：域名不可达或被墙。不要用 `workers.dev`，改用自定义域名
- **有日志但报错**：检查 `WECHAT_KF_TOKEN` 和 `WECHAT_KF_ENCODING_AES_KEY` 是否与企业微信后台一致
- 确认回调 URL（`https://your-domain.com/callback`）可公网访问，TLS 正常

### 能收到消息但没有 AI 回复

- 检查 `OPENAI_BASE_URL` 是否只填到根路径（不要带 `/v1/chat/completions`）
- 检查 `OPENAI_API_KEY` 是否正确
- 检查 `OPENAI_MODEL` 名称是否正确
- 在 Cloudflare Workers 控制台查看实时日志排查

### Cloudflare 日志无输出

在 Workers 控制台开启 Logs，重新触发一条消息查看请求链路。

## 项目结构

```
wxkfbot/
├── index.js            # 主入口，路由和业务逻辑
├── config.js           # AI 配置管理
├── clients.js          # OpenAI / 微信 API 客户端
├── conversation.js     # 多轮对话管理
├── crypto.js           # 消息加解密
├── message-tracker.js  # 消息去重跟踪
├── response.js         # 统一响应格式
├── kf-management.js    # 微信客服管理 API 封装
├── kf-routes.js        # 客服管理路由处理
├── admin.js            # 管理后台（构建产物）
├── admin/              # 管理后台源码（Vue 3 + Element Plus）
└── scripts/
    └── build-admin.mjs # 前端构建脚本
```

## 本地开发

```bash
# 前端开发
cd admin && npm run dev

# Worker 本地调试
wrangler dev

# 构建管理后台
cd admin && npm run build && cd .. && node scripts/build-admin.mjs

# 部署
wrangler deploy
```

## 许可证

MIT License

## 链接

- 项目地址：[github.com/bestK/wxkfbot](https://github.com/bestK/wxkfbot)
- 部署教程：[linux.do 社区帖](https://linux.do/t/topic/991301)
- 问题反馈：[Issues](https://github.com/bestK/wxkfbot/issues)
