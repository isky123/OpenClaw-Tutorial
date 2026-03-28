# 第3章：部署指南（多环境覆盖）

> 从本地开发到生产环境的完整部署方案

---

## 3.1 本地开发环境

对于开发测试或个人轻度使用，在本地机器上部署是最快的方式。

### macOS 部署

**使用 Docker Desktop**

```bash
# 1. 安装 Docker Desktop
# 从 https://www.docker.com/products/docker-desktop 下载安装

# 2. 创建项目目录
mkdir -p ~/openclaw && cd ~/openclaw

# 3. 创建 docker-compose.yml（使用第2章的配置）

# 4. 启动服务
docker compose up -d

# 5. 访问 http://localhost:3000
```

**使用 OrbStack（推荐，更轻量）**

```bash
# 安装 OrbStack
brew install orbstack

# 其余步骤与 Docker Desktop 相同
# OrbStack 性能更好，资源占用更低
```

### Windows 部署

**使用 Docker Desktop for Windows**

```powershell
# 1. 安装 Docker Desktop
# 从官网下载安装程序，按向导安装

# 2. 确保 WSL2 已启用（Docker Desktop 会提示安装）

# 3. 在 PowerShell 中执行
cd ~
mkdir openclaw
cd openclaw

# 4. 创建 docker-compose.yml 文件

# 5. 启动
docker compose up -d
```

⚠️ **Windows 用户注意**：
- 确保 BIOS 中已启用虚拟化
- Windows 10 需要 1903 或更高版本
- 建议使用 WSL2 后端（在 Docker Desktop 设置中开启）

### Linux 部署

**Ubuntu/Debian**

```bash
# 1. 安装 Docker
curl -fsSL https://get.docker.com | sh

# 2. 安装 Docker Compose
sudo apt-get install docker-compose-plugin

# 3. 将当前用户加入 docker 组（免 sudo）
sudo usermod -aG docker $USER
newgrp docker

# 4. 创建项目并启动
mkdir -p ~/openclaw && cd ~/openclaw
# 创建 docker-compose.yml 后启动
docker compose up -d
```

**CentOS/RHEL**

```bash
# 1. 安装 Docker
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker

# 2. 安装 Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# 3. 后续步骤与 Ubuntu 相同
```

---

## 3.2 云服务器部署

对于需要 7×24 小时在线的生产环境，云服务器是最佳选择。

### 服务器选购建议

| 用途 | 配置建议 | 月成本（参考） |
|------|----------|----------------|
| 个人测试 | 2核4G | 50-100 元 |
| 小型团队 | 4核8G | 100-200 元 |
| 企业应用 | 4核16G 或更高 | 300-800 元 |

**国内云厂商推荐**：
- 阿里云 ECS
- 腾讯云 CVM
- 华为云 ECS
- 百度云 BCC

**海外云厂商**（如需接入海外平台）：
- AWS Lightsail（新加坡节点，国内访问快）
- Vultr
- DigitalOcean

### 阿里云部署示例

```bash
# 1. 购买 ECS 实例后，SSH 登录
ssh root@your-server-ip

# 2. 更新系统
apt-get update && apt-get upgrade -y

# 3. 安装 Docker
curl -fsSL https://get.docker.com | sh
systemctl start docker
systemctl enable docker

# 4. 创建项目目录
mkdir -p /opt/openclaw && cd /opt/openclaw

# 5. 创建 docker-compose.yml
# 注意：将 APP_HOST 改为 0.0.0.0，确保外网可访问

# 6. 启动服务
docker compose up -d

# 7. 配置安全组
# 在阿里云控制台，进入 ECS 安全组配置
# 添加规则：允许 3000 端口入站（建议后续配置 Nginx + HTTPS）
```

### 自动启动配置

创建 systemd 服务，确保服务器重启后自动启动：

```bash
# 创建服务文件
sudo tee /etc/systemd/system/openclaw.service > /dev/null <<EOF
[Unit]
Description=OpenClaw Service
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/openclaw
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
EOF

# 启用服务
sudo systemctl daemon-reload
sudo systemctl enable openclaw
sudo systemctl start openclaw

# 查看状态
sudo systemctl status openclaw
```

---

## 3.3 家庭服务器/NAS 部署

对于追求数据自主的用户，在家庭服务器或 NAS 上部署是理想选择。

### 群晖 NAS 部署

**使用 Container Manager（Docker）**

1. 打开「Container Manager」套件
2. 点击「项目」→「新建"
3. 项目名称：`openclaw`
4. 路径：选择一个共享文件夹（如 `/docker/openclaw`）
5. 粘贴 docker-compose.yml 内容
6. 点击「下一步」→「完成"

**使用 SSH 命令行**

```bash
# 1. 在群晖控制面板开启 SSH

# 2. SSH 登录群晖
ssh admin@your-nas-ip

# 3. 切换到 root
sudo -i

# 4. 创建目录
cd /volume1/docker
mkdir openclaw && cd openclaw

# 5. 创建 docker-compose.yml 并启动
# 注意：群晖的 Docker 命令是 `docker-compose` 而非 `docker compose`
docker-compose up -d
```

