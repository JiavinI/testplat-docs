# 通用测试平台鉴权架构设计

> **2026-10-15 当前方向**：登录流程放 Git，测试账号与凭据通过 Web 按项目 + 环境 + 账号名维护。平台用户登录仅识别操作人，现阶段无超管分级 / 复杂权限。详见 [需求基线](../../notes/phase4-roadmap.md) 与 [4.1 设计稿](../../notes/phase4.1-design-draft.md)。4.1 是待实现方案，不能当作当前能力。
>
> **当前实现基线**：Phase 3.3a–3.3d 已落地 Engine v0.5.0；参考 notes 中的登录与缓存设计及 Control docs/phase3_3/control-auth-backend.md。4.0 的 config/auth/<env>.yaml + 服务器秘密变量是历史配置来源，4.1 将由 Control 组合 Git 流程与数据库凭据后构造 AuthTable；Engine 继续负责登录、缓存与注入。

## 本轮必须区分

- 测试账号名选择“谁操作”，Git 登录流程描述“怎么登录”，项目环境决定服务地址和具体凭据；同一流程可供多个账号使用。
- 当前 API 的 auth 可以选择身份；当前 case schema 0.2 尚不支持步骤 account。4.1a 先接配置来源，4.1b 再增量扩展账号选择，不使用旧版 as 坐标偷偷兼容。
- 缓存按项目、环境、账号和材料指纹隔离；新密码不能命中旧材料。匿名、Cookie 会话、401 重登等继续遵循现有 Engine 协议，不扩大自动重放范围。
- 被测系统凭据需可逆加密以用于登录；平台用户登录密码只存哈希。两类账号不共用凭据表，平台用户不会成为 Engine identity。
- 日志始终遮挡秘密；简化权限不等于把凭据写进日志或资产。

## 历史草案范围

以下三层 profile / route、secrets 文件、AuthManager 等代码是早期设计参考；实际 DTO 和行为以 Engine interface.py / 兼容文档及当前批次契约为准，不应直接照抄落地。

## 设计目标

### 核心诉求
1. **通用性**: 覆盖常见鉴权方式 (HTTP Basic/Bearer Token/Cookie/OAuth2)
2. **可扩展性**: 新增鉴权策略时,无需修改引擎核心代码
3. **多租户隔离**: 同一平台支持多个项目,各自使用不同的鉴权方式
4. **配置驱动**: 鉴权策略、登录地址、凭据存储均由配置文件管理

### 设计原则
> 基于 testplat-new 的成功经验提炼

- ✅ **分离身份与策略**: 用例声明"谁在操作"(身份),配置文件决定"怎么登录"(策略)
- ✅ **插件化策略**: 每种登录方式是独立插件,实现统一接口
- ✅ **三维路由**: `(身份, 通道, 目标)` → 选择合适的 profile
- ✅ **缓存与自愈**: Token 缓存 + 401 自动重登

---

## 1. 核心概念模型

### 1.1 三层抽象

```
┌─────────────────────────────────────────────────────────┐
│  用例层 (Case/API)                                       │
│  - 只声明身份: as: "admin"                                │
│  - 不关心 token/cookie/OAuth                             │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────┐
│  配置层 (Config)                                         │
│  - identities: 定义身份映射到凭据路径                      │
│  - profiles: 定义登录策略 (http_form/oauth2/bearer)       │
│  - routes: 三维路由 (身份, 通道, 目标) → profile          │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────┐
│  策略层 (Strategy Plugin)                                │
│  - 每种登录方式是独立插件                                  │
│  - 实现 LoginStrategy 接口                                │
│  - 输入: 账号凭据 + 策略参数                               │
│  - 输出: AuthResult (token/cookies)                       │
└─────────────────────────────────────────────────────────┘
```

### 1.2 关键实体

#### Identity (身份)
```yaml
# 定义: 谁在操作
identities:
  admin:
    credentials: accounts.admin.{variant}  # 凭据路径模板
    scope: shared  # shared | project
  buyer_a:
    credentials: accounts.{project}.buyer.{variant}
    scope: project
```

