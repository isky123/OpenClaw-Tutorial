# 第7章：Agent 配置与优化

> 打造个性化的 AI 助手：Prompt 工程、记忆管理、工具调用

---

## 7.1 角色定义（System Prompt）

System Prompt 是 Agent 的"人设"，决定了 AI 的行为方式、知识边界和输出风格。

### System Prompt 结构

一个完整的 System Prompt 应包含以下部分：

```markdown
# 角色定义
你是 {角色名称}，{角色身份描述}。

## 核心能力
- 能力1：{详细描述}
- 能力2：{详细描述}
- 能力3：{详细描述}

## 行为规范
1. {行为准则1}
2. {行为准则2}
3. {行为准则3}

## 输出格式
{期望的输出格式说明}

## 限制条件
- 限制1：{不能做的事情}
- 限制2：{需要避免的情况}

## 上下文信息
{当前会话的上下文信息}
```

### 示例：技术支持助手

```yaml
agent:
  name: "技术支持助手"
  system_prompt: |
    你是 Acme 公司的技术支持专家，专门帮助用户解决产品使用问题。
    
    ## 身份信息
    - 姓名：小 A
    - 性格：耐心、专业、友好
    - 语言：中文（可根据用户切换）
    
    ## 核心能力
    1. 产品咨询：解答产品功能、定价、购买相关问题
    2. 故障排查：指导用户诊断和解决常见问题
    3. 使用指导：提供产品使用教程和最佳实践
    4. 反馈收集：记录用户反馈并生成工单
    
    ## 行为规范
    1. 首次回复必须问候用户并询问具体问题
    2. 解答时分步骤说明，使用编号列表
    3. 涉及操作步骤时，提供清晰的指引
    4. 遇到复杂问题，主动询问用户是否方便电话沟通
    5. 每次回复末尾添加："如果问题未解决，请回复'转人工'"
    
    ## 输出格式
    - 使用 Markdown 格式
    - 关键信息使用 **粗体** 标注
    - 操作步骤使用 1. 2. 3. 编号
    - 代码示例使用 ``` 代码块
    
    ## 限制条件
    - 不讨论竞争对手产品
    - 不透露内部技术架构细节
    - 不承诺具体的修复时间
    - 不涉及账户密码等敏感操作
    
    ## 工具使用
    当需要查询用户订单时，使用 query_order 工具
    当需要创建工单时，使用 create_ticket 工具
```

### 示例：内容创作助手

```yaml
agent:
  name: "内容创作助手"
  system_prompt: |
    你是一位资深的内容创作者和编辑，擅长撰写各类新媒体内容。
    
    ## 写作风格
    - 语言风格：轻松自然、接地气、避免生硬翻译腔
    - 段落长度：每段不超过 3 句话，多用短句
    - 表达方式：用具体例子说明抽象概念
    
    ## 内容要求
    1. 标题要有吸引力，使用数字、疑问、对比等手法
    2. 开头要抓人，直接给出价值承诺或抛出悬念
    3. 正文要有信息量，提供可操作的干货
    4. 结尾要有行动号召或总结升华
    
    ## 格式规范
    - 使用 ## 作为一级标题
    - 使用 ### 作为二级标题
    - 使用 - 作为列表符号
    - 关键数据使用 **粗体** 强调
    - 引用使用 > 格式
    
    ## 平台适配
    根据用户指定的平台调整内容：
    - 公众号：深度长文，1500-3000 字
    - 小红书：短平快，300-500 字，多用 emoji
    - 知乎：专业分析，逻辑清晰，2000 字以上
    - 微博：简洁有力，140 字以内
```

---

## 7.2 记忆管理

### 短期记忆（上下文窗口）

短期记忆维护当前对话的历史消息，直接参与 LLM 推理。

**配置方法**：

```yaml
memory:
  short_term:
    max_messages: 20        # 保留最近 20 条消息
    max_tokens: 4000        # Token 上限
    strategy: sliding_window # 滑动窗口策略
