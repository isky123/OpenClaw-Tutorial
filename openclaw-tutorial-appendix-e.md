# 附录E：国内部署特别指南

> 针对国内网络环境和合规要求的专项指南

---

## E.1 国内网络环境特有问题

### 镜像拉取加速

由于网络原因，Docker Hub 在国内访问不稳定，建议配置镜像加速：

**Docker Desktop（Mac/Windows）**

1. 打开 Docker Desktop
2. Settings → Docker Engine
3. 添加 registry-mirrors：

```json
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com",
    "https://mirror.baidubce.com",
    "https://ccr.ccs.tencentyun.com"
  ]
}
```

**Linux 命令行配置**

```bash
# 编辑配置文件
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json > /dev/null <<EOF
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com",
    "https://mirror.baidubce.com"
  ]
}
EOF

# 重启 Docker
sudo systemctl restart docker
```

### GitHub 访问加速

如果克隆 GitHub 仓库缓慢，可使用镜像：

```bash
# 使用 ghproxy 代理
git clone https://ghproxy.com/https://github.com/openclaw/openclaw.git

# 或使用 fastgit
git clone https://hub.fastgit.xyz/openclaw/openclaw.git
```

### pip 镜像加速

```bash
# 临时使用
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple package-name

# 永久配置
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple

# 其他镜像源
# 阿里云：https://mirrors.aliyun.com/pypi/simple/
# 腾讯云：https://mirrors.cloud.tencent.com/pypi/simple/
# 豆瓣：http://pypi.douban.com/simple/
```

---

## E.2 国产 LLM 接入汇总

### DeepSeek（深度求索）

**特点**：
- 国内访问速度快
- 价格便宜，性价比高
- 代码能力强

**接入配置**：

```yaml
models:
  - name: "deepseek-chat"
    provider: deepseek
    config:
      api_key: "${DEEPSEEK_API_KEY}"
      base_url: "https://api.deepseek.com/v1"
      model: "deepseek-chat"
```

**获取 API Key**：
1. 访问 https://platform.deepseek.com/
2. 注册账号（支持国内手机号）
3. 进入「API Keys」创建 Key

**价格参考**：
- 输入：1元/百万 tokens
- 输出：2元/百万 tokens
- 新用户赠送 5000万 tokens

### 通义千问（阿里云）

**特点**：
- 阿里出品，中文能力强
- 与阿里云生态集成好

**接入配置**：

```yaml
models:
  - name: "qwen-max"
    provider: dashscope
    config:
      api_key: "${DASHSCOPE_API_KEY}"
      base_url: "https://dashscope.aliyuncs.com/api/v1"
      model: "qwen-max"
```

**获取 API Key**：
1. 访问 https://bailian.console.aliyun.com/
2. 登录阿里云账号
3. 进入「API Key 管理」创建 Key

### 文心一言（百度）

**接入配置**：

```yaml
models:
  - name: "ernie-bot-4"
    provider: baidu
    config:
      api_key: "${BAIDU_API_KEY}"
      secret_key: "${BAIDU_SECRET_KEY}"
      model: "ernie-bot-4"
```

### 智谱 AI（GLM）

**接入配置**：

```yaml
models:
  - name: "glm-4"
    provider: zhipu
    config:
      api_key: "${ZHIPU_API_KEY}"
      base_url: "https://open.bigmodel.cn/api/paas/v4"
      model: "glm-4"
```

### 讯飞星火

**接入配置**：

```yaml
models:
  - name: "spark-pro"
    provider: xfyun
    config:
      app_id: "${XFYUN_APP_ID}"
      api_key: "${XFYUN_API_KEY}"
      api_secret: "${XFYUN_API_SECRET}"
      model: "spark-pro"
```

### 月之暗面（Kimi）

**特点**：
- 支持超长上下文（200万字）
- 适合长文档处理

**接入配置**：

```yaml
models:
  - name: "kimi"
    provider: moonshot
    config:
      api_key: "${MOONSHOT_API_KEY}"
      base_url: "https://api.moonshot.cn/v1"
      model: "moonshot-v1-8k"
```

---

## E.3 国内渠道接入汇总

### 飞书详细配置

**创建应用**：

