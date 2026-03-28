# 第12章：实战一——个人 AI 助手

> 打造 7×24 小时在线的个人智能助手

---

## 12.1 需求分析

### 场景定义

打造一个专属的个人 AI 助手，整合日常信息获取、任务管理、知识问答等功能，在多个聊天平台上随时可用。

### 功能清单

| 功能模块 | 具体功能 | 优先级 |
|----------|----------|--------|
| 每日早报 | 定时推送新闻摘要、天气、日程 | P0 |
| 待办管理 | 自然语言添加任务、到期提醒 | P0 |
| 知识问答 | 基于个人笔记的 RAG 问答 | P1 |
| 生活助手 | 天气、快递、菜谱查询 | P1 |
| 内容收藏 | 保存文章、生成摘要 | P2 |

### 架构设计

```mermaid
flowchart TB
    subgraph PA["👤 个人 AI 助手"]
        direction TB
        
        subgraph Channels["📱 接入渠道"]
            C1["微信 - 移动端"]
            C2["飞书 - PC端"]
        end
        
        subgraph Agent["🤖 Agent 配置"]
            A1["角色：个人助理小助手"]
            A2["记忆：个人偏好、历史对话"]
            A3["工具：天气、日历、搜索、笔记"]
        end
        
        subgraph Workflows["⚙️ 工作流"]
            W1["定时早报工作流"]
            W2["待办提醒工作流"]
            W3["内容收藏工作流"]
        end
        
        subgraph Deploy["🖥️ 部署方案"]
            D1["家庭服务器 / NAS"]
            D2["云服务器 2核4G"]
        end
    end
    
    Channels --> Agent
    Agent --> Workflows
```

---

## 12.2 环境准备

### 服务器选择

对于个人使用，推荐以下方案：

| 方案 | 成本 | 稳定性 | 适合人群 |
|------|------|--------|----------|
| 阿里云轻量应用服务器 | 约 50元/月 | 高 | 追求稳定 |
| 腾讯云 CVM | 约 60元/月 | 高 | 追求稳定 |
| 家庭 NAS | 一次性投入 | 中 | 已有 NAS |
| 树莓派 | 一次性投入 | 低 | 极客玩家 |

### 部署配置

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: personal-assistant-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: assistant
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: assistant
    volumes:
      - ./data/postgres:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    container_name: personal-assistant-cache
    restart: unless-stopped
    volumes:
      - ./data/redis:/data

  openclaw:
    image: openclaw/openclaw:latest
    container_name: personal-assistant
    restart: unless-stopped
    depends_on:
      - postgres
      - redis
    environment:
      DATABASE_URL: postgresql://assistant:${DB_PASSWORD}@postgres:5432/assistant
      REDIS_URL: redis://redis:6379/0
      JWT_SECRET: ${JWT_SECRET}
      ENCRYPTION_KEY: ${ENCRYPTION_KEY}
      DEEPSEEK_API_KEY: ${DEEPSEEK_API_KEY}
    ports:
      - "3000:3000"
    volumes:
      - ./data/uploads:/app/uploads
      - ./config:/app/config
      - ./logs:/app/logs
```

### 配置文件

```yaml
# config/agent.yaml
agents:
  - id: "personal-assistant"
    name: "小助手"
    system_prompt: |
      你是我的个人 AI 助手，帮助处理日常事务。
      
      ## 基本信息
      - 名字：小助手
      - 性格：友好、高效、贴心
      
      ## 核心能力
      1. 每日早报：早上 8 点推送新闻、天气、日程
      2. 待办管理：记录任务、设置提醒
      3. 知识问答：基于我的笔记回答问题
      4. 生活查询：天气、快递、菜谱等
      
      ## 交互风格
      - 回复简洁，不啰嗦
      - 主动提供下一步建议
      - 记住我的偏好和习惯
      
      ## 快捷指令
      - "早报" → 立即发送今日早报
      - "待办" → 列出今日待办
      - "记住 xxx" → 记录重要信息
      
    model: "deepseek-chat"
    
    memory:
      short_term:
        max_messages: 20
      long_term:
        enabled: true
        categories:
          - user_preferences
          - important_facts
          - daily_summaries
```

---

## 12.3 每日早报功能

### 早报内容规划

```
📰 早安！今天是 2024年1月15日 星期一

🌤️ 今日天气
北京：晴，-2°C ~ 8°C，空气质量良
建议：适合户外运动，注意保暖