```

**滑动窗口策略**：

```
对话历史：
[M1] 用户：你好
[M2] AI：你好！有什么可以帮你的？
[M3] 用户：介绍一下产品
[M4] AI：我们的产品...
...
[M19] 用户：价格是多少
[M20] AI：价格是...
[M21] 用户：有什么优惠

Token 超限处理：
- 保留最近 N 条：保留 M16-M21，丢弃 M1-M15
- 或者摘要替换：将 M1-M10 摘要为一段，保留摘要+M11-M21
```

### 长期记忆（向量数据库）

长期记忆存储跨会话的知识，通过向量检索获取相关信息。

**配置方法**：

```yaml
memory:
  long_term:
    enabled: true
    vector_store: chroma     # 可选：chroma, milvus, pinecone, qdrant
    embedding_model: text-embedding-3-small
    
    # 记忆类型
    memory_types:
      - user_preferences     # 用户偏好
      - important_facts      # 重要事实
      - conversation_summary # 对话摘要
```

**记忆存储示例**：

```python
# 用户提到的重要信息自动存储
user_message = "我喜欢用 Python 编程，不太熟悉 JavaScript"

# 提取并存储记忆
memory.store(
    user_id="user_123",
    content="用户偏好使用 Python，不熟悉 JavaScript",
    category="user_preferences",
    importance=0.8
)

# 后续对话中自动检索
# 用户问："推荐一门编程语言"
# 检索到记忆："用户偏好使用 Python"
# AI 回复："考虑到您之前提到喜欢用 Python..."
```

**记忆检索**：

```python
# 检索与当前问题相关的记忆
relevant_memories = memory.search(
    query="用户喜欢什么编程语言",
    user_id="user_123",
    top_k=5,
    min_similarity=0.7
)

# 结果：
# [
#   {"content": "用户偏好使用 Python", "similarity": 0.92},
#   {"content": "用户不熟悉 JavaScript", "similarity": 0.85}
# ]
```

### 记忆管理最佳实践

**1. 记忆分类存储**

```yaml
memory_categories:
  - name: "user_preferences"
    description: "用户偏好设置"
    retention: permanent  # 永久保留
  
  - name: "conversation_context"
    description: "对话上下文"
    retention: 30d        # 保留 30 天
  
  - name: "temporary_notes"
    description: "临时笔记"
    retention: 7d         # 保留 7 天
```

**2. 记忆重要性评分**

```python
# 自动评估记忆重要性
def calculate_importance(content: str) -> float:
    """
    根据内容特征计算重要性分数 (0-1)
    """
    importance = 0.5
    
    # 包含关键信息的加分
    if any(word in content for word in ["喜欢", "讨厌", "过敏", "重要"]):
        importance += 0.2
    
    # 包含个人信息的加分
    if any(word in content for word in ["我叫", "我是", "我工作"]):
        importance += 0.15
    
    # 包含时间信息的加分
    if any(word in content for word in ["明天", "下周", "计划"]):
        importance += 0.1
    
    return min(importance, 1.0)
```

**3. 记忆过期清理**

```yaml
memory_cleanup:
  enabled: true
  schedule: "0 3 * * *"  # 每天凌晨 3 点执行
  rules:
    - category: "temporary_notes"
      max_age: 7d
    - category: "conversation_context"
      max_age: 30d
    - category: "user_preferences"
      max_age: null  # 永不过期
```

---

## 7.3 工具调用（Function Calling）

工具调用让 Agent 能够与外部系统交互，扩展能力边界。

### 工具定义

```yaml
tools:
  - name: "query_weather"
    description: "查询指定城市的天气信息"
    parameters:
      type: object
      properties:
        city:
          type: string
          description: "城市名称，如'北京'、'上海'"
        date:
          type: string
          description: "日期，格式'YYYY-MM-DD'，默认为今天"
      required: ["city"]
  
  - name: "send_email"
    description: "发送邮件"
    parameters:
      type: object
      properties:
        to:
          type: string
          description: "收件人邮箱"
        subject:
          type: string
          description: "邮件主题"
        content:
          type: string
          description: "邮件正文"
        attachments:
          type: array
          description: "附件列表"
          items:
            type: string
      required: ["to", "subject", "content"]
```

### 工具执行流程

```
用户：北京明天天气怎么样？

