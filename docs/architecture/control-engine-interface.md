# testplat-control 与 testplat-engine 接口设计

> **2026-10-15 当前需求**：环境、host、变量和测试账号在 Web 按项目维护；账号按“项目 + 环境 + 账号名”定位。登录流程放 Git。先完成 Web 配置与 GSB 冒烟闭环，再逐步加入步骤账号选择、跨系统场景和后期 Web 编辑 case。平台注册 / 登录记录操作人，现阶段不做超管分级与复杂权限。详见 [需求基线](../../notes/phase4-roadmap.md) 和 [4.1 设计稿](../../notes/phase4.1-design-draft.md)。以下 2026-10-14 说明记录 4.0 当时范围，与本次要求冲突处已被取代；新设计不表示已实现。

> **2026-10-14 变更（Phase 4.0）**：新增第三个仓库 `testplat-assets`（用例与配置，Git 维护）。Control 负责同步仓库并为每次执行提供某个 commit 的只读快照；Engine 接口与职责不变。§6.1 传递方式与 §10.2 目录结构已更新。注：本文代码片段为早期设计草图，现行接口以 Engine `docs/COMPATIBILITY-0.5.md` 为准。

## 1. 职责边界

### testplat-control (控制仓库)
**定位**: 用户交互层 + 任务编排层

**核心职责**:
- ✅ Web UI 展示与交互
- ✅ 项目内环境 / 测试账号管理；Git 清单派生的项目本身只读
- ✅ 测试资产解析与展示 (读取 YAML 文件树)
- ✅ 执行任务调度 (调用 engine)
- ✅ 结果归档与查询 (SQLite)
- ✅ 实时进度推送 (WebSocket)

**不负责**:
- ❌ HTTP 请求发送
- ❌ 断言引擎
- ❌ 变量渲染
- ❌ 鉴权策略执行

### testplat-engine (执行引擎)
**定位**: 纯粹的测试执行内核

**核心职责**:
- ✅ YAML DSL 解析
- ✅ HTTP 客户端封装 (httpx)
- ✅ 变量渲染与上下文管理
- ✅ 断言引擎 (17 种算子)
- ✅ 步骤编排与结果序列化
- ✅ 插件机制 (鉴权/数据库/Redis)

**不负责**:
- ❌ Web UI
- ❌ 项目配置管理
- ❌ 结果持久化存储 (只生成 JSON/HTML)
- ❌ 任务调度

---

## 2. 通信接口设计

### 2.1 接口形式

**方案选择**: Python 包依赖 (推荐)

```python
# testplat-control 的 pyproject.toml
[project]
dependencies = [
    "testplat-engine>=0.1.0",
]
```

**理由**:
- ✅ 类型安全 (直接调用 Python 函数)
- ✅ 无序列化开销
- ✅ 错误栈透明
- ✅ 调试方便

**备选方案**: gRPC / REST API (未来可扩展)
- 场景: 需要支持其他语言调用 engine
- 成本: 需要维护 proto 定义 + 序列化逻辑

---

### 2.2 核心接口定义

