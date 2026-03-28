# 第6章：LLM 接入与配置

> 支持 30+ 大模型提供商的接入方法与最佳实践

---

## 6.1 支持的 LLM 提供商

OpenClaw 通过统一的接口支持 30+ 家 LLM 提供商，包括：

### 国际主流模型

| 提供商 | 代表模型 | 特点 |
|--------|----------|------|
| OpenAI | GPT-4o, GPT-4, GPT-3.5 | 能力最强，价格较高 |
| Anthropic | Claude 3.5 Sonnet, Claude 3 Opus | 长文本处理优秀 |
| Google | Gemini Pro, Gemini Ultra | 多模态能力强 |
| Cohere | Command R | 企业级 RAG 优化 |
| Mistral | Mistral Large | 欧洲领先模型 |

### 国内主流模型

| 提供商 | 代表模型 | 特点 |
|--------|----------|------|
| DeepSeek | DeepSeek-V2, DeepSeek-Coder | 代码能力强，性价比高 |
| 通义千问 | Qwen-Max, Qwen-Plus | 阿里出品，中文优秀 |
| 文心一言 | ERNIE-Bot 4.0 | 百度出品，知识丰富 |
| 智谱 AI | GLM-4, GLM-4-Flash | 清华背景，开源友好 |
| 讯飞星火 | Spark Pro | 语音交互强 |
| 月之暗面 | Kimi | 长上下文窗口 |
| MiniMax | abab6 | 多模态能力 |

### 开源/自托管模型

| 模型 | 参数规模 | 特点 |
|------|----------|------|
| Llama 3 | 8B/70B | Meta 开源，生态丰富 |
| Qwen2 | 7B/72B | 阿里开源，中文优化 |
| ChatGLM3 | 6B | 清华开源，可本地部署 |
| Mistral | 7B | 性能超越 Llama 2 |

---

## 6.2 API Key 获取指南

### OpenAI