1. 访问 https://open.feishu.cn/
2. 点击「创建企业自建应用"
3. 填写应用名称和描述

**开启能力**：

1. 「机器人」→ 开启机器人能力
2. 「权限管理」→ 添加以下权限：
   - `im:chat:readonly`
   - `im:message:send`
   - `im:message:read`
   - `contact:user.base:readonly`

**事件订阅配置**：

1. 「事件订阅」→ 开启加密
2. 复制 Encrypt Key 和 Verification Token
3. 配置请求地址（OpenClaw 生成的 Webhook URL）
4. 添加订阅事件：`im.message.receive_v1`

**发布应用**：

1. 「版本管理与发布」→ 创建版本
2. 填写版本号和更新说明
3. 申请发布 → 管理员审批

### 钉钉详细配置

**创建机器人**：

1. 进入群聊 → 群设置 → 智能群助手
2. 添加机器人 → 自定义
3. 设置安全模式为「加签」

**获取凭证**：

- Webhook URL：`https://oapi.dingtalk.com/robot/send?access_token=xxx`
- 加签密钥：`SECxxx`

**OpenClaw 配置**：

```yaml
channels:
  - name: "钉钉群"
    type: dingtalk
    config:
      access_token: "xxx"
      secret: "SECxxx"
```

### 企业微信详细配置

**创建应用**：

1. 访问 https://work.weixin.qq.com/
2. 应用管理 → 创建应用
3. 上传 logo，填写应用名称

**获取凭证**：

- 企业 ID：「我的企业」页面底部
- AgentId：应用详情页
- Secret：应用详情页 → 查看

**接收消息配置**：

1. 应用详情 → 「接收消息」→ 设置
2. 填写 URL、Token、EncodingAESKey
3. 在 OpenClaw 中配置相同的 Token 和 EncodingAESKey

**配置可信域名**：

1. 「网页授权及JS-SDK」→ 设置可信域名
2. 下载验证文件并上传到域名根目录
3. 填写域名并验证

### 微信公众号详细配置

**服务器配置**：

1. 微信公众平台 → 开发 → 基本配置
2. 启用服务器配置
3. 填写：
   - URL：`https://your-domain.com/api/webhooks/wechat-mp/{channel_id}`
   - Token：自定义（与 OpenClaw 配置一致）
   - EncodingAESKey：随机生成
   - 消息加解密方式：推荐「兼容模式」

**IP 白名单**：

1. 「基本配置」→ IP 白名单
2. 添加服务器公网 IP

**网页授权**（如需）：

1. 「设置与开发」→ 公众号设置 → 功能设置
2. 设置网页授权域名

---

## E.4 备案与合规要点

### ICP 备案

**需要备案的情况**：
- 使用国内云服务器（阿里云、腾讯云等）
- 域名解析到国内服务器
- 提供公开访问的 Web 服务

**备案流程**：

1. 购买域名（建议在国内注册商购买）
2. 购买国内云服务器（需 3 个月以上）
3. 在云厂商备案系统提交申请
4. 等待管局审核（约 7-20 个工作日）

**免备案方案**：

- 使用海外服务器（香港、新加坡、美国等）
- 使用 Cloudflare Tunnel 等内网穿透
- 仅供个人/团队内部使用，不对外公开

### 数据安全合规

**数据本地化**：

```yaml
# 配置数据存储在国内
storage:
  # 使用国内云存储
  oss:
    provider: aliyun
    region: cn-hangzhou
  
  # 使用国内数据库
  database:
    host: rm-xxx.mysql.rds.aliyuncs.com
```

**敏感数据处理**：

```python
# 数据脱敏示例
def mask_sensitive_data(data: dict) -> dict:
    """敏感数据脱敏"""
    masked = data.copy()
    
    # 手机号脱敏
    if "phone" in masked:
        phone = masked["phone"]
        masked["phone"] = phone[:3] + "****" + phone[-4:]
    
    # 邮箱脱敏
    if "email" in masked:
        email = masked["email"]
        local, domain = email.split("@")
        masked["email"] = local[:2] + "***@" + domain
    
    return masked
```

### 隐私政策与用户协议

**建议包含的内容**：