#### Profile (登录配置)
```yaml
# 定义: 怎么登录
profiles:
  api:
    admin_bearer:
      strategy: bearer_token  # 策略名称
      params:
        token_field: access_token
        header_name: Authorization
        header_prefix: "Bearer "
    
    admin_form_login:
      strategy: http_form
      params:
        url: /api/login
        method: POST
        success_path: $.code
        success_value: 0
        token_path: $.data.token
```

#### Route (路由规则)
```yaml
# 定义: 什么场景用什么 profile
routes:
  api:
    - identity: admin
      target: "api-gateway"  # 通配或具体 host
      profile: admin_bearer
    
    - identity: buyer_*
      target: "shop-service"
      profile: buyer_form_login
```

---

## 2. 策略插件接口

### 2.1 统一接口定义

```python
# testplat_engine/auth/strategy.py

from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Any, Dict, Optional, List
from http.cookiejar import CookieJar

@dataclass
class AuthContext:
    """鉴权上下文 (引擎传入)"""
    identity: str  # 身份标识 (如 "admin")
    channel: str   # 通道 (api | web)
    target: str    # 目标 (host 或 web_host)
    credentials: Dict[str, Any]  # 凭据 (从 secrets 解析)
    params: Dict[str, Any]  # 策略参数 (从 profile.params)
    base_url: str  # 目标系统的基础 URL

@dataclass
class AuthResult:
    """鉴权结果"""
    success: bool
    # API 通道返回
    token: Optional[str] = None
    headers: Optional[Dict[str, str]] = None
    # Web 通道返回
    cookies: Optional[CookieJar] = None
    storage_state: Optional[Dict[str, Any]] = None  # Playwright 存储状态
    # 共用
    expires_at: Optional[float] = None  # Unix 时间戳
    error: Optional[str] = None

class LoginStrategy(ABC):
    """登录策略基类"""
    
    @property
    @abstractmethod
    def name(self) -> str:
        """策略名称 (用于配置文件引用)"""
        pass
    
    @abstractmethod
    def login(self, ctx: AuthContext, http_client: Any) -> AuthResult:
        """执行登录
        
        Args:
            ctx: 鉴权上下文
            http_client: HTTP 客户端 (httpx.Client)
        
        Returns:
            AuthResult: 鉴权结果
        """
        pass
    
    def validate_params(self, params: Dict[str, Any]) -> None:
        """验证策略参数 (可选,启动时校验配置)
        
        Raises:
            ValueError: 参数不合法
        """
        pass
    
    def validate_credentials(self, credentials: Dict[str, Any]) -> None:
        """验证凭据字段 (可选)
        
        Raises:
            ValueError: 凭据字段缺失或不合法
        """
        pass
```

### 2.2 内置策略实现

#### 策略 1: Bearer Token (最简单)
```python
class BearerTokenStrategy(LoginStrategy):
    """直接使用预置 token,不需要登录接口"""
    
    @property
    def name(self) -> str:
        return "bearer_token"
    
    def validate_credentials(self, credentials: Dict[str, Any]) -> None:
        if "token" not in credentials:
            raise ValueError("bearer_token 策略需要 credentials.token 字段")
    
    def login(self, ctx: AuthContext, http_client: Any) -> AuthResult:
        token = ctx.credentials["token"]
        prefix = ctx.params.get("header_prefix", "Bearer ")
        header_name = ctx.params.get("header_name", "Authorization")
        
        return AuthResult(
            success=True,
            token=token,
            headers={header_name: f"{prefix}{token}"},
            expires_at=None,  # 永不过期 (或从配置读取)
        )
```

