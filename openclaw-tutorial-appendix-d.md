# 附录D：故障排查指南

> 系统问题排查方法与常用命令

---

## 日志查看方法

### Docker 日志

```bash
# 查看实时日志
docker logs -f openclaw-core

# 查看最近 N 行
docker logs --tail=100 openclaw-core

# 查看特定时间段的日志
docker logs --since="2024-01-15T08:00:00" --until="2024-01-15T09:00:00" openclaw-core

# 查看所有服务日志
docker-compose logs -f

# 导出日志到文件
docker logs openclaw-core > openclaw.log 2>&1
```

### 日志级别过滤

```bash
# 只看错误日志
docker logs openclaw-core 2>&1 | grep ERROR

# 查看包含特定关键词的日志
docker logs openclaw-core 2>&1 | grep -i "webhook"

# 查看多关键词（或关系）
docker logs openclaw-core 2>&1 | grep -E "ERROR|WARN"
```

---

## 常见错误码速查

### HTTP 状态码

| 状态码 | 含义 | 排查方向 |
|--------|------|----------|
| 400 | 请求参数错误 | 检查请求体格式 |
| 401 | 未认证 | 检查 Token 是否过期 |
| 403 | 无权限 | 检查用户角色权限 |
| 404 | 资源不存在 | 检查 URL 路径 |
| 429 | 请求过于频繁 | 检查限流配置 |
| 500 | 服务器内部错误 | 查看服务器日志 |
| 502 | 网关错误 | 检查后端服务状态 |
| 503 | 服务不可用 | 检查服务是否启动 |

### 应用错误码

| 错误码 | 含义 | 解决方案 |
|--------|------|----------|
| `AUTH_001` | Token 无效 | 重新登录获取 Token |
| `AUTH_002` | Token 过期 | 使用 Refresh Token 刷新 |
| `DB_001` | 数据库连接失败 | 检查数据库配置 |
| `DB_002` | 查询超时 | 优化查询或增加超时时间 |
| `LLM_001` | LLM API 调用失败 | 检查 API Key 和网络 |
| `LLM_002` | Token 超限 | 减少上下文长度 |
| `CH_001` | 渠道配置错误 | 检查渠道凭证 |
| `CH_002` | Webhook 验证失败 | 检查签名密钥 |

---

## 网络问题排查

### 检查端口连通性

```bash
# 检查本地端口
netstat -tlnp | grep 3000
ss -tlnp | grep 3000

# 测试远程端口连通性
telnet localhost 3000
nc -zv localhost 3000

# 测试公网可访问性
curl -I http://your-domain.com:3000
```

### DNS 解析检查

```bash
# 检查域名解析
nslookup your-domain.com
dig your-domain.com

# 检查本地 hosts
cat /etc/hosts

# 刷新 DNS 缓存
sudo systemd-resolve --flush-caches  # Ubuntu
sudo killall -HUP mDNSResponder      # Mac
```

### 防火墙检查

```bash
# Linux
sudo iptables -L -n | grep 3000
sudo ufw status

# 云服务器安全组
# 登录云平台控制台检查安全组规则
```

---

## 数据库问题排查

### PostgreSQL

```bash
# 进入数据库容器
docker exec -it openclaw-postgres psql -U openclaw

# 常用诊断命令
\l                    # 列出数据库
\dt                   # 列出表
\du                   # 列出用户
SELECT * FROM pg_stat_activity;  # 查看活动连接

# 检查表大小
SELECT schemaname, tablename, 
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) 
FROM pg_tables 
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

# 检查慢查询
SELECT query, calls, mean_time, total_time 
FROM pg_stat_statements 
ORDER BY mean_time DESC 
LIMIT 10;
```

### Redis

```bash
# 进入 Redis 容器
docker exec -it openclaw-redis redis-cli

# 常用诊断命令
INFO                  # 查看服务器信息
INFO memory           # 查看内存使用
MONITOR               # 实时监控命令（慎用）
SLOWLOG GET 10        # 查看慢查询日志

# 检查键数量
DBSIZE

# 检查大键
redis-cli --bigkeys
```

---

## 性能问题排查

### 系统资源监控

```bash
# CPU 和内存
top
htop

# 磁盘使用
df -h
du -sh /var/lib/docker

# 内存详细使用
free -h
cat /proc/meminfo

# IO 监控
iotop
iostat -x 1
```

### Docker 资源使用

```bash
# 查看容器资源使用
docker stats

# 查看容器详情
docker inspect openclaw-core

# 查看容器进程
docker top openclaw-core
```

### 应用性能分析

```bash
# 进入容器
docker exec -it openclaw-core /bin/sh

# 安装调试工具
pip install py-spy

# CPU 火焰图
py-spy record -o profile.svg -- python main.py

# 实时查看
py-spy top -- python main.py
```

---

## 升级问题回滚

### 数据库回滚

```bash
# 1. 停止服务
docker-compose down

# 2. 恢复数据库备份
zcat backup_20240115.sql.gz | docker exec -i openclaw-postgres psql -U openclaw

# 3. 回滚到旧版本镜像
docker-compose pull openclaw:1.2.3

# 4. 启动服务
docker-compose up -d
```

### 配置回滚

```bash
# 使用 Git 管理配置
cd /opt/openclaw
git init
git add .
git commit -m "Initial config"

# 修改配置前创建分支
git checkout -b feature/new-config

# 需要回滚时
git checkout main
git branch -D feature/new-config
```

---

## 调试模式

### 启用调试日志

```yaml
# docker-compose.yml
services:
  openclaw:
    environment:
      - LOG_LEVEL=debug
      - DEBUG=true
```

### 进入容器调试

```bash
# 进入运行中的容器
docker exec -it openclaw-core /bin/sh

# 查看环境变量
env | grep -i openclaw

# 测试网络连通
ping google.com
wget https://api.openai.com

# 查看配置文件
cat /app/config/config.yaml
```

### Python 调试

```python
# 在代码中添加断点
import pdb; pdb.set_trace()

# 或使用 ipdb
import ipdb; ipdb.set_trace()
```

---

## 紧急恢复

### 服务完全无法启动

```bash
# 1. 清理所有容器和数据（谨慎操作！）
docker-compose down -v

# 2. 重新拉取镜像
docker-compose pull

# 3. 重新初始化
docker-compose up -d

# 4. 执行初始化命令
docker-compose exec openclaw python manage.py init
```

### 数据损坏恢复

```bash
# 1. 停止服务
docker-compose down

# 2. 备份当前数据（以防万一）
cp -r data data.bak.$(date +%Y%m%d)

# 3. 从备份恢复
cp -r /path/to/backup/data ./

# 4. 重启服务
docker-compose up -d
```

---

## 联系支持

如果以上方法无法解决问题：

1. **收集信息**：
   - 错误日志（脱敏后）
   - 配置文件（脱敏后）
   - 环境信息（OS、版本等）
   - 复现步骤

2. **提交 Issue**：
   - GitHub Issues: https://github.com/openclaw/openclaw/issues
   - 使用故障报告模板

3. **社区求助**：
   - Discord: https://discord.gg/openclaw
   - 飞书群（中文）

