# 配置管理模块设计：现行规则与历史草案

> **2026-10-15 修订**：4.1 恢复的是“测试人员在 Web 维护运行配置”的能力，不是恢复整套历史 YAML 文件编辑器。需求以 [Phase 4 基线](../../notes/phase4-roadmap.md) 为准，实施方案见 [4.1 设计稿](../../notes/phase4.1-design-draft.md)。设计尚未实现。

## 当前配置归属

| 内容 | 维护位置与行为 |
|---|---|
| 接口、case、登录流程 | Git 资产仓库、固定 commit 快照 |
| 项目内环境、业务 / 登录服务地址、超时、配置变量 | Web 表单，Control 数据库 |
| 测试账号 | Web 按项目 + 环境 + 账号名维护；多账号可引用同一 Git 登录流程 |
| 被测系统凭据 | 数据库认证加密；Web 可查看 / 编辑；主密钥 TESTPLAT_MASTER_KEY 在服务器 |
| 平台用户密码 | 只保存密码哈希；不使用测试账号凭据的可逆加密方式 |
| 运行追溯 | commit + 环境 + 操作人 + 配置 / 账号版本；历史读取启动时的摘要 |

## 当前生效与实施边界

- 环境在项目内独立管理，运行时选；可新增 Git 中不存在的环境。
- Git 环境文件仅作为初始化模板候选；首次自动导入未单独裁决，先用 Web 创建环境完成闭环。同步不能覆盖人工配置。
- 账号凭据不再要求测试人员改服务器变量或重启服务；每账号独立解析 Git 模板，不能修改 os.environ。
- 修改供下一次运行使用；已开始的执行固定配置。保存带 revision，旧版本写入返回 409。
- 4.1a 使用现有 GSB 冒烟，4.1b 扩展步骤账号选择。复杂权限、配置继承树、通用 YAML 编辑器、自动 Git 提交和迁移工具不在首批。
- 当前开发库可由执行人显式重建，程序不能自动删除；团队正式使用后再定备份 / 升级方式。

## 历史草案说明

**以下各编号章节为早期方案，不是当前接口或实施要求。** 特别是文件型 ConfigLayer、全局环境、Monaco、PUT /secrets/{env}、secrets 文件加密、自动提交 Git 和旧项目迁移工具均不直接采用。重新采用任何细节须在当前批次设计中明示。

## 设计背景

### 核心挑战
基于 testplat-new 的配置实践,测试平台的配置管理面临以下复杂性:

1. **三层配置合并**: `_global.yaml` < `{env}.yaml` < `{env}/{project}.yaml`
2. **敏感信息隔离**: secrets 文件不入 git,需独立管理
3. **多环境支持**: test/staging/prod 各自独立配置
4. **多项目支持**: 同一环境下多个项目共享公共配置,又有项目专属配置
5. **配置校验**: 确保配置有效性,避免运行时错误

### 设计目标

- ✅ **Web 界面管理**: 提供友好的配置编辑界面,降低使用门槛
- ✅ **配置继承可视化**: 清晰展示三层配置的合并结果
- ✅ **敏感信息安全**: secrets 加密存储,支持环境变量注入
- ✅ **版本控制集成**: 非敏感配置自动提交 git,保留变更历史
- ✅ **配置校验**: 保存前自动校验,避免运行时错误
- ✅ **导入导出**: 支持配置文件的导入导出和迁移

---

## 1. 配置层级结构

### 1.1 目录结构

```
config/
├── auth/                      # 鉴权配置 (通用平台新增)
│   ├── _global.yaml           # 跨项目共享的身份/策略模板
│   └── {project}.yaml         # 项目专属鉴权配置
├── envs/                      # 环境配置 (继承 testplat-new)
│   ├── _global.yaml           # 跨环境共享 (identities/profiles/routes)
│   ├── {env}.yaml             # 环境共享 (api_hosts/web_hosts/connections)
│   └── {env}/                 # 环境专属子目录
│       ├── {project1}.yaml    # 项目配置 (项目 Web 地址/profile_params)
│       └── {project2}.yaml
└── secrets/                   # 敏感信息 (不入 git)
    ├── {env}.secrets.yaml     # 环境凭据
    └── {env}.secrets.yaml.example  # 示例文件 (入 git)
```