1. 访问 [OpenAI Platform](https://platform.openai.com/)
2. 注册/登录账号
3. 进入「API keys」页面
4. 点击「Create new secret key"
5. 复制并保存（只显示一次）

⚠️ **国内用户注意**：OpenAI API 在国内无法直接访问，需要：
- 使用代理/VPN
- 使用第三方中转服务
- 使用国内替代模型

### DeepSeek（推荐国内用户）

1. 访问 [DeepSeek 开放平台](https://platform.deepseek.com/)
2. 注册账号（支持国内手机号）
3. 进入「API Keys」页面
4. 点击「创建 API Key"
5. 复制生成的 Key

**价格参考**：
- DeepSeek-V2：输入 1元/百万 tokens，输出 2元/百万 tokens
- 新用户有 5000万 tokens 免费额度

### 通义千问

1. 访问 [阿里云百炼](https://bailian.console.aliyun.com/)
2. 登录阿里云账号
3. 进入「模型广场」→ 「API Key 管理"
4. 创建 API Key

**价格参考**：
- qwen-max：0.02元/千 tokens
- qwen-plus：0.004元/千 tokens
- qwen-turbo：0.002元/千 tokens

### 智谱 AI

1. 访问 [智谱 AI 开放平台](https://open.bigmodel.cn/)
2. 注册账号
3. 进入「API Keys」页面
4. 创建 API Key

---

## 6.3 模型配置方法

### 基础配置

在 OpenClaw Web UI 中配置：

```yaml
# 模型配置
models:
  - name: "deepseek-chat"
    provider: deepseek
    config:
      api_key: "${DEEPSEEK_API_KEY}"
      base_url: "https://api.deepseek.com/v1"
      model: "deepseek-chat"
      temperature: 0.7
      max_tokens: 4096
      timeout: 60
  
  - name: "gpt-4o"
    provider: openai
    config:
      api_key: "${OPENAI_API_KEY}"
      base_url: "https://api.openai.com/v1"  # 或中转地址
      model: "gpt-4o"
      temperature: 0.7
      max_tokens: 4096
```

### 配置文件方式

也可以直接编辑配置文件：

```yaml
# config/models.yaml
models:
  - id: "deepseek-v2"
    name: "DeepSeek-V2"
    provider: "deepseek"
    config:
      api_key: "${DEEPSEEK_API_KEY}"
      model: "deepseek-chat"
    defaults:
      temperature: 0.7
      max_tokens: 4096
      top_p: 0.9
  
  - id: "qwen-max"
    name: "通义千问-Max"
    provider: "dashscope"
    config:
      api_key: "${DASHSCOPE_API_KEY}"
      model: "qwen-max"
    defaults:
      temperature: 0.7
      max_tokens: 2048
```

### 环境变量注入

敏感信息建议通过环境变量注入：

```bash
# .env 文件
DEEPSEEK_API_KEY=sk-xxxxx
OPENAI_API_KEY=sk-xxxxx
DASHSCOPE_API_KEY=sk-xxxxx
```

```yaml
# docker-compose.yml
services:
  openclaw:
    env_file: .env
```

---

## 6.4 模型选择指南

### 按场景选择

| 场景 | 推荐模型 | 理由 |
|------|----------|------|
| 日常对话 | DeepSeek-V2 / GPT-3.5 | 性价比高 |
| 复杂推理 | GPT-4o / Claude 3.5 | 能力强 |
| 代码生成 | DeepSeek-Coder / GPT-4 | 代码专精 |
| 长文档处理 | Claude 3 / Kimi | 上下文长 |
| 中文内容 | 通义千问 / 文心一言 | 中文优化 |
| 实时响应 | GLM-4-Flash / GPT-3.5 | 速度快 |

### 性价比分析

以处理 1000 次对话（每次约 2K tokens 输入，500 tokens 输出）为例：

| 模型 | 单次成本 | 1000次成本 | 质量评分 |
|------|----------|------------|----------|
| GPT-4o | ¥0.15 | ¥150 | ⭐⭐⭐⭐⭐ |
| GPT-3.5 | ¥0.03 | ¥30 | ⭐⭐⭐⭐ |
| DeepSeek-V2 | ¥0.006 | ¥6 | ⭐⭐⭐⭐ |
| 通义千问-Max | ¥0.05 | ¥50 | ⭐⭐⭐⭐ |
| GLM-4-Flash | ¥0.001 | ¥1 | ⭐⭐⭐ |

### 响应速度对比

| 模型 | 首字延迟 | 生成速度 | 稳定性 |
|------|----------|----------|--------|
| GPT-4o | 0.5s | 快 | 高 |
| DeepSeek-V2 | 1s | 快 | 高 |
| 通义千问-Max | 0.8s | 快 | 高 |
| Claude 3 | 1.5s | 中等 | 高 |
| 自托管 7B | 0.2s | 快 | 中 |

---

## 6.5 多模型路由

### 按场景路由

配置不同场景使用不同模型：

```yaml
# 路由规则
routing:
  rules:
    # 代码问题用 DeepSeek-Coder
    - condition: "message.contains('代码') or message.contains('编程')"
      model: "deepseek-coder"
    
    # 长文档用 Claude
    - condition: "message.length > 2000"
      model: "claude-3-sonnet"
    
    # 复杂推理用 GPT-4
    - condition: "intent == 'complex_reasoning'"
      model: "gpt-4o"
    
    # 默认用 DeepSeek
    - condition: "default"
      model: "deepseek-chat"
```

### 降级策略

当主模型不可用时自动降级：

```yaml
models:
  - id: "primary"
    name: "GPT-4o"
    fallback_to: "gpt-4"  # 第一降级
  
  - id: "gpt-4"
    name: "GPT-4"
    fallback_to: "deepseek-chat"  # 第二降级
  
  - id: "deepseek-chat"
    name: "DeepSeek-V2"
    fallback_to: null  # 不再降级
```

### 负载均衡

多个模型实例间负载均衡：

```yaml
model_pool:
  - model: "deepseek-chat"
    instances:
      - api_key: "${DEEPSEEK_KEY_1}"
        weight: 50
      - api_key: "${DEEPSEEK_KEY_2}"
        weight: 50
```

---

## 6.6 参数调优

### Temperature（温度）

控制输出的随机性：

| Temperature | 特点 | 适用场景 |
|-------------|------|----------|
| 0.0 | 确定性输出，每次相同 | 代码生成、数学计算 |
| 0.3-0.5 | 低随机性，较保守 | 问答、知识查询 |
| 0.7-0.9 | 中等随机性，较灵活 | 创意写作、头脑风暴 |
| 1.0+ | 高随机性，可能发散 | 艺术创作、探索性任务 |

### Max Tokens（最大长度）

控制单次响应的最大长度：

```yaml
# 根据场景设置
short_response:
  max_tokens: 256    # 简短回答
  
medium_response:
  max_tokens: 1024   # 中等长度
  
long_response:
  max_tokens: 4096   # 长文生成
```

### Top P（核采样）

控制输出的多样性：

```yaml
# Top P = 0.9 表示只从累积概率前 90% 的 token 中采样
conservative:
  top_p: 0.5  # 保守
  
balanced:
  top_p: 0.9  # 平衡（推荐）
  
creative:
  top_p: 1.0  # 创意
```

### 频率惩罚与存在惩罚

```yaml
# frequency_penalty: 降低重复用词的概率
# presence_penalty: 鼓励讨论新话题

avoid_repetition:
  frequency_penalty: 0.5
  presence_penalty: 0.3
```

---

## 6.7 国产模型特化配置

### 国内网络环境优化

```yaml
# 使用国内 CDN 加速
models:
  - name: "qwen-max"
    provider: dashscope
    config:
      api_key: "${DASHSCOPE_API_KEY}"
      base_url: "https://dashscope.aliyuncs.com/api/v1"
      # 国内直接访问，无需代理
  
  - name: "deepseek-chat"
    provider: deepseek
    config:
      api_key: "${DEEPSEEK_API_KEY}"
      base_url: "https://api.deepseek.com/v1"
      # 国内访问速度快
```

### 中文优化提示词

针对国产模型的中文优化：

```yaml
agent:
  system_prompt: |
    你是一个专业的 AI 助手，请用中文回答用户问题。
    
    回答要求：
    1. 使用规范的中文标点符号
    2. 专业术语可保留英文，但需给出中文解释
    3. 代码块使用 Markdown 格式
    4. 长回答使用分段和列表提高可读性
```

---

## 6.8 Token 用量监控

### 用量统计

```yaml
# 启用用量监控
monitoring:
  token_usage:
    enabled: true
    log_level: info
    alert_threshold:
      daily: 1000000    # 日用量超过 100万 tokens 告警
      monthly: 20000000 # 月用量超过 2000万 tokens 告警
```

### 成本控制策略

```yaml
# 成本优化配置
cost_optimization:
  # 缓存相似请求
  cache:
    enabled: true
    ttl: 3600  # 缓存 1 小时
    similarity_threshold: 0.95
  
  # 按成本选择模型
  model_selection:
    strategy: "cost_aware"  # 或 "quality_first", "balanced"
    max_cost_per_request: 0.1  # 单次请求最大成本（元）
```

---

## 6.9 本章小结

本章讲解了 LLM 接入的核心要点：

1. **模型选择**：根据场景、成本、质量需求选择合适的模型
2. **API 配置**：获取 Key、配置参数、设置环境变量
3. **多模型路由**：按场景自动选择模型，设置降级策略
4. **参数调优**：Temperature、Max Tokens、Top P 等参数的含义和调优方法
5. **国产模型**：国内网络环境下的接入优化

**推荐配置（国内用户）**：

```yaml
models:
  - id: "default"
    name: "DeepSeek-V2"
    provider: deepseek
    config:
      api_key: "${DEEPSEEK_API_KEY}"
      model: "deepseek-chat"
      temperature: 0.7
      max_tokens: 4096
```

---

## 参考配置

```yaml
# config/models.yaml - 完整示例
models:
  # 主力模型
  - id: "primary"
    name: "DeepSeek-V2"
    provider: deepseek
    config:
      api_key: "${DEEPSEEK_API_KEY}"
      model: "deepseek-chat"
    defaults:
      temperature: 0.7
      max_tokens: 4096
      top_p: 0.9
  
  # 代码专用
  - id: "coder"
    name: "DeepSeek-Coder"
    provider: deepseek
    config:
      api_key: "${DEEPSEEK_API_KEY}"
      model: "deepseek-coder"
    defaults:
      temperature: 0.2
      max_tokens: 8192
  
  # 备用模型
  - id: "backup"
    name: "通义千问-Plus"
    provider: dashscope
    config:
      api_key: "${DASHSCOPE_API_KEY}"
      model: "qwen-plus"
    defaults:
      temperature: 0.7
      max_tokens: 2048
```
