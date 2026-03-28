# 附录C：常见问题 FAQ

> 部署、配置、渠道接入常见问题汇总

---

## 部署问题

### Q1: Docker 启动失败，提示端口被占用

**错误信息**：
```
Error starting userland proxy: listen tcp4 0.0.0.0:3000: bind: address already in use
```

**解决方案**：

```bash
# 查找占用端口的进程
sudo lsof -i :3000
# 或
sudo netstat -tlnp | grep 3000

# 停止占用进程
sudo kill -9 <PID>

# 或修改 docker-compose.yml 使用其他端口
ports:
  - "3001:3000"  # 主机3001映射到容器3000
```

### Q2: 容器启动后立即退出

**排查步骤**：

```bash
# 查看容器日志
docker logs openclaw-core

# 常见原因：
# 1. 数据库连接失败 - 检查 DATABASE_URL
# 2. 缺少必要环境变量 - 检查 .env 文件
# 3. 权限问题 - 检查数据目录权限

# 修复权限
chmod -R 755 ./data
```

### Q3: 国内拉取镜像速度慢

**解决方案**：

```bash
# 配置镜像加速
# 编辑 /etc/docker/daemon.json
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com"
  ]
}

# 重启 Docker
sudo systemctl restart docker
```

### Q4: 内存不足导致 OOM

**解决方案**：

```yaml
# docker-compose.yml 中限制内存
services:
  openclaw:
    deploy:
      resources:
        limits:
          memory: 2G
        reservations:
          memory: 1G
```

---

## 配置问题

### Q5: JWT 认证失败

**错误信息**：
```
Invalid token
```

**解决方案**：

```bash
# 1. 检查 JWT_SECRET 是否设置
# 2. 确保长度至少 32 位
# 3. 重新生成 Token

# 生成强密钥
openssl rand -base64 32
```

### Q6: 数据库连接失败

**错误信息**：
```
connection refused
```

**排查步骤**：

```bash
# 1. 检查数据库容器状态
docker ps | grep postgres

# 2. 检查连接字符串
# 确保使用容器名作为主机名
DATABASE_URL=postgresql://user:pass@postgres:5432/db

# 3. 测试连接
docker exec -it openclaw-postgres psql -U openclaw
```

### Q7: 环境变量不生效

**解决方案**：

```bash
# 1. 检查 .env 文件位置（应在 docker-compose.yml 同级目录）
# 2. 检查文件权限
chmod 600 .env

# 3. 重启容器使配置生效
docker-compose down && docker-compose up -d

# 4. 验证环境变量
docker exec openclaw-core env | grep JWT
```

---

## 渠道接入问题

### Q8: 飞书机器人无响应

**排查清单**：

1. ✅ 应用已发布并通过审批
2. ✅ 开通了 `im:message:send` 权限
3. ✅ 订阅了 `im.message.receive_v1` 事件
4. ✅ Webhook URL 配置正确且可公网访问
5. ✅ 加密密钥和验证 Token 匹配

**调试方法**：

```bash
# 查看 OpenClaw 日志
docker logs -f openclaw-core | grep feishu

# 测试 Webhook 可访问性
curl -X POST https://your-domain.com/api/webhooks/feishu/xxx \
  -H "Content-Type: application/json" \
  -d '{"test": true}'
```

### Q9: 微信公众号 Token 验证失败

**常见原因**：

1. **URL 不可访问** - 确保服务器可公网访问
2. **Token 不匹配** - 检查 OpenClaw 和公众号配置是否一致
3. **端口问题** - 微信要求 80/443 端口
4. **响应超时** - 确保 5 秒内响应

**解决方案**：

```python
# 确保正确处理 GET 请求（用于验证）
# 和 POST 请求（用于接收消息）

@app.route('/api/webhooks/wechat-mp/<id>', methods=['GET', 'POST'])
def wechat_webhook(id):
    if request.method == 'GET':
        # 处理验证请求
        return verify_signature(request.args)
    else:
        # 处理消息
        return handle_message(request.data)
```

### Q10: Telegram Bot 收不到消息

**排查步骤**：