### 1.2 配置合并优先级

```
最终配置 = _global.yaml ⊕ {env}.yaml ⊕ {env}/{project}.yaml
```

**合并规则**:
- `dict` 类型: 递归深度合并
- `list` 和标量: 高优先级层完整覆盖

**示例**:
```yaml
# _global.yaml
auth:
  identities:
    admin:
      credentials: accounts.admin.{variant}
  profiles:
    api:
      admin_auth:
        strategy: bearer_token

# test.yaml
api_hosts:
  main: https://api-test.example.com

# test/project-a.yaml
project: project-a
api_hosts:
  main: https://api-test.project-a.com  # 覆盖环境配置

# 合并结果:
auth:
  identities:
    admin:
      credentials: accounts.admin.{variant}
  profiles:
    api:
      admin_auth:
        strategy: bearer_token
api_hosts:
  main: https://api-test.project-a.com  # ← 项目层覆盖成功
project: project-a
```

---

## 2. 数据模型设计（历史草案）

### 2.1 配置实体

```python
# testplat-control/models/config.py

from enum import Enum
from typing import Dict, Any, Optional
from pydantic import BaseModel, Field

class ConfigType(str, Enum):
    """配置类型"""
    GLOBAL = "global"        # 全局配置
    ENV = "env"              # 环境配置
    PROJECT = "project"      # 项目配置
    SECRETS = "secrets"      # 敏感配置

class ConfigLayer(BaseModel):
    """配置层"""
    type: ConfigType
    name: str                # global / test / project-a
    parent: Optional[str] = None  # 父层名称 (用于面包屑导航)
    content: Dict[str, Any]  # YAML 内容
    file_path: str           # 文件路径
    is_editable: bool = True # 是否可编辑
    description: Optional[str] = None

class MergedConfig(BaseModel):
    """合并后的配置"""
    env: str
    project: str
    layers: list[ConfigLayer]  # 按优先级排序: [_global, env, project]
    merged: Dict[str, Any]     # 合并结果
    conflicts: list[str] = []  # 冲突项 (用于高亮显示)

class SecretField(BaseModel):
    """敏感字段"""
    path: str                  # 如 "accounts.admin.default.password"
    value: Optional[str] = None  # 明文值 (仅用于编辑)
    encrypted: str             # 加密值 (存储用)
    is_placeholder: bool = False  # 是否为占位符 (如 "REPLACE")
    required: bool = True      # 是否必填
```

### 2.2 配置变更记录

```python
class ConfigChange(BaseModel):
    """配置变更记录"""
    id: str
    config_type: ConfigType
    config_name: str
    changed_by: str           # 操作用户
    changed_at: datetime
    action: str               # create | update | delete
    diff: Dict[str, Any]      # 变更内容 (JSON diff)
    commit_sha: Optional[str] = None  # git commit SHA (非 secrets)
```

---

## 3. Web 界面设计（历史草案）

### 3.1 配置管理主页

