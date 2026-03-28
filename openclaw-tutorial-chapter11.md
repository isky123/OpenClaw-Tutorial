# 第11章：安全与合规

> 身份认证、权限管理、数据安全与审计日志

---

## 11.1 身份认证

### JWT 认证

```yaml
# 配置 JWT
auth:
  jwt:
    enabled: true
    secret: "${JWT_SECRET}"  # 至少 32 位随机字符串
    algorithm: HS256
    access_token_expires: 3600  # 1小时
    refresh_token_expires: 604800  # 7天
```

**Token 生成与验证**：

```python
import jwt
from datetime import datetime, timedelta
from typing import Dict, Optional

class JWTAuth:
    def __init__(self, secret: str, algorithm: str = "HS256"):
        self.secret = secret
        self.algorithm = algorithm
    
    def generate_tokens(self, user_id: str, **claims) -> Dict[str, str]:
        """生成访问令牌和刷新令牌"""
        now = datetime.utcnow()
        
        # 访问令牌
        access_payload = {
            "sub": user_id,
            "iat": now,
            "exp": now + timedelta(hours=1),
            "type": "access",
            **claims
        }
        access_token = jwt.encode(
            access_payload,
            self.secret,
            algorithm=self.algorithm
        )
        
        # 刷新令牌
        refresh_payload = {
            "sub": user_id,
            "iat": now,
            "exp": now + timedelta(days=7),
            "type": "refresh"
        }
        refresh_token = jwt.encode(
            refresh_payload,
            self.secret,
            algorithm=self.algorithm
        )
        
        return {
            "access_token": access_token,
            "refresh_token": refresh_token,
            "token_type": "Bearer",
            "expires_in": 3600
        }
    
    def verify_token(self, token: str) -> Optional[Dict]:
        """验证令牌"""
        try:
            payload = jwt.decode(
                token,
                self.secret,
                algorithms=[self.algorithm]
            )
            return payload
        except jwt.ExpiredSignatureError:
            return None
        except jwt.InvalidTokenError:
            return None
```

### OAuth2 / SSO 集成

```yaml
# OAuth2 配置
auth:
  oauth2:
    providers:
      - name: google
        client_id: "${GOOGLE_CLIENT_ID}"
        client_secret: "${GOOGLE_CLIENT_SECRET}"
        authorize_url: "https://accounts.google.com/o/oauth2/v2/auth"
        token_url: "https://oauth2.googleapis.com/token"
        userinfo_url: "https://openidconnect.googleapis.com/v1/userinfo"
        scopes:
          - openid
          - email
          - profile
      
      - name: github
        client_id: "${GITHUB_CLIENT_ID}"
        client_secret: "${GITHUB_CLIENT_SECRET}"
        authorize_url: "https://github.com/login/oauth/authorize"
        token_url: "https://github.com/login/oauth/access_token"
        userinfo_url: "https://api.github.com/user"
```

### Webhook 签名验证

```python
import hmac
import hashlib
import base64

class WebhookAuth:
    @staticmethod
    def verify_feishu_signature(
        encrypt_key: str,
        timestamp: str,
        nonce: str,
        body: str,
        signature: str
    ) -> bool:
        """验证飞书签名"""
        raw = f"{timestamp}{nonce}{encrypt_key}{body}"
        computed = hashlib.sha256(raw.encode()).hexdigest()
        return hmac.compare_digest(computed, signature)
    
    @staticmethod
    def verify_dingtalk_signature(
        secret: str,
        timestamp: str,
        signature: str
    ) -> bool:
        """验证钉钉签名"""
        raw = f"{timestamp}\n{secret}"
        mac = hmac.new(
            secret.encode(),
            raw.encode(),
            hashlib.sha256
        )
        computed = base64.b64encode(mac.digest()).decode()
        return hmac.compare_digest(computed, signature)
```

---

## 11.2 权限管理（RBAC）

### 角色定义

```yaml
# RBAC 配置
rbac:
  roles:
    - name: super_admin
      description: 超级管理员
      permissions:
        - "*:*"  # 所有权限
    
    - name: admin
      description: 管理员
      permissions:
        - "user:*"
        - "agent:*"
        - "channel:*"
        - "workflow:*"
        - "setting:read"
    
    - name: operator
      description: 运营人员
      permissions:
        - "agent:read"
        - "channel:read"
        - "workflow:read"
        - "workflow:execute"
        - "message:send"
    
    - name: viewer
      description: 只读用户
      permissions:
        - "agent:read"
        - "channel:read"
        - "message:read"
```

