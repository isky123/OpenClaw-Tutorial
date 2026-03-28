# 第5章：渠道接入实战

> 详细讲解飞书、微信、Telegram、Discord 等主流平台的接入方法

---

## 5.1 国内主流渠道接入

### 5.1.1 飞书机器人

飞书是字节跳动推出的企业协作平台，在国内企业用户中普及率很高。

**创建应用**

1. 访问 [飞书开放平台](https://open.feishu.cn/)
2. 点击「创建企业自建应用"
3. 填写应用信息：
   - 应用名称：OpenClaw助手
   - 应用描述：AI 智能助手
   - 应用头像：上传一个图标

**开启机器人能力**

1. 进入应用详情 → 「机器人」
2. 打开「启用机器人」开关
3. 设置：
   - 机器人名称：OpenClaw助手
   - 机器人描述：你的智能 AI 助手
   - 消息卡片：启用

**获取凭证**

1. 进入「凭证与基础信息」
2. 复制以下信息：
   - App ID：`cli_xxxxx`
   - App Secret：`xxxxx`

3. 进入「事件订阅」
4. 启用加密，复制：
   - Encrypt Key：`xxxxx`
   - Verification Token：`xxxxx`

**配置事件订阅**

在 OpenClaw 中添加飞书渠道：

```yaml
# 渠道配置
channels:
  - name: "飞书助手"
    type: feishu
    enabled: true
    config:
      app_id: "cli_xxxxx"
      app_secret: "xxxxx"
      encrypt_key: "xxxxx"
      verification_token: "xxxxx"
```

保存后，OpenClaw 会生成 Webhook URL：

```
https://your-domain.com/api/webhooks/feishu/{channel_id}
```

将此 URL 填入飞书「事件订阅」→「请求地址配置」。

**订阅事件**

添加以下事件：

```
im.message.receive_v1    # 接收消息
```

**配置权限**

在「权限管理」中添加：

```
contact:user.base:readonly      # 获取用户基本信息
im:chat:readonly                # 获取群组信息
im:message.group_msg:readonly   # 读取群组消息
im:message:send                 # 发送消息
im:message.p2p_msg:readonly     # 读取私聊消息
```

**发布应用**

1. 进入「版本管理与发布」
2. 创建版本 → 填写版本号 → 申请发布
3. 在飞书管理后台审批通过

**测试**

在飞书中搜索应用名称，添加后发送消息测试。

⚠️ **常见问题**：

1. **URL 验证失败**：确保 OpenClaw 可公网访问，SSL 证书有效
2. **收不到消息**：检查事件订阅和权限配置
3. **发送失败**：检查是否开通了 `im:message:send` 权限

---

### 5.1.2 钉钉机器人

钉钉是阿里巴巴的企业协作平台，接入相对简单。

**创建机器人**

1. 打开钉钉，进入要添加机器人的群聊
2. 群设置 → 「智能群助手」→ 「添加机器人」
3. 选择「自定义」→ 「添加"
4. 设置：
   - 机器人名字：OpenClaw助手
   - 安全设置：选择「加签」

**获取凭证**

创建后会显示：
- Webhook URL：`https://oapi.dingtalk.com/robot/send?access_token=xxxxx`
- 密钥（加签用）：`SECxxxxx`

提取 token：
```
access_token= 后面的部分
```

**OpenClaw 配置**

```yaml
channels:
  - name: "钉钉助手"
    type: dingtalk
    enabled: true
    config:
      access_token: "xxxxx"
      secret: "SECxxxxx"
```

钉钉机器人的消息格式：

```json
{
  "msgtype": "text",
  "text": {
    "content": "消息内容"
  },
  "at": {
    "atMobiles": [],
    "isAtAll": false
  }
}
```

⚠️ **限制说明**：

- 每分钟最多发送 20 条消息
- 消息内容超过 20KB 会被截断
- 不支持主动推送，只能被动响应

---

### 5.1.3 企业微信机器人

企业微信是腾讯的企业通讯工具，与微信互通。

**创建应用**

1. 访问 [企业微信管理后台](https://work.weixin.qq.com/)
2. 应用管理 → 「创建应用"
3. 填写应用信息，上传 logo

**获取凭证**

1. 进入应用详情
2. 复制：
   - AgentId：`1000002`
   - Secret：`xxxxx`

3. 进入「我的企业」
4. 复制：
   - 企业 ID：`wwxxxxx`

**配置接收消息**

1. 应用详情 → 「接收消息」→ 「设置"
2. 填写：
   - URL：`https://your-domain.com/api/webhooks/wecom/{channel_id}`
   - Token：随机生成
   - EncodingAESKey：随机生成

3. 复制 Token 和 EncodingAESKey 到 OpenClaw 配置

**OpenClaw 配置**

```yaml
channels:
  - name: "企业微信助手"
    type: wecom
    enabled: true
    config:
      corp_id: "wwxxxxx"
      agent_id: "1000002"
      secret: "xxxxx"
      token: "xxxxx"
      encoding_aes_key: "xxxxx"
```

**配置可信域名**

1. 应用详情 → 「网页授权及JS-SDK」
2. 设置可信域名：`your-domain.com`
3. 下载验证文件并上传到域名根目录

---

### 5.1.4 微信公众号

微信公众号分为订阅号和服务号，接入方式略有不同。

**前提条件**

- 已注册微信公众号
- 已完成微信认证（服务号必须认证）
- 有服务器和域名

**配置服务器**

1. 登录 [微信公众平台](https://mp.weixin.qq.com/)
2. 开发 → 基本配置 → 「服务器配置」→ 「修改配置"
3. 填写：
   - URL：`https://your-domain.com/api/webhooks/wechat-mp/{channel_id}`
   - Token：自定义（与 OpenClaw 配置一致）
   - EncodingAESKey：随机生成
   - 消息加解密方式：明文/兼容/安全模式

**OpenClaw 配置**

```yaml
channels:
  - name: "公众号助手"
    type: wechat_mp
    enabled: true
    config:
      app_id: "wx_xxxxx"
      app_secret: "xxxxx"
      token: "your_token"
      encoding_aes_key: "xxxxx"  # 安全模式需要
      encrypt_mode: "compatible"  # plaintext/compatible/safe
```

**获取 AccessToken**

微信公众号需要定期刷新 access_token：

```python
import requests

def get_access_token(app_id: str, app_secret: str) -> str:
    url = f"https://api.weixin.qq.com/cgi-bin/token"
    params = {
        "grant_type": "client_credential",
        "appid": app_id,
        "secret": app_secret
    }
    resp = requests.get(url, params=params)
    return resp.json()["access_token"]
```

⚠️ **重要**：access_token 有效期 2 小时，需要定时刷新。

---

### 5.1.5 个人微信（WeChaty）

⚠️ **风险提示**：使用个人微信接入存在封号风险，建议仅用于个人测试，生产环境请使用企业微信或公众号。

**WeChaty 方案**

WeChaty 是一个开源的微信机器人 SDK，支持多种协议：

```yaml
channels:
  - name: "个人微信"
    type: wechaty
    enabled: true
    config:
      protocol: "puppet_padlocal"  # 或 puppet_wechat
      token: "your_padlocal_token"  # 需要购买
```

**PadLocal 协议**

PadLocal 是稳定的 iPad 协议实现：

1. 访问 [PadLocal](http://pad-local.com/) 购买 token
2. 在配置中填入 token
3. 扫码登录

**免费方案（web 协议）**

```yaml
channels:
  - name: "个人微信"
    type: wechaty
    enabled: true
    config:
      protocol: "puppet_wechat"
      # 无需 token，但稳定性较差，容易被限制
```

---

## 5.2 海外主流渠道接入

### 5.2.1 Telegram Bot

Telegram 是海外最流行的即时通讯工具之一，Bot API 非常友好。

**创建 Bot**

1. 在 Telegram 中搜索 `@BotFather`
2. 发送 `/newbot`
3. 按提示设置：
   - 机器人名称：OpenClaw助手
   - 用户名：openclaw_bot（必须以 bot 结尾）

4. 获得 Token：`123456789:ABCdefGHIjklMNOpqrsTUVwxyz`

**配置 Webhook**

Telegram 支持轮询和 Webhook 两种方式，生产环境推荐使用 Webhook：

```bash
# 设置 Webhook
curl -X POST "https://api.telegram.org/bot<TOKEN>/setWebhook" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-domain.com/api/webhooks/telegram/{channel_id}",
    "allowed_updates": ["message", "callback_query"]
  }'
```

**OpenClaw 配置**

```yaml
channels:
  - name: "Telegram助手"
    type: telegram
    enabled: true
    config:
      bot_token: "123456789:ABCdefGHIjklMNOpqrsTUVwxyz"
      webhook_url: "https://your-domain.com/api/webhooks/telegram/{channel_id}"
      # 或使用轮询模式（开发环境）
      # polling: true
```

**支持的富媒体**

Telegram 支持丰富的消息类型：

```python
# 发送图片
bot.send_photo(chat_id, photo=open('image.jpg', 'rb'))

# 发送文档
bot.send_document(chat_id, document=open('file.pdf', 'rb'))

# 发送 Markdown
bot.send_message(
    chat_id, 
    text="*粗体* _斜体_ `代码`",
    parse_mode="Markdown"
)

# 发送按钮
keyboard = [["按钮1", "按钮2"]]
bot.send_message(
    chat_id,
    text="请选择：",
    reply_markup={"keyboard": keyboard, "resize_keyboard": True}
)
```

---

### 5.2.2 Discord Bot

Discord 是海外游戏和社区常用的聊天平台。

**创建应用**

1. 访问 [Discord Developer Portal](https://discord.com/developers/applications)
2. 点击「New Application」
3. 填写应用名称

**创建 Bot**

1. 进入应用 → 「Bot」→ 「Add Bot"
2. 复制 Token（只显示一次，务必保存）

**获取权限**

1. 进入「OAuth2」→ 「URL Generator"
2. 选择 Scopes：`bot`
3. 选择 Bot Permissions：
   - Send Messages
   - Read Message History
   - Mention Everyone

4. 复制生成的 URL，在浏览器中打开，邀请 Bot 加入服务器

**OpenClaw 配置**

```yaml
channels:
  - name: "Discord助手"
    type: discord
    enabled: true
    config:
      bot_token: "xxxxx"
      # Discord 使用 Gateway 连接，无需 Webhook
```

**Discord 特性**

- 支持富文本（Markdown）
- 支持 Embed（卡片消息）
- 支持 Slash Command（/命令）
- 支持线程（Thread）

---

### 5.2.3 Slack App

Slack 是企业协作平台，Bot 开发体验优秀。

**创建 App**

1. 访问 [Slack API](https://api.slack.com/apps)
2. 点击「Create New App」→ 「From scratch"
3. 填写应用名称，选择工作区

**配置权限**

1. 进入「OAuth & Permissions」
2. 添加 Bot Token Scopes：
   - `chat:write` - 发送消息
   - `im:history` - 读取私聊历史
   - `channels:history` - 读取频道历史
   - `app_mentions:read` - 读取提及

3. 点击「Install to Workspace」
4. 复制 Bot User OAuth Token：`xoxb-xxxxx`

**配置事件订阅**

1. 进入「Event Subscriptions」→ 启用
2. 填写 Request URL：`https://your-domain.com/api/webhooks/slack/{channel_id}`
3. 订阅 Bot Events：
   - `message.im` - 私聊消息
   - `message.channels` - 频道消息
   - `app_mention` - 提及机器人

**OpenClaw 配置**

```yaml
channels:
  - name: "Slack助手"
    type: slack
    enabled: true
    config:
      bot_token: "xoxb-xxxxx"
      signing_secret: "xxxxx"  # 从 Basic Info → Signing Secret 获取
```

---

### 5.2.4 WhatsApp Business API

WhatsApp Business API 需要通过 Meta 官方或合作伙伴（如 360dialog）接入。

**通过 360dialog 接入**

1. 注册 [360dialog](https://www.360dialog.com/) 账号
2. 创建 WhatsApp Business API 账号
3. 获取 API Key

**OpenClaw 配置**

```yaml
channels:
  - name: "WhatsApp助手"
    type: whatsapp
    enabled: true
    config:
      provider: "360dialog"
      api_key: "xxxxx"
      webhook_url: "https://your-domain.com/api/webhooks/whatsapp/{channel_id}"
```

⚠️ **限制**：

- 需要商业账号
- 有消息模板审核机制
- 按消息收费

---

## 5.3 Webhook 配置原理

### 5.3.1 签名验证机制

各平台使用不同的签名算法验证请求合法性：

**飞书签名验证**

```python
import hmac
import hashlib
import base64

def verify_feishu_signature(encrypt_key: str, timestamp: str, nonce: str, body: str, signature: str) -> bool:
    """验证飞书请求签名"""
    # 拼接字符串
    raw = f"{timestamp}{nonce}{encrypt_key}{body}"
    
    # SHA256 哈希
    computed = hashlib.sha256(raw.encode()).hexdigest()
    
    return computed == signature
```

**钉钉签名验证**

```python
import hmac
import hashlib
import base64

def verify_dingtalk_signature(secret: str, timestamp: str, signature: str) -> bool:
    """验证钉钉请求签名"""
    # 拼接字符串
    raw = f"{timestamp}\n{secret}"
    
    # HMAC-SHA256
    mac = hmac.new(secret.encode(), raw.encode(), hashlib.sha256)
    computed = base64.b64encode(mac.digest()).decode()
    
    return computed == signature
```

**Slack 签名验证**

```python
import hmac
import hashlib

def verify_slack_signature(signing_secret: str, timestamp: str, body: str, signature: str) -> bool:
    """验证 Slack 请求签名"""
    # 拼接字符串
    raw = f"v0:{timestamp}:{body}"
    
    # HMAC-SHA256
    mac = hmac.new(signing_secret.encode(), raw.encode(), hashlib.sha256)
    computed = f"v0={mac.hexdigest()}"
    
    return computed == signature
```

### 5.3.2 本地调试技巧

**使用 ngrok**

```bash
# 安装 ngrok
brew install ngrok

# 配置 token
ngrok config add-authtoken your_token

# 启动穿透
ngrok http 3000
```

**使用 Cloudflare Tunnel（免费稳定）**

```bash
# 安装 cloudflared
brew install cloudflared

# 登录
cloudflared tunnel login

# 创建隧道
cloudflared tunnel create openclaw

# 配置并运行
cloudflared tunnel route dns openclaw your-domain.com
cloudflared tunnel run openclaw
```

---

## 5.4 本章小结

本章详细讲解了主流聊天平台的接入方法：

| 平台 | 难度 | 特点 |
|------|------|------|
| 飞书 | ⭐⭐ | 企业友好，功能丰富 |
| 钉钉 | ⭐ | 接入简单，功能有限 |
| 企业微信 | ⭐⭐⭐ | 与微信互通，配置复杂 |
| 微信公众号 | ⭐⭐⭐ | 需要认证，有 access_token 机制 |
| 个人微信 | ⭐⭐⭐⭐ | 有封号风险，不推荐生产使用 |
| Telegram | ⭐ | API 友好，功能强大 |
| Discord | ⭐⭐ | 社区友好，支持富媒体 |
| Slack | ⭐⭐ | 企业协作，开发体验好 |
| WhatsApp | ⭐⭐⭐ | 需要商业账号，按量付费 |

**接入通用流程**：

1. 在平台创建应用/机器人
2. 获取凭证（App ID、Token、Secret 等）
3. 配置 Webhook URL
4. 在 OpenClaw 中添加渠道配置
5. 测试并验证

---

## 参考配置汇总

```yaml
# config/channels.yaml
channels:
  # 国内渠道
  - name: "飞书助手"
    type: feishu
    enabled: true
    config:
      app_id: "${FEISHU_APP_ID}"
      app_secret: "${FEISHU_APP_SECRET}"
      encrypt_key: "${FEISHU_ENCRYPT_KEY}"
      verification_token: "${FEISHU_VERIFICATION_TOKEN}"
  
  - name: "钉钉助手"
    type: dingtalk
    enabled: true
    config:
      access_token: "${DINGTALK_ACCESS_TOKEN}"
      secret: "${DINGTALK_SECRET}"
  
  # 海外渠道
  - name: "Telegram助手"
    type: telegram
    enabled: true
    config:
      bot_token: "${TELEGRAM_BOT_TOKEN}"
  
  - name: "Discord助手"
    type: discord
    enabled: true
    config:
      bot_token: "${DISCORD_BOT_TOKEN}"
  
  - name: "Slack助手"
    type: slack
    enabled: true
    config:
      bot_token: "${SLACK_BOT_TOKEN}"
      signing_secret: "${SLACK_SIGNING_SECRET}"
```