📅 今日日程
• 09:00 团队周会
• 14:00 项目评审
• 16:00 健身

📰 热点新闻
1. [科技] xxxx
2. [财经] xxxx
3. [国际] xxxx

💡 每日一句
"xxxxxx"
```

### 早报工作流

```yaml
# workflows/morning-briefing.yaml
workflow:
  name: "每日早报"
  trigger:
    type: cron
    schedule: "0 8 * * *"  # 每天 8:00
  
  steps:
    - name: "获取天气"
      type: skill
      skill: "weather"
      input:
        city: "{{user.preferences.city}}"
    
    - name: "获取日程"
      type: skill
      skill: "calendar"
      input:
        date: "today"
    
    - name: "获取新闻"
      type: skill
      skill: "news"
      input:
        categories: ["tech", "finance", "world"]
        limit: 5
    
    - name: "获取每日一句"
      type: skill
      skill: "daily_quote"
    
    - name: "生成早报"
      type: llm
      prompt: |
        请根据以下信息生成早报：
        
        天气：{{steps.获取天气.output}}
        日程：{{steps.获取日程.output}}
        新闻：{{steps.���取新闻.output}}
        每日一句：{{steps.获取每日一句.output}}
        
        格式要求：
        - 使用 emoji 增加可读性
        - 天气部分给出穿衣建议
        - 新闻只保留标题和一句话摘要
        - 整体控制在 500 字以内
    
    - name: "发送消息"
      type: channel
      channel: "wechat"  # 或 feishu
      message: "{{steps.生成早报.output}}"
```

### 天气查询 Skill

```python
# skills/weather/weather.py
import requests
from typing import Dict, Any

class WeatherSkill:
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://api.seniverse.com/v3"
    
    async def get_weather(self, city: str) -> Dict[str, Any]:
        """获取天气信息"""
        url = f"{self.base_url}/weather/daily.json"
        params = {
            "key": self.api_key,
            "location": city,
            "language": "zh-Hans",
            "unit": "c"
        }
        
        response = requests.get(url, params=params)
        data = response.json()
        
        today = data["results"][0]["daily"][0]
        
        return {
            "city": city,
            "date": today["date"],
            "weather": today["text_day"],
            "temperature_high": today["high"],
            "temperature_low": today["low"],
            "humidity": today["humidity"],
            "wind": today["wind_direction"],
            "suggestion": self._get_suggestion(today)
        }
    
    def _get_suggestion(self, weather: Dict) -> str:
        """根据天气给出建议"""
        high = int(weather["high"])
        
        if high < 5:
            return "天气寒冷，注意保暖，建议穿羽绒服"
        elif high < 15:
            return "天气较凉，建议穿外套"
        elif high < 25:
            return "温度适宜，适合户外活动"
        else:
            return "天气炎热，注意防暑降温"

# 注册
manifest = {
    "name": "weather",
    "version": "1.0.0",
    "tools": [
        {
            "name": "get_weather",
            "handler": WeatherSkill.get_weather,
            "description": "获取指定城市的天气信息"
        }
    ]
}
```

### 新闻聚合 Skill

```python
# skills/news/news.py
import feedparser
from typing import List, Dict

class NewsSkill:
    def __init__(self):
        self.sources = {
            "tech": [
                "https://www.36kr.com/feed",
                "https://www.pingwest.com/feed"
            ],
            "finance": [
                "https://www.36kr.com/feed/newsflashes"
            ],
            "world": [
                "https://feeds.bbci.co.uk/news/world/rss.xml"
            ]
        }
    
    async def get_news(self, categories: List[str], limit: int = 5) -> List[Dict]:
        """获取新闻"""
        news = []
        
        for category in categories:
            if category not in self.sources:
                continue
            
            for feed_url in self.sources[category]:
                feed = feedparser.parse(feed_url)
                for entry in feed.entries[:2]:  # 每个源取 2 条
                    news.append({
                        "category": category,
                        "title": entry.title,
                        "summary": entry.summary[:100] + "...",
                        "link": entry.link
                    })
        
        # 去重并限制数量
        seen = set()
        unique_news = []
        for item in news:
            if item["title"] not in seen and len(unique_news) < limit:
                seen.add(item["title"])
                unique_news.append(item)
        
        return unique_news
```

---

## 12.4 待办管理功能

### 自然语言解析

```python
# skills/todo/todo.py
import re
from datetime import datetime, timedelta
from typing import Dict, Any