#### 策略 2: HTTP Form 登录
```python
class HttpFormStrategy(LoginStrategy):
    """表单登录 (用户名密码 POST)"""
    
    @property
    def name(self) -> str:
        return "http_form"
    
    def validate_params(self, params: Dict[str, Any]) -> None:
        required = ["url", "success_path", "token_path"]
        for key in required:
            if key not in params:
                raise ValueError(f"http_form 策略缺少参数: {key}")
    
    def validate_credentials(self, credentials: Dict[str, Any]) -> None:
        if "username" not in credentials or "password" not in credentials:
            raise ValueError("http_form 策略需要 credentials.username 和 password")
    
    def login(self, ctx: AuthContext, http_client: Any) -> AuthResult:
        url = ctx.base_url + ctx.params["url"]
        method = ctx.params.get("method", "POST").upper()
        
        # 构建请求体
        body = {
            ctx.params.get("username_field", "username"): ctx.credentials["username"],
            ctx.params.get("password_field", "password"): ctx.credentials["password"],
        }
        
        # 发送登录请求
        response = http_client.request(method, url, json=body)
        
        # 判断成功
        from jsonpath_ng import parse
        success_expr = parse(ctx.params["success_path"])
        matches = success_expr.find(response.json())
        if not matches or str(matches[0].value) != str(ctx.params.get("success_value", "0")):
            return AuthResult(success=False, error=f"登录失败: {response.text}")
        
        # 提取 token
        token_expr = parse(ctx.params["token_path"])
        token_matches = token_expr.find(response.json())
        if not token_matches:
            return AuthResult(success=False, error="响应中未找到 token")
        
        token = str(token_matches[0].value)
        header_name = ctx.params.get("header_name", "Authorization")
        header_prefix = ctx.params.get("header_prefix", "Bearer ")
        
        return AuthResult(
            success=True,
            token=token,
            headers={header_name: f"{header_prefix}{token}"},
            expires_at=None,  # TODO: 从响应提取过期时间
        )
```

#### 策略 3: Cookie Session
```python
class CookieSessionStrategy(LoginStrategy):
    """基于 Cookie 的会话登录"""
    
    @property
    def name(self) -> str:
        return "cookie_session"
    
    def login(self, ctx: AuthContext, http_client: Any) -> AuthResult:
        url = ctx.base_url + ctx.params["url"]
        
        body = {
            ctx.params.get("username_field", "username"): ctx.credentials["username"],
            ctx.params.get("password_field", "password"): ctx.credentials["password"],
        }
        
        # 发送登录请求 (httpx 自动管理 cookies)
        response = http_client.request("POST", url, json=body)
        
        # 检查成功
        success_expr = parse(ctx.params["success_path"])
        matches = success_expr.find(response.json())
        if not matches or str(matches[0].value) != str(ctx.params.get("success_value", "0")):
            return AuthResult(success=False, error="登录失败")
        
        # 提取 cookies (httpx.Client 已存储)
        cookies = http_client.cookies
        
        return AuthResult(
            success=True,
            cookies=cookies,
            expires_at=None,
        )
```

#### 策略 4: OAuth2 (扩展示例)
```python
class OAuth2Strategy(LoginStrategy):
    """OAuth2 授权码模式"""
    
    @property
    def name(self) -> str:
        return "oauth2"
    
    def validate_params(self, params: Dict[str, Any]) -> None:
        required = ["auth_url", "token_url", "client_id", "client_secret"]
        for key in required:
            if key not in params:
                raise ValueError(f"oauth2 策略缺少参数: {key}")
    
    def login(self, ctx: AuthContext, http_client: Any) -> AuthResult:
        # 1. 获取授权码 (简化示例,实际需要浏览器交互)
        auth_url = ctx.params["auth_url"]
        # ... 省略授权码获取逻辑
        
        # 2. 用授权码换取 access_token
        token_url = ctx.params["token_url"]
        token_response = http_client.post(token_url, data={
            "grant_type": "authorization_code",
            "code": "...",  # 授权码
            "client_id": ctx.params["client_id"],
            "client_secret": ctx.params["client_secret"],
        })
        
        token_data = token_response.json()
        access_token = token_data["access_token"]
        expires_in = token_data.get("expires_in", 3600)
        
        return AuthResult(
            success=True,
            token=access_token,
            headers={"Authorization": f"Bearer {access_token}"},
            expires_at=time.time() + expires_in,
        )
```

---

## 3. 配置文件结构

### 3.1 全局配置 (跨项目共享)

```yaml
# config/auth/_global.yaml

# 定义身份契约
identities:
  admin:
    credentials: accounts.admin.{variant}
    scope: shared  # 跨项目共享
    required_fields: [username, password]
  
  api_client:
    credentials: accounts.api_client.{variant}
    scope: shared
    required_fields: [token]

# 定义通用策略模板
strategy_templates:
  default_bearer:
    strategy: bearer_token
    params:
      header_name: Authorization
      header_prefix: "Bearer "
  
  default_form_login:
    strategy: http_form
    params:
      method: POST
      username_field: username
      password_field: password
      success_path: $.code
      success_value: 0
```