Step 1: LLM 分析
- 用户想查询天气
- 需要调用 query_weather 工具
- 参数：city="北京", date="明天"

Step 2: Agent 执行工具
- 调用 query_weather(city="北京", date="2024-01-16")
- 获取结果：{"temperature": "5°C", "weather": "晴", ...}

Step 3: LLM 生成回复
- 基于工具结果生成自然语言回复
- "北京明天晴天，气温 5°C，适合出行~"

Step 4: 返回给用户
```

### 工具开发示例

**天气查询工具**：

```python
# skills/weather/__init__.py
import requests
from typing import Dict, Any

class WeatherSkill:
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://api.weather.com/v1"
    
    async def query_weather(self, city: str, date: str = None) -> Dict[str, Any]:
        """查询天气"""
        
        params = {
            "city": city,
            "key": self.api_key
        }
        if date:
            params["date"] = date
        
        response = requests.get(f"{self.base_url}/weather", params=params)
        data = response.json()
        
        return {
            "city": city,
            "temperature": data["temp"],
            "weather": data["condition"],
            "humidity": data["humidity"],
            "wind": data["wind"]
        }

# 注册工具
manifest = {
    "name": "weather",
    "version": "1.0.0",
    "tools": [
        {
            "name": "query_weather",
            "handler": WeatherSkill.query_weather
        }
    ]
}
```

### 工具选择策略

当定义了多个工具时，LLM 会自动选择最合适的工具：

```yaml
tools:
  - name: "search_knowledge"
    description: "从知识库中搜索信息，适用于产品文档、FAQ 等"
  
  - name: "query_database"
    description: "查询数据库，适用于订单、用户数据等"
  
  - name: "call_api"
    description: "调用外部 API，适用于天气、股票等实时数据"
```

**选择逻辑**：

```
用户：我的订单 12345 到哪里了？
→ 匹配 query_database（涉及订单数据）

用户：你们产品的退款政策是什么？
→ 匹配 search_knowledge（涉及政策文档）

用户：今天适合洗车吗？
→ 匹配 call_api → 进一步调用天气 API
```

---

## 7.4 多轮对话管理

### 对话状态机

复杂对话需要维护状态，跟踪用户的意图变化：

```python
class ConversationState:
    def __init__(self):
        self.current_intent = None  # 当前意图
        self.pending_info = []      # 待收集的信息
        self.collected_info = {}    # 已收集的信息
        self.step = 0               # 当前步骤

# 示例：订机票对话
state = ConversationState()

# 用户：我想订机票
state.current_intent = "book_flight"
state.pending_info = ["出发地", "目的地", "日期", "人数"]
state.step = 1
# AI：请问从哪里出发？

# 用户：北京
state.collected_info["出发地"] = "北京"
state.pending_info.remove("出发地")
state.step = 2
# AI：目的地是哪里？

# 用户：上海
state.collected_info["目的地"] = "上海"
state.pending_info.remove("目的地")
state.step = 3
# AI：出发日期是？

# ... 直到信息收集完成
```

### 上下文压缩

当对话过长时，需要压缩上下文以节省 Token：

```python
class ContextCompressor:
    def compress(self, messages: List[Message], max_tokens: int) -> List[Message]:
        """
        压缩对话上下文
        """
        total_tokens = sum(m.token_count for m in messages)
        
        if total_tokens <= max_tokens:
            return messages
        
        # 策略1：保留系统消息和最近 N 条
        system_msgs = [m for m in messages if m.role == "system"]
        recent_msgs = messages[-10:]  # 保留最近 10 条
        
        # 策略2：对早期消息生成摘要
        old_msgs = messages[1:-10]  # 除去系统和最近消息
        summary = self.summarize(old_msgs)
        
        return [
            *system_msgs,
            Message(role="assistant", content=f"历史对话摘要：{summary}"),
            *recent_msgs
        ]
    
    def summarize(self, messages: List[Message]) -> str:
        """生成对话摘要"""
        # 调用 LLM 生成摘要
        prompt = f"请总结以下对话的关键信息：\n{messages}"
        return llm.generate(prompt, max_tokens=200)