```python
# testplat_engine/interface.py

from pathlib import Path
from typing import Any, Dict, List, Optional
from dataclasses import dataclass
from enum import Enum

# ============ 输入数据结构 ============

@dataclass
class ProjectContext:
    """项目上下文 (从 control 传入)"""
    project_id: str
    env: str  # test/staging/prod
    assets_root: Path
    config_root: Path
    secrets_root: Path

@dataclass
class ExecutionConfig:
    """执行配置"""
    timeout: int = 30
    retry: int = 0
    http_log_level: str = "failed"  # none/summary/full/failed
    fail_fast: bool = False

# ============ 输出数据结构 ============

class StepStatus(Enum):
    SUCCESS = "success"
    FAILED = "failed"
    SKIPPED = "skipped"

@dataclass
class StepResult:
    """单步骤结果"""
    name: str
    status: StepStatus
    duration: float  # 秒
    request: Optional[Dict[str, Any]] = None
    response: Optional[Dict[str, Any]] = None
    error: Optional[str] = None
    assertions: List[Dict[str, Any]] = None

@dataclass
class ApiResult:
    """单接口执行结果"""
    api_id: str
    status: StepStatus
    duration: float
    request: Dict[str, Any]
    response: Dict[str, Any]
    assertions: List[Dict[str, Any]]
    error: Optional[str] = None

@dataclass
class CaseResult:
    """场景执行结果"""
    case_id: str
    status: StepStatus
    duration: float
    steps: List[StepResult]
    assertions: List[Dict[str, Any]]
    error: Optional[str] = None

# ============ 主入口函数 ============

def execute_api(
    api_id: str,
    context: ProjectContext,
    params: Dict[str, Any],
    config: ExecutionConfig = ExecutionConfig(),
) -> ApiResult:
    """
    执行单个 API 调用
    
    Args:
        api_id: API 资产 ID (如 "user.login")
        context: 项目上下文
        params: 请求参数 (覆盖 API 定义的默认值)
        config: 执行配置
        
    Returns:
        ApiResult: 执行结果
        
    Raises:
        AssetNotFoundError: 资产文件不存在
        YamlParseError: YAML 解析失败
        ExecutionError: 执行过程异常
    """
    pass

def execute_case(
    case_id: str,
    context: ProjectContext,
    config: ExecutionConfig = ExecutionConfig(),
    callback: Optional[Callable[[StepResult], None]] = None,
) -> CaseResult:
    """
    执行测试场景
    
    Args:
        case_id: 场景资产 ID (如 "user.login_success")
        context: 项目上下文
        config: 执行配置
        callback: 步骤完成回调 (用于实时推送进度)
        
    Returns:
        CaseResult: 执行结果
    """
    pass

def export_html_report(
    result: CaseResult,
    output_path: Path,
) -> None:
    """
    导出 HTML 报告
    
    Args:
        result: 场景执行结果
        output_path: 输出文件路径
    """
    pass
```

---

## 3. 调用流程示例

### 3.1 单接口运行

```python
# testplat-control/services/execution_service.py

from testplat_engine import execute_api, ProjectContext, ExecutionConfig

async def run_api_task(
    project_id: str,
    api_id: str,
    params: dict,
) -> dict:
    """执行单接口任务 (control 层的业务逻辑)"""
    
    # 1. 加载项目配置
    project = await db.get_project(project_id)
    context = ProjectContext(
        project_id=project_id,
        env=project.env,
        assets_root=Path(project.assets_root),
        config_root=Path(project.config_root),
        secrets_root=Path(project.secrets_root),
    )
    
    # 2. 调用 engine 执行
    result = execute_api(
        api_id=api_id,
        context=context,
        params=params,
        config=ExecutionConfig(http_log_level="full"),
    )
    
    # 3. 保存执行记录
    record = await db.save_execution_record(
        project_id=project_id,
        asset_type="api",
        asset_id=api_id,
        status=result.status.value,
        duration=result.duration,
        result_json=result.to_dict(),
    )
    
    # 4. 返回结果
    return {
        "record_id": record.id,
        "status": result.status.value,
        "duration": result.duration,
        "request": result.request,
        "response": result.response,
        "assertions": result.assertions,
    }
```

### 3.2 场景运行 (带实时推送)

```python
# testplat-control/services/execution_service.py

from testplat_engine import execute_case, StepResult

async def run_case_task(
    project_id: str,
    case_id: str,
    websocket_session: WebSocket,
) -> dict:
    """执行场景任务,并实时推送进度"""
    
    # 1. 加载项目配置
    project = await db.get_project(project_id)
    context = ProjectContext(...)
    
    # 2. 定义步骤完成回调
    async def on_step_complete(step: StepResult):
        await websocket_session.send_json({
            "type": "step_complete",
            "data": {
                "name": step.name,
                "status": step.status.value,
                "duration": step.duration,
            }
        })
    
    # 3. 调用 engine 执行
    result = execute_case(
        case_id=case_id,
        context=context,
        callback=on_step_complete,
    )
    
    # 4. 生成 HTML 报告
    report_path = Path(f"/var/testplat/reports/{record_id}.html")
    export_html_report(result, report_path)
    
    # 5. 保存执行记录
    record = await db.save_execution_record(
        project_id=project_id,
        asset_type="case",
        asset_id=case_id,
        status=result.status.value,
        duration=result.duration,
        report_path=str(report_path),
    )
    
    return {"record_id": record.id, "report_url": f"/reports/{record_id}"}
```

---

## 4. 数据流图