### 资源级权限控制

```python
from functools import wraps
from typing import List, Callable

class RBAC:
    def __init__(self):
        self.permissions_cache = {}
    
    def check_permission(self, user: dict, resource: str, action: str) -> bool:
        """检查用户是否有权限"""
        user_permissions = self._get_user_permissions(user["id"])
        required = f"{resource}:{action}"
        
        for permission in user_permissions:
            if self._match_permission(permission, required):
                return True
        
        return False
    
    def _match_permission(self, granted: str, required: str) -> bool:
        """匹配权限通配符"""
        # 完全匹配
        if granted == required:
            return True
        
        # 通配符匹配
        granted_parts = granted.split(":")
        required_parts = required.split(":")
        
        if len(granted_parts) != len(required_parts):
            return False
        
        for g, r in zip(granted_parts, required_parts):
            if g != "*" and g != r:
                return False
        
        return True
    
    def require_permission(self, resource: str, action: str):
        """权限装饰器"""
        def decorator(func: Callable):
            @wraps(func)
            async def wrapper(*args, **kwargs):
                user = kwargs.get("user") or args[0].user
                
                if not self.check_permission(user, resource, action):
                    raise PermissionError(
                        f"Permission denied: {resource}:{action}"
                    )
                
                return await func(*args, **kwargs)
            return wrapper
        return decorator

# 使用示例
rbac = RBAC()

class AgentController:
    @rbac.require_permission("agent", "create")
    async def create_agent(self, user, agent_data):
        # 只有有权限的用户才能执行
        pass
    
    @rbac.require_permission("agent", "delete")
    async def delete_agent(self, user, agent_id):
        # 检查用户是否有删除权限
        pass
```

### 数据权限隔离

```python
class DataIsolation:
    def __init__(self):
        self.tenant_field = "tenant_id"
    
    def apply_tenant_filter(self, query, user: dict):
        """应用租户隔离"""
        if user.get("role") == "super_admin":
            return query  # 超管不限制
        
        tenant_id = user.get("tenant_id")
        if tenant_id:
            query = query.filter(**{self.tenant_field: tenant_id})
        
        return query
    
    def check_ownership(self, resource: dict, user: dict) -> bool:
        """检查资源所有权"""
        if user.get("role") == "super_admin":
            return True
        
        # 检查是否是资源所有者
        if resource.get("owner_id") == user["id"]:
            return True
        
        # 检查是否在同一租户
        if resource.get("tenant_id") == user.get("tenant_id"):
            return True
        
        return False
```

---

## 11.3 数据安全

### 敏感信息加密

```python
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
import base64
import os

class Encryption:
    def __init__(self, master_key: str):
        """初始化加密器"""
        kdf = PBKDF2HMAC(
            algorithm=hashes.SHA256(),
            length=32,
            salt=os.urandom(16),
            iterations=100000,
        )
        key = base64.urlsafe_b64encode(kdf.derive(master_key.encode()))
        self.cipher = Fernet(key)
    
    def encrypt(self, plaintext: str) -> str:
        """加密数据"""
        return self.cipher.encrypt(plaintext.encode()).decode()
    
    def decrypt(self, ciphertext: str) -> str:
        """解密数据"""
        return self.cipher.decrypt(ciphertext.encode()).decode()

# 使用
encryption = Encryption(os.environ["ENCRYPTION_KEY"])

# 加密 API Key
encrypted_api_key = encryption.encrypt("sk-xxxxx")

# 解密使用
decrypted = encryption.decrypt(encrypted_api_key)
```

### 数据库加密

```yaml
# 数据库字段加密
database:
  encryption:
    enabled: true
    fields:
      - user.api_keys
      - channel.secrets
      - agent.webhook_tokens
```