```
┌─────────────────────────────────────────────────────────────┐
│  配置管理                                    [新建项目] [导入]  │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  📁 全局配置 (_global.yaml)              最后更新: 2026-10-09 │
│     跨环境共享的鉴权身份、策略模板                             │
│     [查看] [编辑]                                             │
│                                                               │
│  📁 环境配置                                                  │
│     ├─ 🟢 test                            最后更新: 2026-10-08│
│     │   ├─ test.yaml (环境配置)              [查看] [编辑]     │
│     │   ├─ test.secrets.yaml (敏感配置) 🔒   [编辑]           │
│     │   └─ 项目 (3)                                           │
│     │       ├─ project-a                     [查看] [编辑]     │
│     │       ├─ project-b                     [查看] [编辑]     │
│     │       └─ project-c                     [查看] [编辑]     │
│     ├─ 🟡 staging                                             │
│     └─ 🔴 prod                                                │
│                                                               │
│  💾 变更历史                              [查看所有变更记录]    │
│     · admin 更新了 test/project-a 配置    10 分钟前           │
│     · system 创建了新项目 project-c       1 小时前             │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 配置编辑页面

```
┌─────────────────────────────────────────────────────────────┐
│  编辑配置: test / project-a                    [保存] [取消]   │
├─────────────────────────────────────────────────────────────┤
│  面包屑: 全局 > test > project-a                              │
│                                                               │
│  ┌───────────────────┬──────────────────────────────────┐   │
│  │  配置层级         │  配置内容                          │   │
│  ├───────────────────┼──────────────────────────────────┤   │
│  │ □ _global.yaml    │  # project-a 项目配置             │   │
│  │ □ test.yaml       │  project: project-a              │   │
│  │ ☑ project-a.yaml  │                                  │   │
│  │   (当前编辑层)     │  api_hosts:                      │   │
│  │                   │    main: [___________________]    │   │
│  │ [查看合并结果]     │    payment: [_______________]    │   │
│  │                   │                                  │   │
│  │                   │  web_hosts:                      │   │
│  │                   │    portal: [________________]    │   │
│  │                   │                                  │   │
│  │                   │  auth:                           │   │
│  │                   │    profile_params:               │   │
│  │                   │      admin_auth:                 │   │
│  │                   │        timeout: [30__]           │   │
│  └───────────────────┴──────────────────────────────────┘   │
│                                                               │
│  📝 配置说明:                                                 │
│     - api_hosts.main: 主 API 服务地址                         │
│     - web_hosts.portal: 门户 Web 地址                         │
│     - 更多字段说明请查看 [配置文档]                            │
│                                                               │
│  ⚠️  校验结果:                                                │
│     ✅ YAML 语法正确                                          │
│     ✅ 必填字段完整                                           │
│     ⚠️  警告: api_hosts.main 覆盖了环境配置 (预期行为)        │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 配置合并预览

```
┌─────────────────────────────────────────────────────────────┐
│  配置合并预览: test / project-a                [下载] [关闭]   │
├─────────────────────────────────────────────────────────────┤
│  合并层级: _global.yaml ⊕ test.yaml ⊕ project-a.yaml         │
│                                                               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  api_hosts:                                           │  │
│  │    main: https://api-test.project-a.com  ← project-a │  │
│  │    payment: https://pay-test.example.com ← test      │  │
│  │                                                       │  │
│  │  auth:                                                │  │
│  │    identities:                         ← _global     │  │
│  │      admin:                                           │  │
│  │        credentials: accounts.admin.{variant}          │  │
│  │    profiles:                                          │  │
│  │      api:                                             │  │
│  │        admin_auth:                                    │  │
│  │          strategy: bearer_token        ← _global     │  │
│  │          params:                                      │  │
│  │            timeout: 30                 ← project-a   │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                               │
│  🏷️  图例:                                                    │
│     ← _global: 来自全局配置                                    │
│     ← test: 来自环境配置                                       │
│     ← project-a: 来自项目配置 (最高优先级)                     │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### 3.4 敏感配置编辑 (Secrets)

```
┌─────────────────────────────────────────────────────────────┐
│  编辑敏感配置: test.secrets.yaml         🔒 安全模式  [保存]   │
├─────────────────────────────────────────────────────────────┤
│  ⚠️  注意: 敏感配置不会被提交到 git 仓库                       │
│                                                               │
│  accounts:                                                    │
│    admin:                                                     │
│      default:                                                 │
│        username: [___________________]  👁️                   │
│        password: [*******************]  👁️  [生成密码]       │
│                                                               │
│    project-a:                                                 │
│      buyer:                                                   │
│        primary:                                               │
│          username: [___________________]  👁️                │
│          password: [*******************]  👁️                │
│                                                               │
│  logins:                                                      │
│    api_client:                                                │
│      default:                                                 │
│        token: [*********************************]  👁️        │
│                                                               │
│  connections:                                                 │
│    redis_main:                                                │
│      password: [*******************]  👁️                     │
│    mysql_main:                                                │
│      password: [*******************]  👁️                     │
│                                                               │
│  📋 从示例模板导入: [test.secrets.yaml.example]                │
│  💾 导出为环境变量脚本: [export.sh]                            │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. 后端 API 设计（历史草案）