```markdown
# 隐私政策

## 信息收集
我们收集的信息包括：
- 用户账号信息（昵称、头像）
- 对话记录（用于改进服务质量）
- 使用日志（用于故障排查）

## 信息使用
我们使用信息用于：
- 提供 AI 对话服务
- 改进产品功能
- 保障服务安全

## 信息保护
- 数据存储在国内服务器
- 采用加密传输和存储
- 定期删除历史数据

## 用户权利
- 查看和导出个人数据
- 删除账号和相关数据
- 撤回授权同意

## 联系我们
如有问题请联系：privacy@your-company.com
```

---

## E.5 国内云服务器推荐

### 轻量应用服务器（个人/小团队）

| 厂商 | 配置 | 价格 | 特点 |
|------|------|------|------|
| 阿里云 | 2核2G 3M | 99元/年 | 新用户优惠 |
| 腾讯云 | 2核2G 4M | 99元/年 | 带宽较高 |
| 华为云 | 2核2G 2M | 99元/年 | 稳定可靠 |

### 标准云服务器（企业）

| 厂商 | 配置 | 价格 | 特点 |
|------|------|------|------|
| 阿里云 ECS | 4核8G 5M | 约 300元/月 | 生态丰富 |
| 腾讯云 CVM | 4核8G 5M | 约 280元/月 | 性价比高 |
| 华为云 ECS | 4核8G 5M | 约 320元/月 | 企业级服务 |

### 海外服务器（免备案）

| 厂商 | 位置 | 价格 | 特点 |
|------|------|------|------|
| AWS Lightsail | 新加坡 | $5/月 | 国内访问快 |
| Vultr | 东京 | $6/月 | 按小时计费 |
| DigitalOcean | 新加坡 | $6/月 | 简单易用 |

---

## E.6 常见问题

### Q1: 国内访问 OpenAI API 有什么方案？

**方案一：使用代理**

```yaml
models:
  - name: "gpt-4"
    provider: openai
    config:
      api_key: "${OPENAI_API_KEY}"
      base_url: "https://api.openai.com/v1"
      http_proxy: "http://proxy-server:port"
```

**方案二：使用中转服务**

```yaml
models:
  - name: "gpt-4"
    provider: openai
    config:
      api_key: "${OPENAI_API_KEY}"
      base_url: "https://api.xxx.com/v1"  # 第三方中转地址
```

**方案三：使用国产替代模型**

推荐使用 DeepSeek、通义千问等国产模型，无需代理。

### Q2: 飞书机器人发送消息失败？

**检查清单**：

1. 应用是否已发布并通过审批？
2. 是否开通了 `im:message:send` 权限？
3. Webhook URL 是否配置正确？
4. 服务器是否能访问飞书 API？

### Q3: 微信公众号 Token 验证失败？

**常见原因**：

1. URL 不可访问（检查防火墙/安全组）
2. Token 不匹配（确保 OpenClaw 和公众号配置一致）
3. 未使用 80/443 端口（微信要求）
4. 响应时间超过 5 秒（优化服务器性能）

### Q4: 如何降低 LLM 调用成本？

**优化策略**：

1. 使用国产模型（DeepSeek 比 GPT-4 便宜 20 倍）
2. 启用缓存（相似请求直接返回缓存结果）
3. 压缩上下文（减少 Token 消耗）
4. 设置 Token 上限（防止意外消耗）

---

## E.7 参考资源

### 官方文档

- 飞书开放平台：https://open.feishu.cn/
- 钉钉开放平台：https://open.dingtalk.com/
- 企业微信开发者中心：https://developer.work.weixin.qq.com/
- 微信开放平台：https://open.weixin.qq.com/

### 国产大模型文档

- DeepSeek：https://platform.deepseek.com/
- 通义千问：https://bailian.console.aliyun.com/
- 文心一言：https://cloud.baidu.com/doc/WENXINWORKSHOP/index.html
- 智谱 AI：https://open.bigmodel.cn/

### 社区资源

- OpenClaw GitHub：https://github.com/openclaw/openclaw
- OpenClaw 中文社区：https://forum.openclaw.cn/

---

*本附录持续更新中，如有问题欢迎反馈*