```python
from sqlalchemy import TypeDecorator, String

class EncryptedString(TypeDecorator):
    """加密字符串字段"""
    impl = String
    
    def __init__(self, encryption, **kwargs):
        super().__init__(**kwargs)
        self.encryption = encryption
    
    def process_bind_param(self, value, dialect):
        if value is None:
            return None
        return self.encryption.encrypt(value)
    
    def process_result_value(self, value, dialect):
        if value is None:
            return None
        return self.encryption.decrypt(value)

# 模型定义
class Channel(Base):
    __tablename__ = "channels"
    
    id = Column(Integer, primary_key=True)
    name = Column(String(100))
    app_id = Column(String(100))
    app_secret = Column(EncryptedString(encryption))  # 自动加密
```

### 传输加密

```yaml
# TLS 配置
server:
  tls:
    enabled: true
    cert_file: /etc/ssl/certs/openclaw.crt
    key_file: /etc/ssl/private/openclaw.key
    min_version: "1.2"
    cipher_suites:
      - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
      - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
```

---

## 11.4 审计日志

### 审计事件定义

```python
from enum import Enum
from datetime import datetime
from typing import Optional, Dict, Any

class AuditEventType(Enum):
    # 认证事件
    LOGIN = "auth.login"
    LOGOUT = "auth.logout"
    TOKEN_REFRESH = "auth.token_refresh"
    PASSWORD_CHANGE = "auth.password_change"
    
    # 用户管理
    USER_CREATE = "user.create"
    USER_UPDATE = "user.update"
    USER_DELETE = "user.delete"
    
    # Agent 管理
    AGENT_CREATE = "agent.create"
    AGENT_UPDATE = "agent.update"
    AGENT_DELETE = "agent.delete"
    
    # 渠道管理
    CHANNEL_CREATE = "channel.create"
    CHANNEL_UPDATE = "channel.update"
    CHANNEL_DELETE = "channel.delete"
    
    # 消息事件
    MESSAGE_SEND = "message.send"
    MESSAGE_RECEIVE = "message.receive"
    
    # 工作流
    WORKFLOW_EXECUTE = "workflow.execute"
    WORKFLOW_FAIL = "workflow.fail"

class AuditLogger:
    def __init__(self, storage):
        self.storage = storage
    
    async def log(
        self,
        event_type: AuditEventType,
        user_id: str,
        resource_type: str,
        resource_id: str,
        action: str,
        details: Optional[Dict[str, Any]] = None,
        ip_address: Optional[str] = None,
        user_agent: Optional[str] = None
    ):
        """记录审计日志"""
        event = {
            "timestamp": datetime.utcnow().isoformat(),
            "event_type": event_type.value,
            "user_id": user_id,
            "resource_type": resource_type,
            "resource_id": resource_id,
            "action": action,
            "details": details or {},
            "ip_address": ip_address,
            "user_agent": user_agent,
            "request_id": self._get_request_id()
        }
        
        # 异步写入
        await self.storage.save(event)
```

### 审计日志存储

```yaml
# 审计日志配置
audit:
  enabled: true
  storage: elasticsearch  # elasticsearch, database, file
  retention_days: 365
  
  elasticsearch:
    hosts: ["http://localhost:9200"]
    index: "openclaw-audit"
  
  sensitive_fields:
    - password
    - token
    - secret
    - api_key
```

### 审计报表

```python
class AuditReport:
    def __init__(self, audit_storage):
        self.audit_storage = audit_storage
    
    async def generate_user_activity_report(
        self,
        user_id: str,
        start_date: datetime,
        end_date: datetime
    ) -> Dict:
        """生成用户活动报告"""
        events = await self.storage.query(
            user_id=user_id,
            start_date=start_date,
            end_date=end_date
        )
        
        # 统计
        event_counts = {}
        for event in events:
            event_type = event["event_type"]
            event_counts[event_type] = event_counts.get(event_type, 0) + 1
        
        return {
            "user_id": user_id,
            "period": {
                "start": start_date.isoformat(),
                "end": end_date.isoformat()
            },
            "total_events": len(events),
            "event_breakdown": event_counts,
            "events": events
        }
    
    async def generate_security_report(
        self,
        start_date: datetime,
        end_date: datetime
    ) -> Dict:
        """生成安全审计报告"""
        # 登录失败统计
        failed_logins = await self.storage.query(
            event_type=AuditEventType.LOGIN,
            status="failed",
            start_date=start_date,
            end_date=end_date
        )
        
        # 权限变更
        permission_changes = await self.storage.query(
            event_type=[
                AuditEventType.USER_CREATE,
                AuditEventType.USER_UPDATE,
                AuditEventType.USER_DELETE
            ],
            start_date=start_date,
            end_date=end_date
        )
        
        return {
            "period": {
                "start": start_date.isoformat(),
                "end": end_date.isoformat()
            },
            "failed_login_attempts": len(failed_logins),
            "permission_changes": len(permission_changes),
            "suspicious_activities": self._detect_anomalies(failed_logins)
        }
```

