# 附录B：API 参考

> OpenClaw REST API 完整参考

---

## 认证

所有 API 请求需要在 Header 中携带认证信息：

```http
Authorization: Bearer {access_token}
```

### 获取 Token

```http
POST /api/v1/auth/login
Content-Type: application/json

{
  "username": "admin",
  "password": "your-password"
}
```

**响应**：

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

---

## Agent API

### 列出所有 Agent

```http
GET /api/v1/agents
```

**响应**：

```json
{
  "items": [
    {
      "id": "agent_123",
      "name": "客服助手",
      "description": "智能客服 Agent",
      "status": "active",
      "created_at": "2024-01-15T08:00:00Z"
    }
  ],
  "total": 10,
  "page": 1,
  "page_size": 20
}
```

### 创建 Agent

```http
POST /api/v1/agents
Content-Type: application/json

{
  "name": "新 Agent",
  "description": "Agent 描述",
  "system_prompt": "你是...",
  "model": "deepseek-chat",
  "tools": ["weather", "search"]
}
```

### 获取 Agent 详情

```http
GET /api/v1/agents/{agent_id}
```

### 更新 Agent

```http
PUT /api/v1/agents/{agent_id}
Content-Type: application/json

{
  "name": "更新后的名称",
  "system_prompt": "更新后的提示词"
}
```

### 删除 Agent

```http
DELETE /api/v1/agents/{agent_id}
```

---

## Channel API

### 列出渠道

```http
GET /api/v1/channels
```

### 创建渠道

```http
POST /api/v1/channels
Content-Type: application/json

{
  "name": "飞书助手",
  "type": "feishu",
  "config": {
    "app_id": "cli_xxx",
    "app_secret": "xxx",
    "encrypt_key": "xxx",
    "verification_token": "xxx"
  }
}
```

### 获取渠道详情

```http
GET /api/v1/channels/{channel_id}
```

### 更新渠道

```http
PUT /api/v1/channels/{channel_id}
Content-Type: application/json

{
  "name": "新名称",
  "config": {
    "app_id": "cli_new"
  }
}
```

### 删除渠道

```http
DELETE /api/v1/channels/{channel_id}
```

### 测试渠道连接

```http
POST /api/v1/channels/{channel_id}/test
```

---

## 消息 API

### 发送消息

```http
POST /api/v1/messages/send
Content-Type: application/json

{
  "channel_id": "channel_123",
  "user_id": "user_456",
  "content": {
    "type": "text",
    "text": "Hello World"
  }
}
```

**响应**：

```json
{
  "message_id": "msg_789",
  "status": "sent",
  "timestamp": "2024-01-15T08:00:00Z"
}
```

### 发送富媒体消息

```http
POST /api/v1/messages/send
Content-Type: application/json

{
  "channel_id": "channel_123",
  "user_id": "user_456",
  "content": {
    "type": "image",
    "image_url": "https://example.com/image.jpg"
  }
}
```

### 获取消息历史

```http
GET /api/v1/messages?channel_id={channel_id}&limit=50
```

---

## 工作流 API

### 列出工作流

```http
GET /api/v1/workflows
```

### 创建工作流

```http
POST /api/v1/workflows
Content-Type: application/json

{
  "name": "每日早报",
  "description": "定时发送早报",
  "trigger": {
    "type": "cron",
    "schedule": "0 8 * * *"
  },
  "steps": [
    {
      "name": "获取天气",
      "type": "skill",
      "skill": "weather",
      "method": "get_current"
    }
  ]
}
```

### 触发工作流

```http
POST /api/v1/workflows/{workflow_id}/trigger
Content-Type: application/json

{
  "data": {
    "city": "北京"
  }
}
```

### 获取工作流执行状态

```http
GET /api/v1/workflow-runs/{run_id}
```

**响应**：

```json
{
  "id": "run_xxx",
  "workflow_id": "wf_xxx",
  "status": "running",
  "started_at": "2024-01-15T08:00:00Z",
  "steps": [
    {
      "name": "获取天气",
      "status": "completed",
      "output": {"temperature": 25}
    }
  ]
}
```

---

## Skill API

### 列出已安装 Skills

```http
GET /api/v1/skills
```

### 安装 Skill

```http
POST /api/v1/skills/install
Content-Type: application/json

{
  "name": "weather-query",
  "version": "1.0.0"
}
```

### 调用 Skill

```http
POST /api/v1/skills/{skill_name}/call
Content-Type: application/json

{
  "method": "get_current_weather",
  "params": {
    "city": "北京"
  }
}
```

---

## 用户 API

### 列出用户

```http
GET /api/v1/users
```

### 创建用户

```http
POST /api/v1/users
Content-Type: application/json

{
  "username": "newuser",
  "email": "user@example.com",
  "password": "secure-password",
  "role": "operator"
}
```

### 获取当前用户

```http
GET /api/v1/users/me
```

### 更新用户

```http
PUT /api/v1/users/{user_id}
Content-Type: application/json

{
  "email": "newemail@example.com"
}
```

---

## Webhook API

### 列出 Webhooks

```http
GET /api/v1/webhooks
```

### 创建 Webhook

```http
POST /api/v1/webhooks
Content-Type: application/json

{
  "name": "外部通知",
  "url": "https://example.com/webhook",
  "events": ["message.received", "agent.created"],
  "secret": "webhook-secret"
}
```

---

## 错误码

| 状态码 | 错误码 | 说明 |
|--------|--------|------|
| 400 | `BAD_REQUEST` | 请求参数错误 |
| 401 | `UNAUTHORIZED` | 未认证 |
| 403 | `FORBIDDEN` | 无权限 |
| 404 | `NOT_FOUND` | 资源不存在 |
| 409 | `CONFLICT` | 资源冲突 |
| 422 | `VALIDATION_ERROR` | 验证失败 |
| 429 | `RATE_LIMITED` | 请求过于频繁 |
| 500 | `INTERNAL_ERROR` | 服务器内部错误 |

### 错误响应格式

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "请求参数验证失败",
    "details": [
      {
        "field": "email",
        "message": "邮箱格式不正确"
      }
    ]
  }
}
```

---

## 分页

列表接口支持分页：

```http
GET /api/v1/agents?page=2&page_size=20
```

**响应**：

```json
{
  "items": [...],
  "total": 100,
  "page": 2,
  "page_size": 20,
  "total_pages": 5
}
```

---

## 限流

API 限流规则：

| 接口类型 | 限制 |
|----------|------|
| 普通接口 | 100 请求/分钟 |
| 消息发送 | 60 请求/分钟 |
| LLM 调用 | 20 请求/分钟 |

超过限流会返回 429 状态码：

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "请求过于频繁，请稍后重试",
    "retry_after": 60
  }
}
```