### 4.1 配置 CRUD 接口

```python
# testplat-control/api/config.py

from fastapi import APIRouter, HTTPException, Depends
from typing import List

router = APIRouter(prefix="/api/config", tags=["配置管理"])

@router.get("/layers")
async def list_config_layers(
    env: Optional[str] = None,
    project: Optional[str] = None,
) -> List[ConfigLayer]:
    """列出配置层"""
    pass

@router.get("/merged")
async def get_merged_config(
    env: str,
    project: str,
) -> MergedConfig:
    """获取合并后的配置"""
    pass

@router.get("/layer/{type}/{name}")
async def get_config_layer(
    type: ConfigType,
    name: str,
) -> ConfigLayer:
    """获取单层配置"""
    # type=global, name=_global
    # type=env, name=test
    # type=project, name=test/project-a
    pass

@router.put("/layer/{type}/{name}")
async def update_config_layer(
    type: ConfigType,
    name: str,
    content: Dict[str, Any],
    auto_commit: bool = True,
) -> ConfigLayer:
    """更新配置层
    
    Args:
        type: 配置类型
        name: 配置名称
        content: 新的配置内容
        auto_commit: 是否自动提交到 git (仅非 secrets)
    """
    # 1. 校验 YAML 语法
    # 2. 校验必填字段
    # 3. 写入文件
    # 4. 如果是非 secrets,提交 git
    # 5. 记录变更历史
    pass

@router.post("/validate")
async def validate_config(
    env: str,
    project: str,
    content: Dict[str, Any],
) -> Dict[str, Any]:
    """校验配置
    
    Returns:
        {
            "valid": true,
            "errors": [],
            "warnings": ["api_hosts.main 覆盖了环境配置"],
        }
    """
    pass

@router.get("/secrets/{env}")
async def get_secrets_template(env: str) -> Dict[str, Any]:
    """获取 secrets 模板 (脱敏)"""
    # 读取 {env}.secrets.yaml.example
    # 或生成基于当前配置的模板
    pass

@router.put("/secrets/{env}")
async def update_secrets(
    env: str,
    secrets: Dict[str, Any],
    encrypt: bool = True,
) -> Dict[str, Any]:
    """更新敏感配置
    
    Args:
        env: 环境名
        secrets: 敏感配置内容
        encrypt: 是否加密存储 (推荐 True)
    """
    pass

@router.get("/changes")
async def list_config_changes(
    limit: int = 50,
    config_type: Optional[ConfigType] = None,
) -> List[ConfigChange]:
    """获取配置变更历史"""
    pass
```

### 4.2 配置导入导出

```python
@router.post("/import")
async def import_config(
    file: UploadFile,
    target_env: str,
    target_project: Optional[str] = None,
    override: bool = False,
) -> Dict[str, Any]:
    """导入配置文件
    
    支持格式:
    - 单个 YAML 文件
    - ZIP 压缩包 (包含多个配置文件)
    """
    pass

@router.get("/export")
async def export_config(
    env: str,
    project: Optional[str] = None,
    include_secrets: bool = False,
    format: str = "yaml",  # yaml | json | zip
) -> FileResponse:
    """导出配置文件"""
    pass

@router.post("/clone")
async def clone_config(
    source_env: str,
    source_project: str,
    target_env: str,
    target_project: str,
) -> Dict[str, Any]:
    """克隆配置 (用于快速创建新项目)"""
    pass
```

