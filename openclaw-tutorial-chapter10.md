# 第10章：高级部署与运维

> 生产环境的高可用架构、监控告警与性能优化

---

## 10.1 高可用架构

### 架构设计

```mermaid
flowchart TB
    subgraph LB["负载均衡层"]
        N1["Nginx/Caddy"]
    end
    
    subgraph App["应用层"]
        A1["OpenClaw-1"]
        A2["OpenClaw-2"]
        A3["OpenClaw-3"]
    end
    
    subgraph Data["数据层"]
        P1[(PostgreSQL<br/>主)]
        P2[(PostgreSQL<br/>从)]
        R1[(Redis<br/>主)]
        R2[(Redis<br/>从)]
    end
    
    subgraph Queue["消息队列"]
        Q["Redis/RabbitMQ"]
    end
    
    N1 --> A1
    N1 --> A2
    N1 --> A3
    
    A1 --> P1
    A2 --> P1
    A3 --> P1
    A1 --> R1
    A2 --> R1
    A3 --> R1
    A1 --> Q
    A2 --> Q
    A3 --> Q
    
    P1 -.->|复制| P2
    R1 -.->|复制| R2
```

### Docker Swarm 部署

```yaml
# docker-stack.yml
version: '3.8'

services:
  openclaw:
    image: openclaw/openclaw:latest
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      resources:
        limits:
          cpus: '1.0'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 1G
    environment:
      - DATABASE_URL=postgresql://openclaw:password@postgres:5432/openclaw
      - REDIS_URL=redis://redis:6379/0
    networks:
      - openclaw-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  postgres:
    image: postgres:15-alpine
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
    environment:
      - POSTGRES_USER=openclaw
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=openclaw
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - openclaw-network

  redis:
    image: redis:7-alpine
    deploy:
      replicas: 1
    volumes:
      - redis-data:/data
    networks:
      - openclaw-network

  nginx:
    image: nginx:alpine
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    networks:
      - openclaw-network

volumes:
  postgres-data:
  redis-data:

networks:
  openclaw-network:
    driver: overlay
```

部署命令：

```bash
# 初始化 Swarm
docker swarm init

# 部署 Stack
docker stack deploy -c docker-stack.yml openclaw

# 查看服务状态
docker stack ps openclaw
docker service ls

# 扩容
docker service scale openclaw_openclaw=5
```

---

## 10.2 监控告警

### Prometheus + Grafana 监控

```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin

  node-exporter:
    image: prom/node-exporter:latest
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'

volumes:
  prometheus-data:
  grafana-data:
```