⚠️ **群晖注意事项**：
- 确保共享文件夹有足够的权限
- 建议将数据目录映射到共享文件夹，便于备份
- 群晖的防火墙需要放行对应端口

### 威联通 NAS 部署

```bash
# 1. 安装 Container Station

# 2. SSH 登录
ssh admin@your-nas-ip

# 3. 创建项目目录
mkdir -p /share/Container/openclaw
cd /share/Container/openclaw

# 4. 创建 docker-compose.yml 并启动
```

### 自组 NAS / 树莓派部署

对于使用 Ubuntu/Debian 的自组 NAS 或树莓派，部署方式与普通 Linux 服务器相同。

**树莓派特别说明**：

```bash
# 树莓派需要使用 ARM 架构镜像
# 修改 docker-compose.yml 中的镜像标签
services:
  openclaw:
    image: openclaw/openclaw:latest-arm64  # ARM64 版本
    # 或
    image: openclaw/openclaw:latest-armv7  # ARMv7 版本（树莓派3）
```

⚠️ **树莓派性能限制**：
- 建议使用树莓派 4B（4GB 内存以上）
- 首次启动较慢，需要耐心等待
- 不建议同时运行太多渠道

---

## 3.4 K8s 集群部署

对于需要高可用、弹性伸缩的企业场景，Kubernetes 是最佳选择。

### 使用 Helm 部署

```bash
# 1. 添加 OpenClaw Helm 仓库
helm repo add openclaw https://charts.openclaw.io
helm repo update

# 2. 创建命名空间
kubectl create namespace openclaw

# 3. 安装（使用默认配置）
helm install openclaw openclaw/openclaw -n openclaw

# 4. 查看状态
kubectl get pods -n openclaw
```

### 自定义配置安装

创建 `values.yaml`：

```yaml
# values.yaml
replicaCount: 2

image:
  repository: openclaw/openclaw
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 3000

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
  hosts:
    - host: openclaw.yourdomain.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: openclaw-tls
      hosts:
        - openclaw.yourdomain.com

resources:
  limits:
    cpu: 1000m
    memory: 2Gi
  requests:
    cpu: 500m
    memory: 1Gi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

# 数据库配置（使用外部 PostgreSQL）
externalDatabase:
  enabled: true
  host: postgres.example.com
  port: 5432
  database: openclaw
  username: openclaw
  password: "your-password"

# Redis 配置（使用外部 Redis）
externalRedis:
  enabled: true
  host: redis.example.com
  port: 6379
```

安装：

```bash
helm install openclaw openclaw/openclaw -n openclaw -f values.yaml
```

### 原生 YAML 部署

如果不使用 Helm，可以直接应用 YAML：

```yaml
# openclaw-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openclaw
  namespace: openclaw
spec:
  replicas: 2
  selector:
    matchLabels:
      app: openclaw
  template:
    metadata:
      labels:
        app: openclaw
    spec:
      containers:
      - name: openclaw
        image: openclaw/openclaw:latest
        ports:
        - containerPort: 3000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: openclaw-secrets
              key: database-url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: openclaw-secrets
              key: redis-url
        resources:
          limits:
            memory: "2Gi"
            cpu: "1000m"
          requests:
            memory: "1Gi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: openclaw
  namespace: openclaw
spec:
  selector:
    app: openclaw
  ports:
  - port: 3000
    targetPort: 3000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: openclaw
  namespace: openclaw
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - openclaw.yourdomain.com
    secretName: openclaw-tls
  rules:
  - host: openclaw.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: openclaw
            port:
              number: 3000
```

应用：

```bash
kubectl apply -f openclaw-deployment.yaml
```

---

## 3.5 反向代理与 HTTPS

生产环境必须使用 HTTPS，以下是两种主流方案。

### Nginx 反向代理

**安装 Nginx**

```bash
# Ubuntu/Debian
sudo apt-get install nginx

# CentOS
sudo yum install nginx
```

**配置反向代理**

创建 `/etc/nginx/sites-available/openclaw`：

```nginx
server {
    listen 80;
    server_name openclaw.yourdomain.com;
    
    # 重定向到 HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name openclaw.yourdomain.com;

    # SSL 证书（使用 Let's Encrypt 或自签名）
    ssl_certificate /etc/letsencrypt/live/openclaw.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/openclaw.yourdomain.com/privkey.pem;
    
    # SSL 优化
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;

    # 日志
    access_log /var/log/nginx/openclaw.access.log;
    error_log /var/log/nginx/openclaw.error.log;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        
        # WebSocket 支持
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # 转发真实 IP
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # 超时设置
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

启用配置：

```bash
# 创建软链接
sudo ln -s /etc/nginx/sites-available/openclaw /etc/nginx/sites-enabled/

# 测试配置
sudo nginx -t

# 重载配置
sudo systemctl reload nginx
```

**使用 Let's Encrypt 获取免费证书**

```bash
# 安装 Certbot
sudo apt-get install certbot python3-certbot-nginx