---

## 5. 配置校验规则

### 5.1 结构校验

```python
# testplat-control/services/config_validator.py

from pydantic import BaseModel, ValidationError

class ConfigValidator:
    """配置校验器"""
    
    def validate_global(self, content: Dict[str, Any]) -> ValidationResult:
        """校验全局配置"""
        errors = []
        warnings = []
        
        # 1. 必须包含 auth.identities
        if "auth" not in content or "identities" not in content["auth"]:
            errors.append("缺少 auth.identities 配置")
        
        # 2. identities 必须定义 credentials 模板
        for identity_name, identity in content.get("auth", {}).get("identities", {}).items():
            if "credentials" not in identity:
                errors.append(f"身份 {identity_name} 缺少 credentials 模板")
            if "scope" not in identity:
                errors.append(f"身份 {identity_name} 缺少 scope 字段")
        
        return ValidationResult(valid=len(errors) == 0, errors=errors, warnings=warnings)
    
    def validate_env(self, content: Dict[str, Any]) -> ValidationResult:
        """校验环境配置"""
        errors = []
        warnings = []
        
        # 1. api_hosts 必须是 dict
        if "api_hosts" in content and not isinstance(content["api_hosts"], dict):
            errors.append("api_hosts 必须是字典类型")
        
        # 2. URL 格式校验
        for host_name, url in content.get("api_hosts", {}).items():
            if not url.startswith(("http://", "https://")):
                errors.append(f"api_hosts.{host_name} 不是有效的 URL: {url}")
        
        return ValidationResult(valid=len(errors) == 0, errors=errors, warnings=warnings)
    
    def validate_project(self, content: Dict[str, Any], env_content: Dict[str, Any]) -> ValidationResult:
        """校验项目配置"""
        errors = []
        warnings = []
        
        # 1. 必须声明 project 字段
        if "project" not in content:
            errors.append("项目配置必须包含 project 字段")
        
        # 2. 检测覆盖情况
        if "api_hosts" in content:
            for host_name in content["api_hosts"]:
                if host_name in env_content.get("api_hosts", {}):
                    warnings.append(f"api_hosts.{host_name} 覆盖了环境配置 (预期行为)")
        
        return ValidationResult(valid=len(errors) == 0, errors=errors, warnings=warnings)
    
    def validate_secrets(self, content: Dict[str, Any], identities: Dict[str, Any]) -> ValidationResult:
        """校验敏感配置"""
        errors = []
        warnings = []
        
        # 1. 检查必填账号是否存在
        for identity_name, identity in identities.items():
            template = identity.get("credentials", "")
            # 简化示例: 检查 accounts.admin.default 是否存在
            # 实际需要解析模板并检查所有可能的路径
            if template.startswith("accounts."):
                # TODO: 实现模板解析逻辑
                pass
        
        # 2. 检查占位符
        for path, value in self._flatten_dict(content).items():
            if isinstance(value, str) and value.startswith("REPLACE"):
                warnings.append(f"{path} 仍为占位符,请填入真实值")
        
        return ValidationResult(valid=len(errors) == 0, errors=errors, warnings=warnings)
    
    def _flatten_dict(self, d: Dict[str, Any], parent_key: str = "") -> Dict[str, Any]:
        """扁平化字典 (用于检查占位符)"""
        items = []
        for k, v in d.items():
            new_key = f"{parent_key}.{k}" if parent_key else k
            if isinstance(v, dict):
                items.extend(self._flatten_dict(v, new_key).items())
            else:
                items.append((new_key, v))
        return dict(items)
```

### 5.2 校验结果模型

```python
class ValidationResult(BaseModel):
    """校验结果"""
    valid: bool
    errors: List[str] = []      # 阻塞性错误,不能保存
    warnings: List[str] = []    # 警告,可以保存但需提示用户
    info: List[str] = []        # 信息提示
```

