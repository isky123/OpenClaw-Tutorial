# 第18章：性能优化与调优

> 延迟优化、吞吐量提升、成本控制与压测方法

---

## 18.1 延迟优化

### 响应时间分析

```mermaid
flowchart LR
    A["用户发送消息"] -->|10ms| B["渠道接收"]
    B -->|20ms| C["Gateway处理"]
    C -->|50ms| D["Agent处理"]
    D -->|1000ms| E["LLM推理"]
    E -->|20ms| F["返回用户"]
    
    style E fill:#ff9999
```

**各阶段优化策略**：

| 阶段 | 正常延迟 | 优化方法 |
|------|----------|----------|
| 渠道接收 | 10-50ms | 选择就近服务器 |
| Gateway | 5-20ms | 连接池、缓存 |
| Agent | 10-50ms | 异步处理、缓存 |
| LLM | 500-3000ms | 模型选择、流式输出 |

### 流式响应

```python
# 流式输出配置
llm:
  streaming:
    enabled: true
    chunk_size: 20  # 每 20 个 token 发送一次
    first_chunk_timeout: 500ms  # 首包超时
```

```python
# 实现流式响应
async def stream_response(self, message: str, channel: str):
    """流式发送 LLM 响应"""
    
    # 发送"正在输入"状态
    await self.send_typing_indicator(channel)
    
    # 流式获取 LLM 输出
    buffer = ""
    async for chunk in self.llm.astream(message):
        buffer += chunk
        
        # 按句子分割发送
        if any(c in chunk for c in [".", "!", "?", "。", "！", "？"]):
            await self.send_message(channel, buffer)
            buffer = ""
    
    # 发送剩余内容
    if buffer:
        await self.send_message(channel, buffer)
```

### 缓存优化

```yaml
# 多级缓存配置
cache:
  # L1: 内存缓存
  memory:
    enabled: true
    ttl: 60
    max_size: 10000
  
  # L2: Redis 缓存
  redis:
    enabled: true
    ttl: 3600
  
  # 缓存策略
  strategies:
    - pattern: "天气查询"
      ttl: 1800  # 30分钟
    
    - pattern: "知识库问答"
      ttl: 86400  # 24小时
      similarity: 0.95  # 相似度阈值
```

---

## 18.2 吞吐量优化

### 连接池配置

```yaml
# 数据库连接池
database:
  pool:
    min_size: 10
    max_size: 100
    max_overflow: 20
    pool_timeout: 30
    pool_recycle: 3600
    pool_pre_ping: true

# HTTP 连接池
http:
  pool:
    max_connections: 100
    max_keepalive: 20
```

### 异步处理

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

class AsyncProcessor:
    def __init__(self):
        self.executor = ThreadPoolExecutor(max_workers=10)
    
    async def process_batch(self, messages: List[Dict]):
        """批量异步处理"""
        
        # 创建任务列表
        tasks = [
            self.process_message(msg)
            for msg in messages
        ]
        
        # 并发执行
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        return results
    
    async def process_message(self, message: Dict):
        """处理单条消息"""
        
        # 在线程池中执行阻塞操作
        loop = asyncio.get_event_loop()
        result = await loop.run_in_executor(
            self.executor,
            self._blocking_operation,
            message
        )
        
        return result
```

### 批处理优化

```python
class BatchProcessor:
    def __init__(self, batch_size: int = 10, max_wait: float = 0.1):
        self.batch_size = batch_size
        self.max_wait = max_wait
        self.queue = asyncio.Queue()
        self.processing = False
    
    async def add(self, item: Dict) -> asyncio.Future:
        """添加任务到批次"""
        future = asyncio.Future()
        await self.queue.put((item, future))
        
        if not self.processing:
            asyncio.create_task(self._process_batch())
        
        return future
    
    async def _process_batch(self):
        """处理批次"""
        self.processing = True
        
        batch = []
        futures = []
        
        # 收集批次
        start_time = asyncio.get_event_loop().time()
        
        while len(batch) < self.batch_size:
            elapsed = asyncio.get_event_loop().time() - start_time
            timeout = max(0, self.max_wait - elapsed)
            
            try:
                item, future = await asyncio.wait_for(
                    self.queue.get(),
                    timeout=timeout
                )
                batch.append(item)
                futures.append(future)
            except asyncio.TimeoutError:
                break
        
        # 批量处理
        results = await self._batch_call(batch)
        
        # 设置结果
        for future, result in zip(futures, results):
            future.set_result(result)
        
        self.processing = False
```

---

## 18.3 成本控制

### Token 用量控制

```yaml
# Token 限制配置
llm:
  limits:
    # 单用户日限额
    per_user_daily: 100000
    
    # 单请求最大 Token
    per_request_max: 4000
    
    # 上下文压缩阈值
    context_compression_threshold: 3000
  
  # 模型路由
  routing:
    - condition: "complexity < 0.5"
      model: "deepseek-chat"  # 便宜模型
    - condition: "complexity >= 0.5"
      model: "gpt-4"  # 强模型