### 3.2 项目配置 (覆盖 + 扩展)

```yaml
# config/auth/my-api-tests.yaml

project: my-api-tests

# 定义项目专属身份
identities:
  buyer:
    credentials: accounts.{project}.buyer.{variant}
    scope: project
    required_fields: [username, password]

# 定义登录 profile
profiles:
  api:
    admin_auth:
      extends: default_bearer  # 继承全局模板
    
    buyer_auth:
      strategy: http_form
      params:
        url: /api/user/login
        success_path: $.code
        success_value: 0
        token_path: $.data.token
        header_name: X-Auth-Token
  
  web:
    admin_web_auth:
      strategy: cookie_session
      params:
        url: /admin/login
        success_path: $.success
        success_value: true

# 定义路由规则
routes:
  api:
    - identity: admin
      target: "*"  # 所有 API 目标
      profile: admin_auth
    
    - identity: buyer
      target: "user-service"
      profile: buyer_auth
  
  web:
    - identity: admin
      target: "admin-portal"
      profile: admin_web_auth
```

### 3.3 凭据配置（历史 secrets 文件方案；4.1 改为 Web / 数据库）

```yaml
# config/secrets/my-api-tests.secrets.yaml

accounts:
  admin:
    default:
      username: test_admin
      password: secret123
  
  api_client:
    default:
      token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
  
  my-api-tests:  # 项目专属账号
    buyer:
      primary:
        username: buyer001
        password: buyer_pass
      secondary:
        username: buyer002
        password: buyer_pass2
```

---

## 4. 引擎集成方式

### 4.1 初始化鉴权管理器

```python
# testplat_engine/auth/manager.py

from typing import Dict
from .strategy import LoginStrategy, AuthContext, AuthResult

class AuthManager:
    """鉴权管理器"""
    
    def __init__(self, config: Dict[str, Any], secrets: Dict[str, Any]):
        self.config = config
        self.secrets = secrets
        self.strategies: Dict[str, LoginStrategy] = {}
        self._register_builtin_strategies()
    
    def _register_builtin_strategies(self):
        """注册内置策略"""
        from .strategies import (
            BearerTokenStrategy,
            HttpFormStrategy,
            CookieSessionStrategy,
        )
        for strategy_cls in [BearerTokenStrategy, HttpFormStrategy, CookieSessionStrategy]:
            strategy = strategy_cls()
            self.strategies[strategy.name] = strategy
    
    def register_strategy(self, strategy: LoginStrategy):
        """注册自定义策略"""
        self.strategies[strategy.name] = strategy
    
    def authenticate(
        self,
        identity: str,
        channel: str,
        target: str,
        http_client: Any,
    ) -> AuthResult:
        """执行鉴权
        
        流程:
        1. 路由: (identity, channel, target) → profile
        2. 解析: profile → strategy + params
        3. 凭据: identity → credentials
        4. 执行: strategy.login(ctx)
        """
        # 1. 路由选择 profile
        profile = self._route_profile(identity, channel, target)
        if profile is None:
            return AuthResult(success=False, error=f"未找到匹配的 profile: {identity}/{channel}/{target}")
        
        # 2. 获取策略
        strategy_name = profile["strategy"]
        if strategy_name not in self.strategies:
            return AuthResult(success=False, error=f"未知策略: {strategy_name}")
        strategy = self.strategies[strategy_name]
        
        # 3. 解析凭据
        credentials = self._resolve_credentials(identity)
        if not credentials:
            return AuthResult(success=False, error=f"未找到身份凭据: {identity}")
        
        # 4. 构建上下文
        ctx = AuthContext(
            identity=identity,
            channel=channel,
            target=target,
            credentials=credentials,
            params=profile.get("params", {}),
            base_url=self._get_base_url(channel, target),
        )
        
        # 5. 执行登录
        return strategy.login(ctx, http_client)
    
    def _route_profile(self, identity: str, channel: str, target: str) -> Optional[Dict]:
        """路由规则匹配"""
        routes = self.config.get("routes", {}).get(channel, [])
        for route in routes:
            if self._match_identity(route["identity"], identity) and \
               self._match_target(route["target"], target):
                profile_id = route["profile"]
                return self.config["profiles"][channel][profile_id]
        return None
    
    def _match_identity(self, pattern: str, identity: str) -> bool:
        """身份匹配 (支持通配符)"""
        if pattern == "*":
            return True
        if pattern.endswith("*"):
            return identity.startswith(pattern[:-1])
        return pattern == identity
    
    def _match_target(self, pattern: str, target: str) -> bool:
        """目标匹配"""
        if pattern == "*":
            return True
        return pattern == target
    
    def _resolve_credentials(self, identity: str) -> Optional[Dict[str, Any]]:
        """解析身份凭据"""
        identity_config = self.config["identities"].get(identity)
        if not identity_config:
            return None
        
        # 解析凭据路径模板 (如 accounts.admin.{variant})
        path_template = identity_config["credentials"]
        # TODO: 支持 {project} 和 {variant} 变量替换
        # 简化示例: 直接取 default
        path = path_template.replace("{variant}", "default")
        
        # 从 secrets 中获取
        parts = path.split(".")
        current = self.secrets
        for part in parts:
            if part not in current:
                return None
            current = current[part]
        
        return current
    
    def _get_base_url(self, channel: str, target: str) -> str:
        """获取目标系统的基础 URL"""
        # 从环境配置读取
        if channel == "api":
            return self.config.get("api_hosts", {}).get(target, "")
        elif channel == "web":
            return self.config.get("web_hosts", {}).get(target, "")
        return ""
```