---

## 6. 配置加密存储（历史草案）

### 6.1 加密策略

**方案选择**: AES-256-GCM (推荐)

```python
# testplat-control/services/secret_encryptor.py

from cryptography.hazmat.primitives.ciphers.aead import AESGCM
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2
import os
import base64

class SecretEncryptor:
    """敏感配置加密器"""
    
    def __init__(self, master_key: str):
        """
        Args:
            master_key: 主密钥 (从环境变量 TESTPLAT_MASTER_KEY 读取)
        """
        self.aesgcm = AESGCM(self._derive_key(master_key))
    
    def _derive_key(self, password: str) -> bytes:
        """从主密钥派生加密密钥"""
        kdf = PBKDF2(
            algorithm=hashes.SHA256(),
            length=32,
            salt=b"testplat-salt",  # 固定 salt (生产环境应随机生成并存储)
            iterations=100000,
        )
        return kdf.derive(password.encode())
    
    def encrypt(self, plaintext: str) -> str:
        """加密"""
        nonce = os.urandom(12)
        ciphertext = self.aesgcm.encrypt(nonce, plaintext.encode(), None)
        # 格式: base64(nonce + ciphertext)
        return base64.b64encode(nonce + ciphertext).decode()
    
    def decrypt(self, encrypted: str) -> str:
        """解密"""
        data = base64.b64decode(encrypted)
        nonce, ciphertext = data[:12], data[12:]
        plaintext = self.aesgcm.decrypt(nonce, ciphertext, None)
        return plaintext.decode()
    
    def encrypt_secrets_file(self, secrets: Dict[str, Any]) -> Dict[str, Any]:
        """加密整个 secrets 文件"""
        return self._encrypt_recursive(secrets)
    
    def decrypt_secrets_file(self, encrypted_secrets: Dict[str, Any]) -> Dict[str, Any]:
        """解密整个 secrets 文件"""
        return self._decrypt_recursive(encrypted_secrets)
    
    def _encrypt_recursive(self, obj: Any) -> Any:
        """递归加密"""
        if isinstance(obj, dict):
            return {k: self._encrypt_recursive(v) for k, v in obj.items()}
        elif isinstance(obj, list):
            return [self._encrypt_recursive(item) for item in obj]
        elif isinstance(obj, str) and self._is_sensitive_field(obj):
            return self.encrypt(obj)
        else:
            return obj
    
    def _decrypt_recursive(self, obj: Any) -> Any:
        """递归解密"""
        if isinstance(obj, dict):
            return {k: self._decrypt_recursive(v) for k, v in obj.items()}
        elif isinstance(obj, list):
            return [self._decrypt_recursive(item) for item in obj]
        elif isinstance(obj, str) and self._looks_encrypted(obj):
            try:
                return self.decrypt(obj)
            except:
                return obj  # 解密失败,可能不是加密字段
        else:
            return obj
    
    def _is_sensitive_field(self, value: str) -> bool:
        """判断是否为敏感字段 (需要加密)"""
        # 排除占位符
        if value.startswith("REPLACE"):
            return False
        # 长度判断 (密码通常 > 3 个字符)
        if len(value) < 3:
            return False
        return True
    
    def _looks_encrypted(self, value: str) -> bool:
        """判断是否为加密数据"""
        try:
            # 加密数据是 base64 编码且长度 >= 12 (nonce) + 密文
            decoded = base64.b64decode(value)
            return len(decoded) > 12
        except:
            return False
```

### 6.2 主密钥管理

**生产环境**:
```bash
# 环境变量注入
export TESTPLAT_MASTER_KEY="your-secure-master-key-here"

# 或使用密钥管理服务 (AWS KMS / HashiCorp Vault)
```

**开发环境**:
```bash
# .env.local (不入 git)
TESTPLAT_MASTER_KEY=dev-master-key-do-not-use-in-prod
```

---