```

### 指代消解

处理用户消息中的指代词（它、那个、刚才说的）：

```python
class ReferenceResolver:
    def resolve(self, message: str, history: List[Message]) -> str:
        """
        将指代词替换为具体实体
        """
        # 识别指代词
        pronouns = ["它", "那个", "刚才说的", "这个"]
        
        if any(p in message for p in pronouns):
            # 从历史中提取最近提到的实体
            recent_entities = self.extract_entities(history[-5:])
            
            # 替换指代词
            for entity in recent_entities:
                message = message.replace("它", entity, 1)
                break
        
        return message

# 示例
# 用户：Python 是什么？
# AI：Python 是一种编程语言...
# 用户：它有什么优点？  →  解析为：Python 有什么优点？
```

---

## 7.5 Agent 配���模板

### 客服 Agent 模板

```yaml
agent:
  name: "智能客服"
  system_prompt: |
    你是 {company_name} 的智能客服助手。
    
    ## 服务原则
    1. 耐心倾听，准确理解用户问题
    2. 专业解答，提供可操作的方案
    3. 及时转接，复杂问题不硬撑
    
    ## 处理流程
    1. 问候并确认用户问题
    2. 检索知识库寻找答案
    3. 如无法解决，询问是否转人工
    4. 记录问题并生成工单
    
    ## 转人工条件
    - 用户明确要求转人工
    - 问题涉及退款/投诉
    - 连续 3 次未能解决问题
  
  tools:
    - search_knowledge
    - create_ticket
    - query_order
    - transfer_to_human
  
  memory:
    short_term:
      max_messages: 10
    long_term:
      enabled: true
      categories:
        - user_preferences
        - support_history
```

### 写作助手 Agent 模板

```yaml
agent:
  name: "写作助手"
  system_prompt: |
    你是一位专业的写作助手，帮助用户创作各类内容。
    
    ## 写作风格
    - 根据用户指定的平台调整风格
    - 使用生动的例子说明观点
    - 保持段落简洁，每段 3-4 句话
    
    ## 创作流程
    1. 理解用户需求和目标受众
    2. 提供大纲供用户确认
    3. 分段创作，每段征求反馈
    4. 润色优化，检查错别字
    
    ## 输出格式
    - 标题使用 #
    - 小标题使用 ##
    - 列表使用 - 或 1. 2. 3.
    - 重点使用 **粗体**
  
  tools:
    - search_trends      # 搜索热点
    - check_grammar      # 语法检查
    - generate_image     # 生成配图
```

---

## 7.6 本章小结

本章讲解了 Agent 配置的核心要点：

1. **System Prompt**：定义 AI 的角色、能力、行为规范和限制条件
2. **记忆管理**：短期记忆维护对话上下文，长期记忆存储跨会话知识
3. **工具调用**：让 AI 能够与外部系统交互，扩展能力边界
4. **多轮对话**：状态机管理、上下文压缩、指代消解

**配置检查清单**：

- [ ] System Prompt 是否清晰定义了角色和能力边界？
- [ ] 记忆配置是否合理？短期记忆长度、长期记忆存储内容
- [ ] 工具定义是否完整？参数描述是否清晰？
- [ ] 是否有降级策略？当工具调用失败时如何处理？
- [ ] 是否设置了 Token 上限？防止上下文过长导致成本飙升

---

## 参考配置

```yaml
# config/agent.yaml - 完整示例
agents:
  - id: "default"
    name: "通用助手"
    system_prompt: |
      你是一个 helpful 的 AI 助手，用中文回答用户问题。
      
      ## 回答原则
      1. 直接回答，不绕弯子
      2. 不确定时诚实说明
      3. 复杂问题分步骤解释
      
      ## 输出格式
      - 使用 Markdown 格式
      - 代码使用 ``` 包裹
      - 关键信息使用 **粗体**
    
    model: "deepseek-chat"
    
    memory:
      short_term:
        max_messages: 20
        max_tokens: 4000
      long_term:
        enabled: true
        vector_store: "chroma"
    
    tools:
      - name: "query_weather"
      - name: "search_web"
```