```
┌──────────────────────────────────────────────────┐
│  Web Browser                                     │
│  - 用户点击"运行"按钮                              │
└────────────────┬─────────────────────────────────┘
                 │ HTTP POST /api/execute/case
                 ▼
┌──────────────────────────────────────────────────┐
│  testplat-control (FastAPI)                      │
│  ┌────────────────────────────────────────────┐ │
│  │ ExecutionService                           │ │
│  │ - 加载项目配置                              │ │
│  │ - 构建 ProjectContext                      │ │
│  │ - 调用 engine.execute_case()               │ │
│  └──────────────┬─────────────────────────────┘ │
└─────────────────┼───────────────────────────────┘
                  │ Python 函数调用
                  ▼
┌──────────────────────────────────────────────────┐
│  testplat-engine                                 │
│  ┌────────────────────────────────────────────┐ │
│  │ execute_case()                             │ │
│  │ ├─ 加载 YAML 资产                           │ │
│  │ ├─ 构建执行上下文 (变量/鉴权)                │ │
│  │ └─ 循环执行步骤:                            │ │
│  │    ├─ HTTP 请求 (httpx)                    │ │
│  │    ├─ 变量渲染                              │ │
│  │    ├─ 断言判定                              │ │
│  │    └─ 调用 callback 推送进度                │ │
│  └──────────────┬─────────────────────────────┘ │
└─────────────────┼───────────────────────────────┘
                  │ return CaseResult
                  ▼
┌──────────────────────────────────────────────────┐
│  testplat-control                                │
│  - 保存执行记录到 SQLite                          │
│  - 生成 HTML 报告                                 │
│  - WebSocket 推送完成通知                         │
└────────────────┬─────────────────────────────────┘
                 │ WebSocket message
                 ▼
┌──────────────────────────────────────────────────┐
│  Web Browser                                     │
│  - 显示执行结果                                   │
│  - 提供"查看报告"链接                             │
└──────────────────────────────────────────────────┘
```

---

## 5. 错误处理约定

### 5.1 异常分类

```python
# testplat_engine/exceptions.py

class EngineError(Exception):
    """引擎基础异常"""
    pass

class AssetNotFoundError(EngineError):
    """资产文件不存在"""
    def __init__(self, asset_id: str):
        self.asset_id = asset_id
        super().__init__(f"Asset not found: {asset_id}")

class YamlParseError(EngineError):
    """YAML 解析失败"""
    def __init__(self, file_path: str, detail: str):
        self.file_path = file_path
        self.detail = detail
        super().__init__(f"Failed to parse {file_path}: {detail}")

class ExecutionError(EngineError):
    """执行过程异常 (HTTP 错误、断言失败等)"""
    def __init__(self, step_name: str, detail: str):
        self.step_name = step_name
        self.detail = detail
        super().__init__(f"Step '{step_name}' failed: {detail}")
```

### 5.2 control 层处理

```python
# testplat-control/api/execution.py

from testplat_engine.exceptions import AssetNotFoundError, ExecutionError

@router.post("/execute/case")
async def execute_case_endpoint(request: ExecuteCaseRequest):
    try:
        result = await execution_service.run_case_task(...)
        return {"success": True, "data": result}
    
    except AssetNotFoundError as e:
        raise HTTPException(
            status_code=404,
            detail=f"测试资产不存在: {e.asset_id}"
        )
    
    except ExecutionError as e:
        # 执行失败不是 5xx 错误,而是正常的业务结果
        return {
            "success": False,
            "error": {
                "step": e.step_name,
                "detail": e.detail,
            }
        }
    
    except Exception as e:
        logger.exception("Unexpected error during execution")
        raise HTTPException(status_code=500, detail="Internal server error")
```

---

## 6. 配置传递约定

### 6.1 配置传递（2026-10-15 修订，4.1 待实现）

1. Control 固定本次资产 commit，并定位只读快照。
2. Control 按项目和所选环境从数据库读取地址、配置与测试账号；结合快照内 Git 登录流程构造 environment / AuthTable。
3. Control 经 EnginePort 传入 assets_root、已解析 environment、AuthTable、业务参数和有效超时。现有 Engine 已支持这两类运行材料，无须让 Engine 读数据库。
4. Engine 解释 API / case、执行登录与 HTTP、计算结果；Control 保存 env、commit、操作人和配置版本。

### 6.2 边界与兼容

- 平台会话和用户信息不进入被测系统 AuthTable，操作人与测试账号分别记录。
- 凭据不写入业务 params、Git 快照或进程环境变量；每个账号独立解析模板，保持 SecretStr 约束。
- 现有基础跨步骤 alias 引用保留；步骤 account 选择是 4.1b 的契约增量，当前严格 schema 不能直接接受。
- 项目环境由数据库管理，新增环境不要求 Git 存在同名文件。运行全程使用一致配置，修改只影响后续执行。
- 本文其余 ProjectContext / secrets_root / AuthManager 等代码为历史草图；现行协议以 Engine interface.py、兼容文档与相应批次裁决为准。