## 7. Git 集成（历史草案）

### 7.1 自动提交策略

```python
# testplat-control/services/config_git.py

import git
from pathlib import Path

class ConfigGitManager:
    """配置文件 Git 管理器"""
    
    def __init__(self, repo_path: Path):
        self.repo = git.Repo(repo_path)
    
    def commit_config_change(
        self,
        file_path: Path,
        message: str,
        author_name: str,
        author_email: str,
    ) -> str:
        """提交配置变更
        
        Returns:
            commit SHA
        """
        # 1. 添加文件到暂存区
        self.repo.index.add([str(file_path)])
        
        # 2. 提交
        commit = self.repo.index.commit(
            message,
            author=git.Actor(author_name, author_email),
        )
        
        return commit.hexsha
    
    def get_config_history(
        self,
        file_path: Path,
        limit: int = 50,
    ) -> List[Dict[str, Any]]:
        """获取配置文件变更历史"""
        commits = list(self.repo.iter_commits(paths=str(file_path), max_count=limit))
        return [
            {
                "sha": c.hexsha[:8],
                "message": c.message.strip(),
                "author": c.author.name,
                "date": c.committed_datetime.isoformat(),
            }
            for c in commits
        ]
    
    def diff_config(
        self,
        file_path: Path,
        commit_sha: Optional[str] = None,
    ) -> str:
        """对比配置文件变更"""
        if commit_sha:
            commit = self.repo.commit(commit_sha)
            return commit.diff(commit.parents[0], paths=str(file_path))
        else:
            # 对比工作区与 HEAD
            return self.repo.git.diff("HEAD", str(file_path))
```

### 7.2 .gitignore 规则

```gitignore
# config/.gitignore

# 敏感配置不入库
secrets/*.secrets.yaml

# 保留示例文件
!secrets/*.secrets.yaml.example

# 临时文件
*.tmp
*.bak
```

---

## 8. 配置迁移工具（历史草案）

### 8.1 从 testplat-new 迁移

```python
# testplat-control/tools/migrate_from_testplat_new.py

class ConfigMigrator:
    """配置迁移工具"""
    
    def migrate_from_testplat_new(
        self,
        source_dir: Path,
        target_dir: Path,
    ) -> Dict[str, Any]:
        """从 testplat-new 迁移配置
        
        Args:
            source_dir: testplat-new 的 config/ 目录
            target_dir: testplat (通用版) 的 config/ 目录
        
        Returns:
            迁移报告
        """
        report = {
            "migrated_files": [],
            "skipped_files": [],
            "warnings": [],
        }
        
        # 1. 迁移 envs/ 目录 (保持结构)
        envs_source = source_dir / "envs"
        envs_target = target_dir / "envs"
        
        for yaml_file in envs_source.rglob("*.yaml"):
            relative_path = yaml_file.relative_to(envs_source)
            target_file = envs_target / relative_path
            
            # 复制文件
            target_file.parent.mkdir(parents=True, exist_ok=True)
            content = self._load_yaml(yaml_file)
            
            # 去除业务特定配置 (可选)
            cleaned_content = self._remove_business_specific(content)
            
            self._save_yaml(target_file, cleaned_content)
            report["migrated_files"].append(str(relative_path))
        
        # 2. 迁移 secrets/ 目录
        secrets_source = source_dir / "secrets"
        secrets_target = target_dir / "secrets"
        
        for yaml_file in secrets_source.glob("*.secrets.yaml.example"):
            target_file = secrets_target / yaml_file.name
            target_file.parent.mkdir(parents=True, exist_ok=True)
            shutil.copy(yaml_file, target_file)
            report["migrated_files"].append(f"secrets/{yaml_file.name}")
        
        # 3. 生成迁移报告
        report["warnings"].append("请手动检查并填写 secrets/*.secrets.yaml 文件")
        
        return report
    
    def _remove_business_specific(self, content: Dict[str, Any]) -> Dict[str, Any]:
        """去除业务特定配置
        
        可选操作,移除 GSB/GO2/e3e3 等硬编码的项目标识
        """
        # 示例: 移除特定身份
        if "auth" in content and "identities" in content["auth"]:
            identities = content["auth"]["identities"]
            business_ids = ["gsb", "df_admin", "df_picker"]  # 根据需要调整
            for bid in business_ids:
                if bid in identities:
                    # 可以选择删除或重命名
                    pass
        
        return content
```

