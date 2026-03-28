# 附录A：环境变量参考手册

> 完整的环境变量配置参考

---

## 核心配置

### 应用基础配置

| 变量名 | 默认值 | 说明 | 必填 |
|--------|--------|------|------|
| `APP_ENV` | `production` | 运行环境：`development`, `production`, `test` | 否 |
| `APP_PORT` | `3000` | 应用监听端口 | 否 |
| `APP_HOST` | `0.0.0.0` | 应用监听地址 | 否 |
| `APP_URL` | - | 应用公网 URL | 是 |
| `DEBUG` | `false` | 调试模式 | 否 |

### 安全配置

| 变量名 | 默认值 | 说明 | 必填 |
|--------|--------|------|------|
| `JWT_SECRET` | - | JWT 签名密钥（至少32位） | 是 |
| `ENCRYPTION_KEY` | - | 数据加密密钥（32位） | 是 |
| `API_KEY_SALT` | - | API Key 加盐值 | 是 |

---

## 数据库配置

### PostgreSQL

| 变量名 | 默认值 | 说明 | 必填 |
|--------|--------|------|------|
| `DATABASE_URL` | - | 完整数据库连接 URL | 是 |
| `DB_HOST` | `localhost` | 数据库主机 | 否 |
| `DB_PORT` | `5432` | 数据库端口 | 否 |
| `DB_NAME` | `openclaw` | 数据库名 | 否 |
| `DB_USER` | `openclaw` | 数据库用户 | 否 |
| `DB_PASSWORD` | - | 数据库密码 | 是 |
| `DB_POOL_SIZE` | `20` | 连接池大小 | 否 |
| `DB_MAX_OVERFLOW` | `10` | 连接池溢出 | 否 |

### Redis

| 变量名 | 默认值 | 说明 | 必填 |
|--------|--------|------|------|
| `REDIS_URL` | - | 完整 Redis 连接 URL | 是 |
| `REDIS_HOST` | `localhost` | Redis 主机 | 否 |
| `REDIS_PORT` | `6379` | Redis 端口 | 否 |
| `REDIS_DB` | `0` | Redis 数据库号 | 否 |
| `REDIS_PASSWORD` | - | Redis 密码 | 否 |

---

## LLM 配置

### OpenAI

| 变量名 | 默认值 | 说明 | 必填 |
|--------|--------|------|------|
| `OPENAI_API_KEY` | - | OpenAI API Key | 否 |
| `OPENAI_BASE_URL` | `https://api.openai.com/v1` | API 基础 URL | 否 |
| `OPENAI_DEFAULT_MODEL` | `gpt-3.5-turbo` | 默认模型 | 否 |

### DeepSeek

| 变量名 | 默认值 | 说明 | 必填 |
|--------|--------|------|------|
| `DEEPSEEK_API_KEY` | - | DeepSeek API Key | 否 |
| `DEEPSEEK_BASE_URL` | `https://api.deepseek.com/v1` | API 基础 URL | 否 |

### 通义千问

| 变量名 | 默认值 | 说明 | 必填 |
|--------|--------|------|------|
| `DASHSCOPE_API_KEY` | - | 阿里云 Dashscope API Key | 否 |

### 智谱 AI

| 变量名 | 默认值 | 说明 | 必填 |
|--------|--------|------|------|
| `ZHIPU_API_KEY` | - | 智谱 AI API Key | 否 |

---

## 渠道配置

### 飞书

| 变量名 | 默认值 | 说明 | 必填 |
|--------|--------|------|------|
| `FEISHU_APP_ID` | - | 飞书应用 App ID | 否 |
| `FEISHU_APP_SECRET` | - | 飞书应用 App Secret | 否 |
| `FEISHU_ENCRYPT_KEY` | - | 消息加密密钥 | 否 |
| `FEISHU_VERIFICATION_TOKEN` | - | 验证 Token | 否 |

### 钉钉

| 变量名 | 默认值 | 说明 | 必填 |
|--------|--------|------|------|
| `DINGTALK_ACCESS_TOKEN` | - | 钉钉机器人 Access Token | 否 |
| `DINGTALK_SECRET` | - | 钉钉机器人加签密钥 | 否 |