---

## 7. 性能与并发考虑

### 7.1 执行模式

**MVP 阶段**: 串行执行
- 同一时间只执行一个任务
- 实现简单,无并发冲突

**未来扩展**: 任务队列 (Celery / Redis Queue)
```python
# 异步任务模式
@celery.task
def execute_case_async(case_id: str, context: dict):
    result = execute_case(case_id, ProjectContext(**context))
    # 完成后发送 WebSocket 通知
```

### 7.2 超时控制

```python
# engine 内部使用 httpx 的超时机制
import httpx

client = httpx.Client(timeout=config.timeout)
response = client.post(url, json=data)  # 超时会抛出 httpx.TimeoutException
```

---

## 8. 版本兼容性策略

### 8.1 接口稳定性承诺

**Semantic Versioning**:
- `0.x.y`: MVP 阶段,接口可能变更
- `1.0.0`: 稳定版本,遵循向后兼容原则

**变更策略**:
- ✅ 新增字段/参数 (可选) → Patch 版本
- ⚠️ 废弃旧接口 (保留 deprecated 警告) → Minor 版本
- ❌ 删除接口/必填参数 → Major 版本

### 8.2 协议演进示例

```python
# v0.1.0
def execute_case(case_id: str, context: ProjectContext) -> CaseResult:
    pass

# v0.2.0 (新增可选参数)
def execute_case(
    case_id: str,
    context: ProjectContext,
    config: ExecutionConfig = ExecutionConfig(),  # 新增,但有默认值
) -> CaseResult:
    pass

# v1.0.0 (弃用旧参数)
def execute_case(
    case_id: str,
    context: ProjectContext,
    config: ExecutionConfig = ExecutionConfig(),
    # legacy_param: Optional[str] = None,  # 废弃,发出 DeprecationWarning
) -> CaseResult:
    pass
```

---

## 9. 测试策略

### 9.1 engine 单元测试

```python
# tests/test_engine.py

def test_execute_api_success():
    context = ProjectContext(...)
    result = execute_api(
        api_id="user.login",
        context=context,
        params={"username": "admin", "password": "123"},
    )
    assert result.status == StepStatus.SUCCESS
    assert result.response["code"] == 0

def test_execute_case_with_callback():
    step_callbacks = []
    
    def callback(step: StepResult):
        step_callbacks.append(step.name)
    
    result = execute_case(
        case_id="user.login_success",
        context=context,
        callback=callback,
    )
    
    assert len(step_callbacks) == 3  # 假设场景有 3 个步骤
```

### 9.2 control 集成测试

```python
# tests/integration/test_execution_api.py

async def test_execute_case_endpoint(client: TestClient):
    response = await client.post(
        "/api/execute/case",
        json={
            "project_id": "test-project",
            "case_id": "user.login_success",
        }
    )
    
    assert response.status_code == 200
    data = response.json()
    assert data["success"] is True
    assert "record_id" in data["data"]
```

---

## 10. 部署与运维

### 10.1 包发布

```bash
# engine 作为独立 Python 包发布
cd testplat-engine
uv build
uv publish

# control 依赖 engine
cd testplat-control
uv add testplat-engine==0.1.0
```

### 10.2 目录结构示例

```
/opt/testplat/
├── control/           # FastAPI 服务
│   ├── main.py
│   ├── api/
│   ├── services/
│   └── db/
├── repos/             # 用例仓库镜像与 commit 快照（2026-10-14 变更，平台只读）
│   └── main/
│       ├── mirror.git
│       └── snap/<sha>/   # testplat-assets 某个 commit 的只读快照
│           ├── testplat.yaml
│           └── projects/gsb/{api,scenarios,config}
└── reports/           # 执行报告归档
    ├── 2026-10-09/
    └── 2026-10-10/
```

---

## 总结

本接口设计遵循以下原则:

1. **关注点分离**: control 负责编排,engine 负责执行
2. **类型安全**: 使用 dataclass 定义所有数据结构
3. **回调机制**: 支持实时进度推送
4. **错误透明**: 异常分类清晰,便于 control 层处理
5. **配置外置**: engine 不直接读文件,由 control 加载后传入

这样的设计使得:
- ✅ engine 可以独立测试和版本管理
- ✅ control 可以灵活控制执行策略 (串行/并发/超时)
- ✅ 未来可以扩展为 gRPC/REST API 模式