### 4.2 在测试执行中使用

```python
# testplat_engine/executor.py

def execute_api_step(step, context, http_client, auth_manager):
    """执行 API 步骤"""
    
    # 1. 判断是否需要鉴权
    if step.anon:
        # 匿名请求,跳过鉴权
        pass
    else:
        # 2. 获取身份
        identity = step.as_ or context.default_as
        if not identity:
            raise ValueError("步骤未指定身份,且没有 default_as")
        
        # 3. 执行鉴权
        target = step.host  # API 契约的 host 字段
        auth_result = auth_manager.authenticate(
            identity=identity,
            channel="api",
            target=target,
            http_client=http_client,
        )
        
        if not auth_result.success:
            raise ValueError(f"鉴权失败: {auth_result.error}")
        
        # 4. 注入鉴权头
        if auth_result.headers:
            http_client.headers.update(auth_result.headers)
    
    # 5. 发送请求
    response = http_client.request(
        method=step.method,
        url=step.url,
        json=step.body,
    )
    
    return response
```

---

## 5. 扩展新策略的步骤

假设新公司使用了 **HMAC 签名鉴权**,需要扩展支持:

### Step 1: 实现策略插件

```python
# my_project/custom_strategies/hmac_auth.py

import hmac
import hashlib
import time
from testplat_engine.auth import LoginStrategy, AuthContext, AuthResult

class HmacAuthStrategy(LoginStrategy):
    """HMAC 签名鉴权"""
    
    @property
    def name(self) -> str:
        return "hmac_signature"
    
    def validate_credentials(self, credentials):
        if "app_key" not in credentials or "app_secret" not in credentials:
            raise ValueError("hmac_signature 需要 app_key 和 app_secret")
    
    def login(self, ctx: AuthContext, http_client) -> AuthResult:
        # HMAC 不需要登录接口,直接生成签名
        app_key = ctx.credentials["app_key"]
        app_secret = ctx.credentials["app_secret"]
        timestamp = str(int(time.time()))
        
        # 构建签名字符串
        sign_str = f"{app_key}{timestamp}{app_secret}"
        signature = hmac.new(
            app_secret.encode(),
            sign_str.encode(),
            hashlib.sha256
        ).hexdigest()
        
        return AuthResult(
            success=True,
            headers={
                "X-App-Key": app_key,
                "X-Timestamp": timestamp,
                "X-Signature": signature,
            },
        )
```

### Step 2: 注册到引擎