class TodoSkill:
    def __init__(self, db):
        self.db = db
    
    async def parse_todo(self, message: str) -> Dict[str, Any]:
        """解析自然语言待办"""
        
        # 提取任务内容
        patterns = [
            r"(?:记得|记住|提醒我|待办).{0,3}?(.*?)(?:在|于|$)",
            r"(.*?)\s*(?:什么时候|何时)",
        ]
        
        content = None
        for pattern in patterns:
            match = re.search(pattern, message)
            if match:
                content = match.group(1).strip()
                break
        
        if not content:
            content = message
        
        # 提取时间
        due_date = self._extract_time(message)
        
        return {
            "content": content,
            "due_date": due_date,
            "created_at": datetime.now()
        }
    
    def _extract_time(self, message: str) -> datetime:
        """从消息中提取时间"""
        now = datetime.now()
        
        # 今天
        if "今天" in message:
            return now.replace(hour=18, minute=0)
        
        # 明天
        if "明天" in message:
            return (now + timedelta(days=1)).replace(hour=9, minute=0)
        
        # 后天
        if "后天" in message:
            return (now + timedelta(days=2)).replace(hour=9, minute=0)
        
        # 下周
        if "下周" in message:
            days_until_monday = 7 - now.weekday()
            return (now + timedelta(days=days_until_monday)).replace(hour=9, minute=0)
        
        # 具体时间点：下午3点、15:00
        time_patterns = [
            r"(?:下午|晚上)(\d+)点",
            r"(\d{1,2}):(\d{2})",
        ]
        
        for pattern in time_patterns:
            match = re.search(pattern, message)
            if match:
                hour = int(match.group(1))
                if "下午" in message or "晚上" in message:
                    hour += 12
                return now.replace(hour=hour, minute=0)
        
        # 默认今天下班前
        return now.replace(hour=18, minute=0)
    
    async def add_todo(self, user_id: str, message: str) -> str:
        """添加待办"""
        todo = await self.parse_todo(message)
        todo["user_id"] = user_id
        
        # 保存到数据库
        await self.db.todos.insert_one(todo)
        
        # 设置提醒
        await self.schedule_reminder(todo)
        
        return f"✅ 已添加待办：{todo['content']}\n⏰ 截止时间：{todo['due_date'].strftime('%m月%d日 %H:%M')}"
    
    async def list_todos(self, user_id: str, filter_type: str = "today") -> str:
        """列出待办"""
        now = datetime.now()
        
        if filter_type == "today":
            start = now.replace(hour=0, minute=0)
            end = now.replace(hour=23, minute=59)
        elif filter_type == "week":
            start = now
            end = now + timedelta(days=7)
        else:
            start = now
            end = now + timedelta(days=30)
        
        todos = await self.db.todos.find({
            "user_id": user_id,
            "due_date": {"$gte": start, "$lte": end},
            "completed": {"$ne": True}
        }).sort("due_date", 1).to_list(length=20)
        
        if not todos:
            return "🎉 暂无待办事项，享受自由时光吧！"
        
        result = f"📋 {'今日' if filter_type == 'today' else '本周'}待办 ({len(todos)}项)：\n\n"
        for i, todo in enumerate(todos, 1):
            time_str = todo["due_date"].strftime("%H:%M")
            result += f"{i}. {todo['content']} ({time_str})\n"
        
        return result
```

### 待办提醒工作流

```yaml
# workflows/todo-reminder.yaml
workflow:
  name: "待办提醒"
  trigger:
    type: cron
    schedule: "0 9,15,18 * * *"  # 每天 9:00, 15:00, 18:00
  
  steps:
    - name: "查询即将到期的待办"
      type: database
      query: |
        SELECT * FROM todos 
        WHERE completed = false 
        AND due_date BETWEEN NOW() AND NOW() + INTERVAL '2 hours'
        ORDER BY due_date ASC
    
    - name: "发送提醒"
      type: channel
      for_each: "{{steps.查询即将到期的待办.results}}"
      channel: "{{item.user_channel}}"
      message: |
        ⏰ 待办提醒
        
        {{item.content}}
        截止时间：{{item.due_date}}
        
        完成后请回复"完成 {{item.id}}"
```

---

## 12.5 知识库问答

### 个人笔记接入

```python
# skills/knowledge/knowledge.py
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings
from langchain.text_splitter import MarkdownTextSplitter

