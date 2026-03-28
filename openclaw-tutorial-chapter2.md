# 第2章：快速体验（5分钟上手）

> 使用 Docker Compose 一键启动 OpenClaw，完成第一个渠道接入

---

## 2.1 环境准备

在开始之前，请确保你的环境满足以下要求：

### 系统要求

| 组件 | 最低要求 | 推荐配置 |
|------|----------|----------|
| CPU | 2 核 | 4 核及以上 |
| 内存 | 4 GB | 8 GB 及以上 |
| 磁盘 | 20 GB | 50 GB 及以上 |
| 网络 | 可访问互联网 | 有公网 IP 更佳 |

### 软件依赖

- **Docker**：20.10.0 或更高版本
- **Docker Compose**：2.0.0 或更高版本
- **Git**：用于拉取配置文件

### 检查环境

```bash
# 检查 Docker 版本
docker --version
# 输出示例：Docker version 24.0.7, build afdd53b

# 检查 Docker Compose 版本
docker compose version
# 输出示例：Docker Compose version v2.23.0

# 检查 Docker 服务状态
docker info
```

⚠️ **国内用户注意**：如果 Docker 拉取镜像速度很慢，建议先配置镜像加速。编辑 `/etc/docker/daemon.json`（Linux）或 Docker Desktop 设置（Windows/Mac）：

```json
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com",
    "https://mirror.baidubce.com"
  ]
}
```

配置后重启 Docker 服务：

```bash
# Linux
sudo systemctl restart docker

# Mac/Windows: 重启 Docker Desktop
```

---

## 2.2 一键启动 OpenClaw

### 步骤1：创建项目目录

```bash
# 创建并进入项目目录
mkdir -p ~/openclaw && cd ~/openclaw

# 创建数据目录（用于持久化存储）
mkdir -p data/postgres data/redis data/uploads
```

### 步骤2：创建 Docker Compose 配置

创建 `docker-compose.yml` 文件：

```yaml
version: '3.8'

services:
  # PostgreSQL 数据库
  postgres:
    image: postgres:15-alpine
    container_name: openclaw-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: openclaw
      POSTGRES_PASSWORD: openclaw_password
      POSTGRES_DB: openclaw
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U openclaw"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis 缓存
  redis:
    image: redis:7-alpine
    container_name: openclaw-redis
    restart: unless-stopped
    volumes:
      - ./data/redis:/data
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # OpenClaw 核心服务
  openclaw:
    image: openclaw/openclaw:latest
    container_name: openclaw-core
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      # 数据库配置
      DATABASE_URL: postgresql://openclaw:openclaw_password@postgres:5432/openclaw
      REDIS_URL: redis://redis:6379/0
      
      # 应用配置
      APP_ENV: production
      APP_PORT: 3000
      APP_HOST: 0.0.0.0
      
      # 安全密钥（部署时务必修改！）
      JWT_SECRET: your-jwt-secret-key-change-this
      ENCRYPTION_KEY: your-encryption-key-32chars-long!!
      
      # 默认 LLM 配置（可选，可在 Web UI 中配置）
      # OPENAI_API_KEY: sk-your-openai-api-key
      
    ports:
      - "3000:3000"
    volumes:
      - ./data/uploads:/app/uploads
      - ./logs:/app/logs
```

💡 **提示**：上面的配置使用了默认密码，生产环境部署时务必修改 `JWT_SECRET` 和 `ENCRYPTION_KEY`！

### 步骤3：启动服务

```bash
# 在后台启动所有服务
docker compose up -d

# 查看启动日志
docker compose logs -f
```

等待看到类似以下的日志，表示启动成功：

```
openclaw-core  | ✓ Server started successfully
openclaw-core  | ✓ Database connected
openclaw-core  | ✓ Redis connected
openclaw-core  | 
openclaw-core  | 🚀 OpenClaw is running at http://0.0.0.0:3000
```

按 `Ctrl+C` 退出日志查看。

### 步骤4：访问 Web UI

打开浏览器，访问：

```
http://localhost:3000
```

或者如果你部署在服务器上，使用服务器 IP：

```
http://your-server-ip:3000
```

首次访问会进入初始化向导：