### 8.2 CLI 命令

```bash
# 迁移命令
testplat-control migrate \
  --source /path/to/testplat-new/config \
  --target /path/to/testplat/config

# 校验配置
testplat-control config validate --env test --project project-a

# 导出配置
testplat-control config export --env test --project project-a --output project-a.yaml

# 导入配置
testplat-control config import --file project-a.yaml --env test
```

---

## 9. 最佳实践

### 9.1 配置分层原则

| 配置内容 | 放置位置 | 原因 |
|---------|---------|------|
| 身份定义 (identities) | `_global.yaml` | 跨环境共享 |
| 鉴权策略模板 (profiles) | `_global.yaml` | 避免重复定义 |
| 路由规则 (routes) | `_global.yaml` | 统一管理 |
| API/Web 地址 | `{env}.yaml` | 环境差异 |
| 连接配置 (connections) | `{env}.yaml` | 环境差异 |
| 项目专属 Web 地址 | `{env}/{project}.yaml` | 项目差异 |
| profile 参数覆盖 | `{env}/{project}.yaml` | 项目差异 |
| 账号密码 | `secrets/{env}.secrets.yaml` | 敏感信息 |

### 9.2 命名规范

- **环境名**: 小写,如 `test`, `staging`, `prod`
- **项目名**: 小写 + 连字符,如 `project-a`, `my-api-tests`
- **host 名**: 小写 + 下划线,如 `api_main`, `web_portal`
- **身份名**: 小写 + 点分隔,如 `admin`, `buyer.vip`

### 9.3 安全建议

- ✅ 生产环境主密钥使用密钥管理服务 (AWS KMS)
- ✅ secrets 文件权限设为 `600` (仅所有者可读写)
- ✅ CI/CD 通过环境变量注入 secrets,不入镜像
- ✅ 定期轮换密码和 token
- ✅ 记录所有配置变更操作日志

---

## 10. 实现优先级

### Phase 1: 基础功能 (1 周)
- [ ] 配置文件 CRUD API
- [ ] 三层配置合并逻辑
- [ ] 基础校验规则
- [ ] Web 界面骨架 (列表+编辑)

### Phase 2: 安全功能 (3-5 天)
- [ ] secrets 加密存储
- [ ] 敏感字段脱敏显示
- [ ] 主密钥管理

### Phase 3: 增强功能 (3-5 天)
- [ ] Git 自动提交
- [ ] 配置变更历史
- [ ] 配置合并预览
- [ ] 导入导出工具

### Phase 4: 迁移工具 (2-3 天)
- [ ] testplat-new 配置迁移脚本
- [ ] 配置模板生成器
- [ ] CLI 命令行工具

---

## 总结

本设计提供了一个完整的配置管理方案:

| 特性 | 实现方式 | 优势 |
|------|---------|------|
| 三层配置 | 深度合并算法 | 灵活复用,减少重复 |
| Web 管理 | FastAPI + React | 降低使用门槛 |
| 敏感保护 | AES-256-GCM 加密 | 安全存储 secrets |
| 版本控制 | Git 自动提交 | 保留变更历史 |
| 配置校验 | Pydantic 验证 | 避免运行时错误 |
| 可视化预览 | 层级高亮显示 | 理解配置来源 |

**换公司场景**:
1. 创建新项目配置文件
2. 填写环境地址 (API/Web hosts)
3. 定义身份和鉴权策略 (复用通用模板)
4. 填写 secrets 文件

**无需修改平台代码!**