---

## 11.5 国内合规要点

### 数据本地化

```yaml
# 数据存储配置
storage:
  # 数据必须存储在境内
  region: cn-beijing
  
  # 备份也必须在境内
  backup:
    location: cn-shanghai
    cross_border_sync: false
  
  # 日志存储
  audit_logs:
    retention: 180d  # 至少保留6个月
    location: cn-beijing
```

### 隐私政策模板

```markdown
# 隐私政策

## 信息收集
我们收集以下类型的信息：

### 账号信息
- 用户名、邮箱、手机号
- 企业信息（企业用户）

### 使用数据
- 对话记录（用于服务质量改进）
- 操作日志（用于安全保障）
- 设备信息（用于故障排查）

## 信息使用
我们使用收集的信息用于：
- 提供 AI 对话服务
- 改进产品功能和体验
- 保障账号和服务安全
- 遵守法律法规要求

## 信息存储
- 数据存储地点：中国大陆境内
- 对话记录保留：90天
- 日志保留：180天

## 用户权利
您拥有以下权利：
- 查看和导出个人数据
- 更正不准确的个人信息
- ��除账号和相关数据
- 撤回授权同意

## 联系我们
如有隐私相关问题，请联系：privacy@your-company.com
```

### 安全评估

```yaml
# 等保2.0 合规检查清单
compliance:
  level: 3  # 等保三级
  
  requirements:
    # 物理安全
    - id: 1.1
      item: 服务器位于合规机房
      status: completed
    
    # 网络安全
    - id: 2.1
      item: 部署防火墙
      status: completed
    - id: 2.2
      item: 启用 TLS 1.2+
      status: completed
    
    # 主机安全
    - id: 3.1
      item: 定期安全补丁更新
      status: ongoing
    - id: 3.2
      item: 启用防病毒软件
      status: completed
    
    # 应用安全
    - id: 4.1
      item: 代码安全审计
      status: ongoing
    - id: 4.2
      item: 渗透测试
      status: pending
    
    # 数据安全
    - id: 5.1
      item: 敏感数据加密
      status: completed
    - id: 5.2
      item: 数据备份
      status: completed
    
    # 安全管理
    - id: 6.1
      item: 安全管理制度
      status: completed
    - id: 6.2
      item: 应急响应预案
      status: completed
```

---

## 11.6 本章小结

本章讲解了安全与合规的核心要点：

1. **身份认证**：JWT、OAuth2、Webhook 签名验证
2. **权限管理**：RBAC 模型、资源级权限、数据隔离
3. **数据安全**：敏感信息加密、数据库加密、传输加密
4. **审计日志**：事件定义、存储、报表生成
5. **国内合规**：数据本地化、隐私政策、等保要求

**安全 checklist**：
- [ ] 使用强密码和密钥
- [ ] 启用 HTTPS
- [ ] 配置 RBAC 权限
- [ ] 加密敏感数据
- [ ] 启用审计日志
- [ ] 定期安全更新
- [ ] 制定隐私政策
- [ ] 完成合规评估

---

## 参考配置

```yaml
# security.yaml - 安全配置汇总
security:
  auth:
    jwt:
      secret: "${JWT_SECRET}"
      expires: 3600
    
    oauth2:
      enabled: true
      providers:
        - google
        - github
  
  rbac:
    enabled: true
    default_role: viewer
  
  encryption:
    master_key: "${ENCRYPTION_KEY}"
    fields:
      - api_keys
      - secrets
      - tokens
  
  audit:
    enabled: true
    retention: 365d
    storage: elasticsearch
  
  compliance:
    data_localization: true
    region: cn
```