```python
# my_project/setup_auth.py

from testplat_engine.auth import AuthManager
from .custom_strategies.hmac_auth import HmacAuthStrategy

def setup_custom_auth(auth_manager: AuthManager):
    """注册自定义策略"""
    auth_manager.register_strategy(HmacAuthStrategy())
```

### Step 3: 配置文件使用

```yaml
# config/auth/my-api-tests.yaml

profiles:
  api:
    service_auth:
      strategy: hmac_signature  # 使用新策略
```

```yaml
# config/secrets/my-api-tests.secrets.yaml

accounts:
  service:
    default:
      app_key: "your-app-key"
      app_secret: "your-app-secret"
```

**完成!** 无需修改引擎核心代码,只需添加插件 + 配置即可。

---

## 6. 缓存与自愈机制

### 6.1 Token 缓存设计

```python
# testplat_engine/auth/cache.py

import time
import json
from pathlib import Path
from typing import Optional

class AuthCache:
    """鉴权结果缓存"""
    
    def __init__(self, cache_dir: Path, safety_margin: int = 300):
        self.cache_dir = cache_dir
        self.safety_margin = safety_margin  # 提前 5 分钟过期
        self.cache_dir.mkdir(parents=True, exist_ok=True)
    
    def get(self, scope: str) -> Optional[AuthResult]:
        """获取缓存的鉴权结果
        
        scope 格式: {env}/{project}/{identity}/{channel}/{target}
        """
        cache_file = self.cache_dir / f"{self._hash(scope)}.json"
        if not cache_file.exists():
            return None
        
        data = json.loads(cache_file.read_text())
        
        # 检查是否过期
        if data.get("expires_at"):
            if time.time() > (data["expires_at"] - self.safety_margin):
                cache_file.unlink()  # 删除过期缓存
                return None
        
        return AuthResult(**data)
    
    def set(self, scope: str, result: AuthResult):
        """保存鉴权结果"""
        cache_file = self.cache_dir / f"{self._hash(scope)}.json"
        data = {
            "success": result.success,
            "token": result.token,
            "headers": result.headers,
            "expires_at": result.expires_at,
        }
        cache_file.write_text(json.dumps(data, ensure_ascii=False))
    
    def _hash(self, text: str) -> str:
        import hashlib
        return hashlib.md5(text.encode()).hexdigest()
```

### 6.2 401 自动重登

```python
# testplat_engine/http_client_wrapper.py

class AuthenticatedHttpClient:
    """带鉴权的 HTTP 客户端"""
    
    def __init__(self, base_client, auth_manager, auth_cache):
        self.client = base_client
        self.auth_manager = auth_manager
        self.cache = auth_cache
    
    def request(self, method, url, identity, channel, target, **kwargs):
        """发送请求,自动处理鉴权"""
        
        # 1. 尝试使用缓存
        scope = f"{identity}/{channel}/{target}"
        cached_auth = self.cache.get(scope)
        
        if cached_auth and cached_auth.headers:
            kwargs.setdefault("headers", {}).update(cached_auth.headers)
        else:
            # 2. 缓存未命中,执行登录
            auth_result = self.auth_manager.authenticate(
                identity, channel, target, self.client
            )
            if auth_result.success and auth_result.headers:
                kwargs.setdefault("headers", {}).update(auth_result.headers)
                self.cache.set(scope, auth_result)
        
        # 3. 发送请求
        response = self.client.request(method, url, **kwargs)
        
        # 4. 401 自愈
        if response.status_code == 401:
            # 清除缓存,重新登录
            self.cache.clear(scope)
            auth_result = self.auth_manager.authenticate(
                identity, channel, target, self.client
            )
            if auth_result.success and auth_result.headers:
                kwargs["headers"].update(auth_result.headers)
                self.cache.set(scope, auth_result)
                # 重试请求
                response = self.client.request(method, url, **kwargs)
        
        return response
```

---

## 7. 配置迁移指南

### 从 testplat-new 迁移到通用平台

#### 原配置 (testplat-new)
```yaml
# config/envs/_global.yaml
auth:
  identities:
    gsb:
      credentials: accounts.{project}.{role}.{variant}
      scope: project
  
  profiles:
    api:
      gsb_bearer:
        strategy: http_form
        params:
          url: /api/b2/users/sign_in
          token_path: $.result.token
```