```bash
# 1. 检查 Webhook 是否设置成功
curl https://api.telegram.org/bot<TOKEN>/getWebhookInfo

# 2. 重新设置 Webhook
curl -X POST "https://api.telegram.org/bot<TOKEN>/setWebhook" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://your-domain.com/api/webhooks/telegram/xxx"}'

# 3. 如果使用轮询模式，确保没有其他实例在运行
```

### Q11: 钉钉消息发送失败

**错误信息**：
```
{"errcode": 400002, "errmsg": "invalid timestamp"}
```

**解决方案**：

```bash
# 同步服务器时间
sudo ntpdate ntp.aliyun.com

# 或安装 ntp 服务
sudo apt-get install ntp
```

---

## LLM 接入问题

### Q12: OpenAI API 无法访问

**解决方案**：

```yaml
# 方案1：使用代理
models:
  - name: "gpt-4"
    config:
      base_url: "https://api.openai.com/v1"
      http_proxy: "http://proxy-server:port"

# 方案2：使用中转服务
models:
  - name: "gpt-4"
    config:
      base_url: "https://api.xxx.com/v1"  # 第三方中转

# 方案3：使用国产替代
models:
  - name: "deepseek"
    config:
      base_url: "https://api.deepseek.com/v1"
```

### Q13: LLM 响应超时

**解决方案**：

```yaml
# 增加超时时间
llm:
  timeout: 60  # 秒
  
  # 启用流式响应减少等待感
  streaming:
    enabled: true
```

### Q14: Token 用量过高

**优化建议**：

1. 启用上下文压缩
2. 使用更便宜的模型处理简单任务
3. 启用缓存
4. 限制最大 Token 数

```yaml
llm:
  max_tokens: 2000
  cache:
    enabled: true
```

---

## 性能问题

### Q15: 响应速度慢

**排查步骤**：

```bash
# 1. 检查资源使用
docker stats

# 2. 检查数据库慢查询
docker logs openclaw-postgres | grep "duration"

# 3. 检查 Redis 连接
redis-cli ping

# 4. 查看应用日志
docker logs openclaw-core | grep "slow"
```

**优化方案**：

```yaml
# 启用缓存
cache:
  enabled: true
  ttl: 3600

# 增加连接池
database:
  pool:
    max_size: 50
```

### Q16: 内存占用过高

**解决方案**：

```yaml
# 限制上下文长度
agent:
  memory:
    short_term:
      max_messages: 10
      max_tokens: 2000

# 定期清理日志
logging:
  rotation:
    max_size: 100MB
    max_files: 5
```

---

## 其他问题

### Q17: 如何备份数据？

```bash
# 备份脚本
#!/bin/bash
DATE=$(date +%Y%m%d)

# 备份数据库
docker exec openclaw-postgres pg_dump -U openclaw openclaw > backup_${DATE}.sql

# 备份 Redis
docker exec openclaw-redis redis-cli BGSAVE
cp data/redis/dump.rdb backup_redis_${DATE}.rdb

# 备份配置
tar -czf backup_config_${DATE}.tar.gz config/
```

### Q18: 如何查看日志？

```bash
# 实时查看日志
docker logs -f openclaw-core

# 查看最近100行
docker logs --tail=100 openclaw-core

# 查看特定时间
docker logs --since="2024-01-15T08:00:00" openclaw-core

# 导出日志
docker logs openclaw-core > openclaw.log 2>&1
```

### Q19: 如何更新到最新版本？

```bash
# 1. 备份数据
./backup.sh

# 2. 拉取最新镜像
docker-compose pull

# 3. 重启服务
docker-compose down
docker-compose up -d

# 4. 执行数据库迁移
docker-compose run --rm openclaw migrate
```

### Q20: 如何重置管理员密码？

```bash
# 进入容器
docker exec -it openclaw-core /bin/sh

# 运行重置脚本
python scripts/reset_password.py --username admin --password newpassword
```

---

## 获取帮助

如果以上 FAQ 无法解决你的问题：

1. **查看日志**：`docker logs openclaw-core`
2. **搜索 Issues**：https://github.com/openclaw/openclaw/issues
3. **加入社区**：Discord / 飞书群
4. **提交 Issue**：附上日志和复现步骤