**prometheus.yml 配置**：

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'openclaw'
    static_configs:
      - targets: ['openclaw:3000']
    metrics_path: '/metrics'
  
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
  
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']
  
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
```

### 关键监控指标

| 指标 | 说明 | 告警阈值 |
|------|------|----------|
| `openclaw_requests_total` | 总请求数 | - |
| `openclaw_request_duration_seconds` | 请求延迟 | P99 > 2s |
| `openclaw_active_connections` | 活跃连接数 | > 1000 |
| `openclaw_llm_tokens_total` | LLM Token 消耗 | 日增长 > 50% |
| `postgres_connections` | 数据库连接数 | > 80% |
| `redis_memory_usage` | Redis 内存使用 | > 80% |

### 告警规则

```yaml
# alert-rules.yml
groups:
  - name: openclaw
    rules:
      - alert: HighErrorRate
        expr: rate(openclaw_requests_failed_total[5m]) > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "错误率过高"
          description: "错误率超过 10%"
      
      - alert: HighLatency
        expr: histogram_quantile(0.99, rate(openclaw_request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "延迟过高"
          description: "P99 延迟超过 2 秒"
      
      - alert: DatabaseConnectionsHigh
        expr: postgres_connections / postgres_max_connections > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "数据库连接数过高"
      
      - alert: LLMCostSpike
        expr: increase(openclaw_llm_tokens_total[1h]) > 1000000
        for: 1h
        labels:
          severity: info
        annotations:
          summary: "LLM Token 消耗异常"
```

---

## 10.3 日志管理

### ELK Stack 配置

```yaml
# docker-compose.logging.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    volumes:
      - es-data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    ports:
      - "5044:5044"

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.11.0
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    user: root

volumes:
  es-data:
```

### 结构化日志

```python
# 日志配置
import logging
import json
from pythonjsonlogger import jsonlogger

class CustomJsonFormatter(jsonlogger.JsonFormatter):
    def add_fields(self, log_record, record, message_dict):
        super().add_fields(log_record, record, message_dict)
        log_record['timestamp'] = record.created
        log_record['level'] = record.levelname
        log_record['logger'] = record.name

# 配置
logHandler = logging.StreamHandler()
formatter = CustomJsonFormatter('%(timestamp)s %(level)s %(name)s %(message)s')
logHandler.setFormatter(formatter)

logger = logging.getLogger('openclaw')
logger.addHandler(logHandler)
logger.setLevel(logging.INFO)

# 使用
logger.info('Request processed', extra={
    'request_id': 'req_xxx',
    'user_id': 'user_xxx',
    'duration_ms': 150,
    'channel': 'feishu'
})
```

---

## 10.4 性能优化

### 数据库优化

```yaml
# 连接池配置
database:
  pool:
    min_size: 5
    max_size: 20
    max_overflow: 10
    pool_timeout: 30
    pool_recycle: 3600
```

**索引优化**：

```sql
-- 常用查询索引
CREATE INDEX idx_messages_user_id ON messages(user_id);
CREATE INDEX idx_messages_channel_id ON messages(channel_id);
CREATE INDEX idx_messages_created_at ON messages(created_at);

-- 复合索引
CREATE INDEX idx_conversations_user_channel ON conversations(user_id, channel_id);

-- 全文搜索索引
CREATE INDEX idx_knowledge_content ON knowledge USING gin(to_tsvector('chinese', content));
```

### 缓存策略

```yaml
# Redis 缓存配置
cache:
  enabled: true
  ttl:
    user_session: 3600
    conversation_context: 300
    knowledge_search: 600
  
  # 缓存穿透防护
  null_ttl: 60
  
  # 布隆过滤器
  bloom_filter:
    enabled: true
    expected_items: 1000000
    false_positive_rate: 0.01
```

### LLM 优化

```yaml
# 请求合并
llm:
  batch:
    enabled: true
    max_size: 10
    max_wait_ms: 100
  
  # 缓存相似请求
  cache:
    enabled: true
    similarity_threshold: 0.95
    ttl: 3600
  
  # 流式响应
  streaming:
    enabled: true
    chunk_size: 100
```

---

## 10.5 备份恢复

### 自动化备份脚本

```bash
#!/bin/bash
# backup.sh

BACKUP_DIR="/backup/openclaw"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

# 创建备份目录
mkdir -p $BACKUP_DIR

# 备份 PostgreSQL
docker exec openclaw-postgres pg_dump -U openclaw openclaw | gzip > $BACKUP_DIR/postgres_$DATE.sql.gz

# 备份 Redis
docker exec openclaw-redis redis-cli BGSAVE
sleep 5
cp /var/lib/redis/dump.rdb $BACKUP_DIR/redis_$DATE.rdb

# 备份配置文件
tar -czf $BACKUP_DIR/config_$DATE.tar.gz /opt/openclaw/config

# 上传到云存储
aws s3 sync $BACKUP_DIR s3://your-backup-bucket/openclaw/

# 清理旧备份
find $BACKUP_DIR -name "*.gz" -mtime +$RETENTION_DAYS -delete
find $BACKUP_DIR -name "*.rdb" -mtime +$RETENTION_DAYS -delete

echo "Backup completed: $DATE"
```

### 灾难恢复演练

```bash
#!/bin/bash
# disaster-recovery.sh

# 1. 停止服务
docker-compose down

# 2. 恢复数据库
zcat /backup/openclaw/postgres_20240115_030000.sql.gz | docker exec -i openclaw-postgres psql -U openclaw

# 3. 恢复 Redis
cp /backup/openclaw/redis_20240115_030000.rdb /var/lib/redis/dump.rdb

# 4. 恢复配置
tar -xzf /backup/openclaw/config_20240115_030000.tar.gz -C /

# 5. 启动服务
docker-compose up -d

# 6. 验证
curl -f http://localhost:3000/health || exit 1

echo "Disaster recovery completed"
```

---

## 10.6 本章小结

本章讲解了生产环境的运维要点：

1. **高可用架构**：Docker Swarm、负载均衡、多实例部署
2. **监控告警**：Prometheus + Grafana、关键指标、告警规则
3. **日志管理**：ELK Stack、结构化日志
4. **性能优化**：数据库索引、Redis 缓存、LLM 优化
5. **备份恢复**：自动化备份、灾难恢复演练

**运维 checklist**：
- [ ] 配置健康检查
- [ ] 设置监控告警
- [ ] 配置日志收集
- [ ] 制定备份策略
- [ ] 定期灾难恢复演练
- [ ] 更新安全补丁

---

## 参考命令

```bash
# 查看服务状态
docker stack ps openclaw
docker service ls

# 查看日志
docker service logs openclaw_openclaw

# 扩容
docker service scale openclaw_openclaw=5

# 滚动更新
docker service update --image openclaw/openclaw:v2.0 openclaw_openclaw

# 备份
docker exec openclaw-postgres pg_dump -U openclaw openclaw > backup.sql
```