### 企业微信

| 变量名 | 默认值 | 说明 | 必填 |
|--------|--------|------|------|
| `WECOM_CORP_ID` | - | 企业 ID | 否 |
| `WECOM_AGENT_ID` | - | 应用 AgentId | 否 |
| `WECOM_SECRET` | - | 应用 Secret | 否 |
| `WECOM_TOKEN` | - | 消息 Token | 否 |
| `WECOM_ENCODING_AES_KEY` | - | 消息加密密钥 | 否 |

### Telegram

| 变量名 | 默认值 | 说明 | 必填 |
|--------|--------|------|------|
| `TELEGRAM_BOT_TOKEN` | - | Bot Token | 否 |
| `TELEGRAM_WEBHOOK_URL` | - | Webhook URL | 否 |

### Discord

| 变量名 | 默认值 | 说明 | 必填 |
|--------|--------|------|------|
| `DISCORD_BOT_TOKEN` | - | Bot Token | 否 |

---

## 日志配置

| 变量名 | 默认值 | 说明 | 必填 |
|--------|--------|------|------|
| `LOG_LEVEL` | `info` | 日志级别：`debug`, `info`, `warn`, `error` | 否 |
| `LOG_FORMAT` | `json` | 日志格式：`json`, `text` | 否 |
| `LOG_OUTPUT` | `stdout` | 日志输出：`stdout`, `file` | 否 |
| `LOG_FILE_PATH` | `/var/log/openclaw` | 日志文件路径 | 否 |

---

## 监控配置

| 变量名 | 默认值 | 说明 | 必填 |
|--------|--------|------|------|
| `METRICS_ENABLED` | `true` | 启用指标收集 | 否 |
| `METRICS_PORT` | `9090` | 指标端口 | 否 |
| `TRACING_ENABLED` | `false` | 启用链路追踪 | 否 |
| `JAEGER_ENDPOINT` | - | Jaeger 地址 | 否 |

---

## 配置模板

### 开发环境

```bash
# .env.development
APP_ENV=development
DEBUG=true
LOG_LEVEL=debug

DATABASE_URL=postgresql://openclaw:password@localhost:5432/openclaw
REDIS_URL=redis://localhost:6379/0

JWT_SECRET=dev-jwt-secret-key-change-in-production
ENCRYPTION_KEY=dev-encryption-key-32chars-long!!

DEEPSEEK_API_KEY=sk-your-key
```

### 生产环境

```bash
# .env.production
APP_ENV=production
DEBUG=false
LOG_LEVEL=info

DATABASE_URL=postgresql://openclaw:${DB_PASSWORD}@postgres:5432/openclaw
REDIS_URL=redis://redis:6379/0

JWT_SECRET=${JWT_SECRET}
ENCRYPTION_KEY=${ENCRYPTION_KEY}
API_KEY_SALT=${API_KEY_SALT}

DEEPSEEK_API_KEY=${DEEPSEEK_API_KEY}

FEISHU_APP_ID=${FEISHU_APP_ID}
FEISHU_APP_SECRET=${FEISHU_APP_SECRET}
```

### Docker Compose 环境

```yaml
# docker-compose.yml 环境变量
version: '3.8'

services:
  openclaw:
    image: openclaw/openclaw:latest
    environment:
      - APP_ENV=production
      - DATABASE_URL=postgresql://openclaw:password@postgres:5432/openclaw
      - REDIS_URL=redis://redis:6379/0
      - JWT_SECRET=${JWT_SECRET}
      - ENCRYPTION_KEY=${ENCRYPTION_KEY}
      - DEEPSEEK_API_KEY=${DEEPSEEK_API_KEY}
    env_file:
      - .env
```

---

## 安全提示

⚠️ **重要安全提示**：

1. **永远不要**将包含敏感信息的 `.env` 文件提交到 Git
2. **生产环境**必须使用强密码和密钥
3. **定期更换** API Key 和密钥
4. **使用**密钥管理服务（如 AWS Secrets Manager、HashiCorp Vault）
5. **限制**环境变量的访问权限

```bash
# .gitignore
.env
.env.*
*.pem
*.key
```

