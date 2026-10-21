# testplat-engine Phase 1 实施计划
日期: 2026-10-10 · 目标: engine 能执行单个 API

## 执行策略: 多 Agent 并行 + 主线监控

### 角色分工

| 角色 | 模型 | 职责 |
|------|------|------|
| 监控者 (主线程) | Opus | 任务拆解、接口设计、Agent prompt 编写、产出审查 |
| 执行者 (Agent) | Sonnet | 单模块编码、测试编写、文档更新 |

### 并行执行原则
- 能并行的模块用多个 Agent 同时执行
- 每个 Agent 完成后由主线检查产出
- 使用 TaskCreate 跟踪进度

---

## Phase 1 任务拆解

### 模块依赖关系图

```
┌──────────────┐
│ 项目初始化    │ (Task 1)
└──────┬───────┘
       ↓
┌──────────────────────────────────────┐
│     并行开发 (Task 2-5)              │
│  ┌─────────┐  ┌─────────┐           │
│  │ domain  │  │adapters │           │
│  │  层     │  │/yaml    │           │
│  └────┬────┘  └────┬────┘           │
│       ↓            ↓                 │
│  ┌────────────────────┐              │
│  │    ports/http     │              │
│  │    (接口定义)      │              │
│  └─────────┬──────────┘              │
│            ↓                         │
│  ┌─────────────────────┐             │
│  │ plugins/httpx_client│             │
│  └─────────┬────────────┘            │
└────────────┼─────────────────────────┘
             ↓
    ┌────────────────┐
    │ engine/executor│ (Task 6)
    └────────┬───────┘
             ↓
    ┌───────────────┐
    │  测试资产准备  │ (Task 7)
    └────────┬──────┘
             ↓
    ┌───────────────┐
    │  集成验证     │ (Task 8)
    └───────────────┘
```

---

## 详细任务清单

### Task 1: 项目初始化 (串行,主线完成)
**负责人**: 主线 (Opus)  
**时间**: 10 分钟  
**产出**:
- testplat-engine/ 目录结构
- pyproject.toml (uv 配置)
- README.md (项目说明)
- src/testplat_engine/__init__.py
- tests/ 目录结构

**验收**:
```bash
cd testplat-engine
uv venv
uv pip install -e .
python -c "import testplat_engine; print(testplat_engine.__version__)"
```

---

### Task 2: domain 层 (并行 Agent 1)
**负责人**: Agent-domain  
**依赖**: Task 1  
**时间**: 20-30 分钟  
**产出**:
- `src/testplat_engine/domain/api.py` - Api 领域模型
- `src/testplat_engine/domain/step.py` - Step 领域模型
- `src/testplat_engine/domain/result.py` - Result 领域模型
- `tests/domain/test_api.py` - 单元测试

**接口设计**:
```python
# api.py
@dataclass
class Api:
    id: str
    name: str
    method: str
    url: str
    host: str
    request: dict
    success: dict
```

**验收**:
```bash
uv run pytest tests/domain/ -v
```

---

### Task 3: adapters/filesystem YAML 加载 (并行 Agent 2)
**负责人**: Agent-yaml-loader  
**依赖**: Task 1  
**时间**: 20-30 分钟  
**产出**:
- `src/testplat_engine/adapters/filesystem/yaml_loader.py`
- `src/testplat_engine/adapters/filesystem/registry.py`
- `tests/adapters/test_yaml_loader.py`

**接口设计**:
```python
class Registry:
    def __init__(self, assets_root: Path):
        ...
    
    def load_api(self, api_id: str) -> Api:
        """加载 API 定义 (如 api.common.health_check)"""
        ...
```

**验收**:
```bash
uv run pytest tests/adapters/ -v
```

---

### Task 4: ports/http_client (并行 Agent 3)
**负责人**: Agent-ports  
**依赖**: Task 1  
**时间**: 15-20 分钟  
**产出**:
- `src/testplat_engine/ports/http_client.py` - 接口定义
- `tests/ports/test_http_client.py` - 接口测试

**接口设计**:
```python
from abc import ABC, abstractmethod

class HttpClient(ABC):
    @abstractmethod
    async def request(
        self, 
        method: str, 
        url: str, 
        **kwargs
    ) -> HttpResponse:
        pass

@dataclass
class HttpResponse:
    status_code: int
    headers: dict
    body: Any
```

**验收**:
```bash
uv run pytest tests/ports/ -v
```

---

### Task 5: plugins/httpx_client (并行 Agent 4)
**负责人**: Agent-httpx  
**依赖**: Task 4 (需要 ports 接口)  
**时间**: 20-30 分钟  
**产出**:
- `src/testplat_engine/plugins/httpx_client.py`
- `tests/plugins/test_httpx_client.py`