class KnowledgeSkill:
    def __init__(self, persist_dir: str):
        self.embeddings = OpenAIEmbeddings(
            model="text-embedding-3-small"
        )
        self.vectorstore = Chroma(
            persist_directory=persist_dir,
            embedding_function=self.embeddings
        )
        self.text_splitter = MarkdownTextSplitter(
            chunk_size=500,
            chunk_overlap=50
        )
    
    async def add_document(self, content: str, metadata: dict):
        """添加文档到知识库"""
        # 分割文档
        chunks = self.text_splitter.split_text(content)
        
        # 添加到向量数据库
        self.vectorstore.add_texts(
            texts=chunks,
            metadatas=[metadata] * len(chunks)
        )
        
        return len(chunks)
    
    async def query(self, question: str, top_k: int = 3) -> str:
        """基于知识库回答问题"""
        # 检索相关文档
        docs = self.vectorstore.similarity_search(
            question,
            k=top_k
        )
        
        if not docs:
            return "在我的笔记中没有找到相关信息。"
        
        # 构建上下文
        context = "\n\n".join([
            f"[来自 {doc.metadata.get('source', '未知')}]:\n{doc.page_content}"
            for doc in docs
        ])
        
        return {
            "context": context,
            "sources": [doc.metadata for doc in docs]
        }
```

### 笔记同步

```yaml
# workflows/sync-notes.yaml
workflow:
  name: "同步笔记"
  trigger:
    type: cron
    schedule: "0 2 * * *"  # 每天凌晨 2 点
  
  steps:
    - name: "从 Obsidian 同步"
      type: skill
      skill: "obsidian_sync"
      input:
        vault_path: "/path/to/obsidian/vault"
    
    - name: "从 Notion 同步"
      type: skill
      skill: "notion_sync"
      input:
        database_id: "xxx"
    
    - name: "更新向量数据库"
      type: skill
      skill: "knowledge"
      action: "rebuild_index"
```

---

## 12.6 快捷指令

### 指令定义

```yaml
# config/commands.yaml
commands:
  - trigger: "早报"
    action: "workflow.run"
    workflow: "morning-briefing"
    description: "立即发送今日早报"
  
  - trigger: "待办"
    action: "skill.call"
    skill: "todo"
    method: "list_todos"
    description: "列出今日待办"
  
  - trigger: "记住"
    action: "skill.call"
    skill: "knowledge"
    method: "add_memory"
    description: "记录重要信息"
  
  - trigger: "天气"
    action: "skill.call"
    skill: "weather"
    method: "get_weather"
    description: "查询天气"
  
  - trigger: "帮助"
    action: "static.response"
    message: |
      可用指令：
      • 早报 - 发送今日早报
      • 待办 - 列出今日待办
      • 记住 xxx - 记录重要信息
      • 天气 - 查询天气
      • 搜索 xxx - 搜索笔记
```

---

## 12.7 完整配置

```yaml
# config/personal-assistant.yaml
# 完整个人助手配置

agent:
  id: "personal-assistant"
  name: "小助手"
  system_prompt: |
    你是我的个人 AI 助手。
    
    ## 快捷指令
    - 输入"早报"获取今日早报
    - 输入"待办"列出今日任务
    - 输入"记住 xxx"记录信息
    
  model: "deepseek-chat"
  
  tools:
    - weather
    - todo
    - news
    - knowledge
    - calendar
  
  memory:
    short_term:
      max_messages: 20
    long_term:
      enabled: true

workflows:
  - name: "morning-briefing"
    schedule: "0 8 * * *"
  
  - name: "todo-reminder"
    schedule: "0 9,15,18 * * *"
  
  - name: "sync-notes"
    schedule: "0 2 * * *"

channels:
  - name: "微信"
    type: wechat_mp
    enabled: true
  
  - name: "飞书"
    type: feishu
    enabled: true
```

---

## 12.8 本章小结

本章实战构建了一个个人 AI 助手，包含：

1. **每日早报**：定时推送天气、日程、新闻
2. **待办管理**：自然语言添加、到期提醒
3. **知识问答**：基于个人笔记的 RAG
4. **快捷指令**：简化常用操作

**关键配置**：
- 使用 DeepSeek 等性价比高的模型控制成本
- 设置定时工作流实现自动化
- 利用长期记忆记住用户偏好

**优化建议**：
- 早报内容可根据个人兴趣定制
- 待办可与日历应用同步
- 知识库可接入更多数据源（邮件、聊天记录等）