```

### 智能模型选择

```python
class SmartModelRouter:
    def __init__(self):
        self.models = {
            "light": {
                "name": "deepseek-chat",
                "cost_per_1k": 0.001,
                "strength": 0.7
            },
            "standard": {
                "name": "gpt-3.5-turbo",
                "cost_per_1k": 0.002,
                "strength": 0.85
            },
            "premium": {
                "name": "gpt-4",
                "cost_per_1k": 0.03,
                "strength": 0.95
            }
        }
    
    async def select_model(self, message: str, context: Dict) -> str:
        """智能选择模型"""
        
        # 分析复杂度
        complexity = await self._analyze_complexity(message)
        
        # 分析紧急程度
        urgency = context.get("urgency", 1)
        
        # 用户等级
        user_tier = context.get("user_tier", "standard")
        
        # 选择策略
        if user_tier == "premium":
            return self.models["premium"]["name"]
        
        if complexity > 0.8 or urgency > 3:
            return self.models["premium"]["name"]
        
        if complexity < 0.4:
            return self.models["light"]["name"]
        
        return self.models["standard"]["name"]
    
    async def _analyze_complexity(self, message: str) -> float:
        """分析消息复杂度"""
        
        factors = {
            "length": min(len(message) / 500, 1.0),
            "questions": message.count("?") + message.count("？") * 0.1,
            "technical_terms": self._count_technical_terms(message) * 0.2
        }
        
        return min(sum(factors.values()), 1.0)
```

### 缓存命中率优化

```python
class SmartCache:
    def __init__(self):
        self.cache = {}
        self.embeddings = {}
    
    async def get_with_semantic_match(
        self,
        query: str,
        threshold: float = 0.95
    ) -> Optional[str]:
        """语义匹配缓存"""
        
        # 获取查询的 embedding
        query_embedding = await self._get_embedding(query)
        
        # 查找最相似的缓存
        best_match = None
        best_score = 0
        
        for key, key_embedding in self.embeddings.items():
            similarity = self._cosine_similarity(
                query_embedding,
                key_embedding
            )
            
            if similarity > threshold and similarity > best_score:
                best_score = similarity
                best_match = key
        
        if best_match:
            return self.cache[best_match]
        
        return None
    
    def _cosine_similarity(self, a: List[float], b: List[float]) -> float:
        """计算余弦相似度"""
        import numpy as np
        
        a = np.array(a)
        b = np.array(b)
        
        return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
```

---

## 18.4 压测与基准

### 压测工具配置

```yaml
# k6 压测脚本
# load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },  //  ramp up
    { duration: '5m', target: 100 },  //  steady
    { duration: '2m', target: 200 },  //  ramp up
    { duration: '5m', target: 200 },  //  steady
    { duration: '2m', target: 0 },    //  ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<2000'],  // 95% 请求 < 2s
    http_req_failed: ['rate<0.1'],       // 错误率 < 10%
  },
};

export default function () {
  const payload = JSON.stringify({
    message: '你好',
    user_id: `user_${__VU}`,
    channel: 'test'
  });
  
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${__ENV.API_TOKEN}`
  };
  
  const res = http.post(
    'https://your-openclaw.com/api/chat',
    payload,
    { headers }
  );
  
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 2s': (r) => r.timings.duration < 2000,
  });
  
  sleep(1);
}
```

### 运行压测

```bash
# 安装 k6
brew install k6  # Mac
# 或下载二进制包

# 运行压测
k6 run --env API_TOKEN=your_token load-test.js

# 输出报告
k6 run --out json=results.json load-test.js
```

### 性能基准

```python
# 性能测试基线
PERFORMANCE_BASELINE = {
    "response_time": {
        "p50": 500,    # 50% 请求 < 500ms
        "p95": 2000,   # 95% 请求 < 2s
        "p99": 5000    # 99% 请求 < 5s
    },
    "throughput": {
        "rps": 100,     # 每秒 100 请求
        "concurrent": 1000  # 并发 1000
    },
    "error_rate": {
        "max": 0.01     # 最大 1% 错误率
    },
    "resource_usage": {
        "cpu_max": 80,      # CPU 最大 80%
        "memory_max": 80,   # 内存最大 80%
        "db_connections_max": 80  # DB 连接最大 80%
    }
}
```

---

## 18.5 监控告警

### 性能指标监控

```yaml
# 性能告警规则
alerts:
  - name: HighLatency
    expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
    for: 5m
    severity: warning
    
  - name: HighErrorRate
    expr: rate(http_requests_failed_total[5m]) / rate(http_requests_total[5m]) > 0.05
    for: 5m
    severity: critical
    
  - name: HighCost
    expr: increase(llm_tokens_total[1h]) > 1000000
    for: 1h
    severity: info
```

### 自动扩缩容

```yaml
# K8s HPA 配置
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: openclaw-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: openclaw
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

---

## 18.6 本章小结

本章讲解了性能优化的核心要点：

1. **延迟优化**：流式响应、多级缓存、连接池
2. **吞吐量优化**：异步处理、批处理、连接池
3. **成本控制**：Token 限制、智能模型选择、语义缓存
4. **压测基准**：k6 压测、性能基线定义
5. **监控告警**：性能指标、自动扩缩容

**优化 checklist**：
- [ ] 启用流式响应
- [ ] 配置多级缓存
- [ ] 优化数据库连接池
- [ ] 设置 Token 限额
- [ ] 配置智能模型路由
- [ ] 定期压测验证
- [ ] 配置性能告警
- [ ] 启用自动扩缩容

---

## 参考命令

```bash
# 运行压测
k6 run load-test.js

# 查看性能指标
curl http://localhost:3000/metrics

# 分析慢查询
docker logs openclaw-postgres | grep "duration"

# 监控资源使用
docker stats
```