1. **创建管理员账号**：设置邮箱和密码
2. **配置默认 LLM**：选择提供商并输入 API Key（可以跳过，后续配置）
3. **完成初始化**：进入主控制台

---

## 2.3 接入第一个渠道（飞书机器人）

为了让 OpenClaw 能够接收和发送消息，我们需要接入至少一个聊天渠道。这里以**飞书机器人**为例，因为它是国内用户最容易获取的渠道之一。

### 步骤1：创建飞书应用

1. 访问 [飞书开放平台](https://open.feishu.cn/)
2. 登录后点击「创建企业自建应用"
3. 填写应用名称（如"OpenClaw助手"）和描述
4. 点击「创建应用」

### 步骤2：配置机器人能力

1. 进入应用详情页，点击左侧「机器人」
2. 开启「机器人」能力开关
3. 设置机器人名称和描述
4. 点击「保存」

### 步骤3：获取凭证信息

1. 点击左侧「凭证与基础信息」
2. 复制 **App ID** 和 **App Secret**

```
App ID: cli_xxxxxxxxxxxxxxxx
App Secret: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

3. 点击左侧「事件订阅」
4. 复制 **Encrypt Key** 和 **Verification Token**

### 步骤4：配置事件订阅

1. 在 OpenClaw Web UI 中，进入「渠道管理」→「添加渠道」
2. 选择「飞书」
3. 填写以下信息：
   - 渠道名称：飞书助手（自定义）
   - App ID：从飞书后台复制的 App ID
   - App Secret：从飞书后台复制的 App Secret
   - Encrypt Key：从飞书后台复制的 Encrypt Key
   - Verification Token：从飞书后台复制的 Verification Token
   - Webhook URL：系统会自动生成，格式为 `http://your-domain/api/webhooks/feishu/xxx`

4. 保存后，复制生成的 Webhook URL

5. 回到飞书后台「事件订阅」页面：
   - 请求地址配置：粘贴刚才复制的 Webhook URL
   - 点击「保存」

⚠️ **重要**：如果你的 OpenClaw 部署在本地或内网，飞书无法直接访问。此时需要使用内网穿透工具（如 ngrok），详见下文「本地调试技巧」。

### 步骤5：订阅事件

在飞书后台「事件订阅」页面，添加以下事件：

```
- 接收消息 v2.0 (im.message.receive_v1)
```

### 步骤6：配置权限

1. 点击左侧「权限管理」
2. 搜索并添加以下权限：
   - `im:chat:readonly`（获取群组信息）
   - `im:message:send`（发送消息）
   - `im:message:read`（读取消息）

3. 点击「批量开通」

### 步骤7：发布应用

1. 点击左侧「版本管理与发布」
2. 点击「创建版本」
3. 填写版本号（如 1.0.0）和更新说明
4. 点击「保存」→「申请发布」
5. 在飞书管理后台审批通过

### 步骤8：测试机器人

1. 在飞书中搜索你的机器人名称
2. 点击「添加」或「进入聊天」
3. 发送一条消息，如"你好"
4. 如果配置正确，机器人会回复 AI 的响应

---

## 2.4 配置第一个 LLM

如果初始化时没有配置 LLM，或者想更换模型，可以在 Web UI 中进行配置。

### 步骤1：获取 API Key

以 DeepSeek 为例（国内用户友好）：

1. 访问 [DeepSeek 开放平台](https://platform.deepseek.com/)
2. 注册并登录
3. 进入「API Keys」页面
4. 点击「创建 API Key"
5. 复制生成的 Key（格式如 `sk-xxxxxxxxxxxxxxxx`）

### 步骤2：在 OpenClaw 中配置

1. 进入 OpenClaw Web UI
2. 点击「模型管理」→「添加模型"
3. 填写配置：

```yaml
提供商: DeepSeek
模型名称: deepseek-chat
API Key: sk-xxxxxxxxxxxxxxxx
Base URL: https://api.deepseek.com/v1  # 默认，一般无需修改
温度: 0.7                              # 控制创造性，0-1之间
最大Token: 4096                        # 单次响应最大长度
```

4. 点击「测试连接」验证配置
5. 保存配置

### 步骤3：设置默认模型

1. 进入「Agent 管理」→「默认 Agent"
2. 在「模型」下拉框中选择刚配置的 DeepSeek
3. 保存配置

---

## 2.5 本地调试技巧

如果你在本地开发环境部署 OpenClaw，聊天平台的 Webhook 无法直接访问你的本地地址。这时需要内网穿透。

### 使用 ngrok

```bash
# 安装 ngrok（macOS）
brew install ngrok

# 或者下载安装（Linux/Windows）
# 访问 https://ngrok.com/download

# 配置 authtoken（需注册 ngrok 账号）
ngrok config add-authtoken your_token

# 启动穿透，将本地 3000 端口暴露到公网
ngrok http 3000
```

启动后会看到类似输出：

```
Session Status                online
Account                       your@email.com
Version                       3.0.0
Region                        United States (us)
Web Interface                 http://127.0.0.1:4040
Forwarding                    https://xxxx.ngrok-free.app -> http://localhost:3000
```

将 `https://xxxx.ngrok-free.app` 作为你的公网地址，配置到飞书等平台的 Webhook URL 中。

⚠️ **注意**：ngrok 免费版每次重启会更换域名，适合临时调试。长期使用建议购买付费版或使用其他方案（如 frp）。

---

## 2.6 常见问题排查

### 问题1：Docker 启动失败

**现象**：`docker compose up` 报错，提示端口被占用

**解决**：

```bash
# 查看端口占用
sudo lsof -i :3000

# 停止占用进程，或修改 docker-compose.yml 中的端口映射
# 例如改为 "3001:3000"，然后通过 3001 端口访问
```

### 问题2：无法访问 Web UI

**现象**：浏览器访问 `localhost:3000` 无响应

**排查步骤**：

```bash
# 1. 检查容器状态
docker compose ps

# 2. 查看日志
docker compose logs openclaw

# 3. 检查端口监听
netstat -tlnp | grep 3000

# 4. 如果是云服务器，检查安全组/防火墙是否放行 3000 端口
```

### 问题3：飞书机器人无响应

**现象**：在飞书发送消息，机器人没有回复

**排查步骤**：

1. 检查 OpenClaw 日志：
   ```bash
   docker compose logs -f openclaw
   ```

2. 检查飞书后台「事件订阅」的 URL 配置是否正确

3. 检查是否订阅了 `im.message.receive_v1` 事件

4. 检查是否开通了必要的权限

5. 检查应用是否已发布并通过审批

### 问题4：LLM 无响应或报错

**现象**：机器人回复"服务异常"或长时间无响应

**排查步骤**：

1. 在「模型管理」中点击「测试连接」验证 API Key
2. 检查 API Key 余额是否充足
3. 查看 OpenClaw 日志中的详细错误信息
4. 如果是国内模型，检查网络是否能访问对应 API 地址

---

## 2.7 本章小结

恭喜你！现在你已经：

1. ✅ 使用 Docker Compose 成功部署了 OpenClaw
2. ✅ 完成了初始化配置
3. ✅ 接入了第一个渠道（飞书机器人）
4. ✅ 配置了第一个 LLM（DeepSeek）
5. ✅ 成功与 AI 助手进行了对话

下一章，我们将详细介绍在不同环境下的部署方案，包括云服务器、NAS、K8s 等。

---

## 练习任务

1. 尝试在 OpenClaw 中配置另一个 LLM（如通义千问、GPT-4）
2. 修改默认 Agent 的 System Prompt，让 AI 以"专业程序员"的身份回答问题
3. 邀请同事或朋友加入飞书群组，测试群聊功能

---

## 参考配置

### 完整 .env 模板

```bash
# 数据库配置
DATABASE_URL=postgresql://openclaw:openclaw_password@localhost:5432/openclaw
REDIS_URL=redis://localhost:6379/0

# 应用配置
APP_ENV=production
APP_PORT=3000
APP_HOST=0.0.0.0

# 安全密钥（务必修改！）
JWT_SECRET=your-very-long-and-random-secret-key-here
ENCRYPTION_KEY=your-32-characters-encryption-key!

# 日志配置
LOG_LEVEL=info
LOG_FORMAT=json

# 可选：默认 LLM 配置
# OPENAI_API_KEY=sk-xxxxxxxx
# OPENAI_BASE_URL=https://api.openai.com/v1
# DEEPSEEK_API_KEY=sk-xxxxxxxx
```