**实现要点**:
```python
class HttpxClient(HttpClient):
    def __init__(self):
        self._client = httpx.AsyncClient()
    
    async def request(self, method, url, **kwargs):
        response = await self._client.request(method, url, **kwargs)
        return HttpResponse(
            status_code=response.status_code,
            headers=dict(response.headers),
            body=response.json()
        )
```

**验收**:
```bash
uv run pytest tests/plugins/ -v
```

---

### Task 6: engine/executor (串行,主线完成)
**负责人**: 主线 (Opus)  
**依赖**: Task 2-5 全部完成  
**时间**: 30-40 分钟  
**产出**:
- `src/testplat_engine/engine/executor.py`
- `src/testplat_engine/engine/resolver.py` (变量解析)
- `src/testplat_engine/engine/assert_engine.py` (断言)
- `tests/engine/test_executor.py`

**核心函数**:
```python
async def run_api(
    api_id: str,
    registry: Registry,
    env: str,
    params: dict = None
) -> ApiResult:
    # 1. 加载 API 定义
    api = registry.load_api(api_id)
    
    # 2. 渲染变量
    resolved = resolver.resolve(api, params)
    
    # 3. 执行 HTTP 请求
    response = await http_client.request(
        method=api.method,
        url=resolved.url,
        json=resolved.body
    )
    
    # 4. 断言
    result = assert_engine.check(response, api.success)
    
    return ApiResult(status=result.status, response=response)
```

**验收**:
```bash
uv run pytest tests/engine/ -v
```

---

### Task 7: 测试资产准备 (串行,主线完成)
**负责人**: 主线 (Opus)  
**依赖**: Task 6  
**时间**: 10-15 分钟  
**产出**:
- `test_assets/api/common/health_check.yaml`
- `test_assets/config/envs/test.yaml`

**示例 API**:
```yaml
# test_assets/api/common/health_check.yaml
name: 健康检查
host: httpbin
method: GET
url: /status/200
success:
  path: $.status  # httpbin 不返回 JSON,改为检查 status_code
  value: 200
request:
  type: json
  inputs: {}
```

**配置**:
```yaml
# test_assets/config/envs/test.yaml
api_hosts:
  httpbin: https://httpbin.org
```

---

### Task 8: 集成验证 (串行,主线完成)
**负责人**: 主线 (Opus)  
**依赖**: Task 7  
**时间**: 10 分钟  
**产出**:
- `examples/run_simple_api.py` - 验证脚本
- 执行日志

**验证脚本**:
```python
# examples/run_simple_api.py
import asyncio
from pathlib import Path
from testplat_engine import run_api
from testplat_engine.adapters.filesystem import Registry

async def main():
    registry = Registry(assets_root=Path("test_assets"))
    
    result = await run_api(
        api_id="api.common.health_check",
        registry=registry,
        env="test",
        params={}
    )
    
    print(f"✅ 执行成功!")
    print(f"Status: {result.status}")
    print(f"Response: {result.response}")

if __name__ == "__main__":
    asyncio.run(main())
```

**验收**:
```bash
cd testplat-engine
uv run python examples/run_simple_api.py
# 期望输出: ✅ 执行成功!
```

---

## 执行时间表

| 阶段 | 时间 | 任务 |
|------|------|------|
| 1. 初始化 | 10 min | Task 1 (主线) |
| 2. 并行开发 | 30 min | Task 2-5 (4 个 Agent 并行) |
| 3. 集成开发 | 40 min | Task 6 (主线) |
| 4. 验证准备 | 15 min | Task 7-8 (主线) |
| **总计** | **~1.5 小时** | |

---

## 质量检查点

### 每个 Agent 完成后:
- [ ] 代码符合 Python 3.11 类型提示规范
- [ ] 单元测试覆盖率 > 80%
- [ ] 通过 ruff 检查 (格式化 + lint)
- [ ] 文档字符串完整 (Google 风格)

### 最终验收:
- [ ] `uv run pytest` 全部通过
- [ ] `examples/run_simple_api.py` 执行成功
- [ ] 能加载 YAML 资产
- [ ] 能执行 HTTP 请求
- [ ] 能验证响应断言

---

## 风险与应对

| 风险 | 概率 | 应对 |
|------|------|------|
| Agent 产出不符合接口设计 | 中 | 主线严格审查,立即修正 |
| httpbin.org 访问失败 | 低 | 提供 mock 方案 |
| 依赖冲突 | 低 | uv 自动处理 |

---

**准备就绪,等待用户确认开始执行!**