# 获取证书
sudo certbot --nginx -d openclaw.yourdomain.com

# 自动续期（默认已配置）
```

### Caddy 自动 HTTPS（推荐新手）

Caddy 会自动申请和续期 HTTPS 证书，配置更简单。

```bash
# 安装 Caddy
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

**配置 Caddyfile**

创建 `/etc/caddy/Caddyfile`：

```caddyfile
openclaw.yourdomain.com {
    reverse_proxy localhost:3000
    
    # 日志
    log {
        output file /var/log/caddy/openclaw.log
    }
    
    # 压缩
    encode gzip
}
```

启动：

```bash
sudo systemctl restart caddy
```

Caddy 会自动申请 Let's Encrypt 证书并配置 HTTPS。

---

## 3.6 数据库持久化与备份

### 数据持久化配置

确保 docker-compose.yml 中正确配置了数据卷：

```yaml
services:
  postgres:
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
  
  redis:
    volumes:
      - ./data/redis:/data
  
  openclaw:
    volumes:
      - ./data/uploads:/app/uploads
```

### 数据库备份脚本

创建 `backup.sh`：

```bash
#!/bin/bash

# 配置
BACKUP_DIR="/opt/backups/openclaw"
RETENTION_DAYS=30
DATE=$(date +%Y%m%d_%H%M%S)

# 创建备份目录
mkdir -p $BACKUP_DIR

# 备份 PostgreSQL
docker exec openclaw-postgres pg_dump -U openclaw openclaw | gzip > $BACKUP_DIR/postgres_$DATE.sql.gz

# 备份 Redis
docker exec openclaw-redis redis-cli BGSAVE
sleep 5
cp ./data/redis/dump.rdb $BACKUP_DIR/redis_$DATE.rdb

# 备份上传文件
tar -czf $BACKUP_DIR/uploads_$DATE.tar.gz ./data/uploads

# 清理旧备份
find $BACKUP_DIR -name "*.gz" -mtime +$RETENTION_DAYS -delete
find $BACKUP_DIR -name "*.rdb" -mtime +$RETENTION_DAYS -delete

echo "Backup completed: $DATE"
```

设置定时任务：

```bash
# 添加执行权限
chmod +x backup.sh

# 编辑 crontab
crontab -e

# 添加每天凌晨 3 点备份
0 3 * * * /opt/openclaw/backup.sh >> /var/log/openclaw-backup.log 2>&1
```

### 数据库恢复

```bash
# 1. 停止服务
docker compose down

# 2. 恢复 PostgreSQL
gunzip < /opt/backups/openclaw/postgres_20240101_030000.sql.gz | docker exec -i openclaw-postgres psql -U openclaw

# 3. 恢复 Redis
cp /opt/backups/openclaw/redis_20240101_030000.rdb ./data/redis/dump.rdb

# 4. 恢复上传文件
tar -xzf /opt/backups/openclaw/uploads_20240101_030000.tar.gz

# 5. 启动服务
docker compose up -d
```

---

## 3.7 日志管理

### 查看日志

```bash
# 查看所有服务日志
docker compose logs

# 查看特定服务日志
docker compose logs -f openclaw

# 查看最近 100 行
docker compose logs --tail=100 openclaw

# 查看特定时间段的日志
docker compose logs --since="2024-01-01T00:00:00" openclaw
```

### 日志切割配置

创建 `/etc/logrotate.d/openclaw`：

```
/opt/openclaw/logs/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 0644 root root
    sharedscripts
    postrotate
        /usr/bin/docker kill --signal="USR1" openclaw-core
    endscript
}
```

---

## 3.8 本章小结

本章介绍了 OpenClaw 在各种环境下的部署方案：

| 环境 | 适用场景 | 难度 |
|------|----------|------|
| 本地开发 | 开发测试 | ⭐ |
| 云服务器 | 生产环境 | ⭐⭐ |
| NAS/家庭服务器 | 个人长期使用 | ⭐⭐ |
| K8s 集群 | 企业高可用 | ⭐⭐⭐⭐ |

无论选择哪种部署方式，都要注意：

1. ✅ 配置数据持久化，防止容器重启数据丢失
2. ✅ 配置自动备份，定期验证备份可恢复性
3. ✅ 生产环境使用 HTTPS，配置反向代理
4. ✅ 配置日志切割，防止磁盘占满
5. ✅ 定期更新到最新版本，获取安全补丁

下一章，我们将深入讲解 OpenClaw 的核心概念，帮助你理解 Gateway、Agent、Channels 的工作原理。

---

## 参考命令速查

```bash
# 启动服务
docker compose up -d

# 停止服务
docker compose down

# 查看状态
docker compose ps

# 查看日志
docker compose logs -f

# 重启服务
docker compose restart

# 更新镜像
docker compose pull && docker compose up -d

# 进入容器
docker exec -it openclaw-core /bin/sh

# 数据库备份
docker exec openclaw-postgres pg_dump -U openclaw openclaw > backup.sql

# 数据库恢复
cat backup.sql | docker exec -i openclaw-postgres psql -U openclaw
```