#### 迁移后 (testplat 通用版)
```yaml
# config/auth/my-company.yaml
identities:
  gsb:
    credentials: accounts.{project}.{role}.{variant}
    scope: project

profiles:
  api:
    gsb_auth:
      strategy: http_form
      params:
        url: /api/b2/users/sign_in
        success_path: $.errorCode
        success_value: "0"
        token_path: $.result.token
        header_name: Authorization
        header_prefix: "Bearer "
```

**差异**:
- ✅ 配置文件路径调整 (`config/envs/` → `config/auth/`)
- ✅ 新增 `success_path` 和 `success_value` (显式判断成功)
- ✅ 新增 `header_name` 和 `header_prefix` (显式声明注入方式)

---

## 8. 最佳实践

### 8.1 配置分层原则（历史表格；当前归属见文首及 4.1 设计）

| 配置类型 | 存放位置 | 是否入 git | 说明 |
|---------|---------|-----------|------|
| 身份定义 | `config/auth/_global.yaml` | ✅ | 跨项目共享的身份契约 |
| 策略模板 | `config/auth/_global.yaml` | ✅ | 通用登录策略模板 |
| 项目 profile | `config/auth/{project}.yaml` | ✅ | 项目专属登录配置 |
| 敏感凭据 | `config/secrets/{project}.secrets.yaml` | ❌ | 账号密码/token |

### 8.2 命名约定

- **身份名**: 小写 + 下划线,如 `admin`, `buyer_vip`
- **profile 名**: `{身份}_{通道}_{方式}`,如 `admin_api_bearer`, `buyer_web_cookie`
- **策略名**: 小写 + 下划线,如 `http_form`, `bearer_token`, `oauth2`

### 8.3 安全建议

- ✅ 凭据文件添加到 `.gitignore`
- ✅ 生产环境使用环境变量覆盖敏感值
- ✅ Token 缓存文件设置合理的过期时间
- ✅ 登录失败时不要在日志中暴露完整密码
- ✅ 使用 `RedactingObserver` 脱敏日志输出

---

## 9. 与 control 层的集成

### control 层职责
- ✅ 加载鉴权配置文件
- ✅ 初始化 `AuthManager`
- ✅ 注册项目自定义策略
- ✅ 将 `AuthManager` 传给 engine

### engine 层职责
- ✅ 定义策略插件接口
- ✅ 提供内置策略实现
- ✅ 执行鉴权并返回结果
- ✅ 不关心配置文件存放位置

### 调用示例
```python
# testplat-control/services/execution_service.py

from testplat_engine.auth import AuthManager
from testplat_engine import execute_api

async def run_api_with_auth(project_id: str, api_id: str, params: dict):
    # 1. 加载项目配置
    project = await db.get_project(project_id)
    auth_config = load_yaml(project.auth_config_path)
    secrets = load_yaml(project.secrets_path)
    
    # 2. 初始化鉴权管理器
    auth_manager = AuthManager(auth_config, secrets)
    
    # 3. 注册自定义策略 (如果有)
    if project.custom_strategy_module:
        setup_custom_auth(auth_manager)
    
    # 4. 执行 API
    result = execute_api(
        api_id=api_id,
        context=ProjectContext(...),
        params=params,
        auth_manager=auth_manager,  # 传给 engine
    )
    
    return result
```

---

## 总结

本设计通过**三层抽象 + 插件化策略**实现了通用性与扩展性的平衡:

| 维度 | 实现方式 | 通用性保障 |
|------|---------|-----------|
| 身份管理 | 配置驱动,凭据路径模板 | ✅ 不硬编码业务角色 |
| 登录策略 | 插件接口 + 内置实现 | ✅ 新增策略无需改引擎 |
| 路由匹配 | 三维 (身份/通道/目标) | ✅ 支持复杂多租户场景 |
| 凭据存储 | secrets 文件 + 路径解析 | ✅ 不限定存储结构 |
| 缓存自愈 | Token 缓存 + 401 重登 | ✅ 提升执行效率 |

**换公司场景**:
1. 编写新的策略插件 (如 HMAC/SAML)
2. 调整配置文件 (identities/profiles/routes)
3. 填写 secrets 文件

**无需修改 engine 核心代码!**
