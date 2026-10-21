# Phase 2.5 Agent 提示词：真实 Engine 集成

日期：2026-10-11  
目标：把 `testplat-control` 的单 API 执行从隐式 sibling 路径和测试替身，收敛为可安装、可诊断、可替换的真实 `testplat-engine` 集成。

> 本文是执行提示词，不是实现代码。Phase 2.5 不增加新的前端功能；只稳定 Control–Engine seam、真实执行链路、错误分类、部署方式和验收证据。

## 0. 启动前硬门禁

### 0.1 当前事实

- `testplat-control` 已通过 PR #34 合入 `main`，合并提交为 `954e264`。
- Phase 2.4 已覆盖项目 CRUD、资产浏览、单 API 表单/结果、执行历史和响应式 UI。
- 当前 Control 通过 `src/testplat_control/adapters/engine_adapter.py` 修改 `sys.path`，尝试从固定 sibling 目录导入 Engine。
- 干净 Control checkout 未安装 Engine 时，应用仍可启动，但 `POST .../run` 返回 500。
- FakeEngine 双视口门禁可通过；这只能证明 Control/UI，不是“真实 Engine 已集成”的证据。
- `testplat-engine` 当前 Git 仓库的 `main @ e202b2f` 只跟踪 `README.md`；`pyproject.toml`、`src/`、`tests/`、`examples/` 等实现仍是未跟踪文件，且仓库没有 remote。

### 0.2 未满足时禁止编码

Phase 2.5 Agent 1–4 启动前，主线程必须给出：

1. Engine 当前实现的只读审计结论。
2. 经用户确认的 Engine 基线提交 SHA；基线提交不得混入 Phase 2.5 功能修改。
3. 可供 clean-machine 安装的 Engine remote、tag/commit 和锁定方式。

未取得这些信息时，只能执行 Agent 0；其他 Agent 必须停止。

### 0.3 已批准决策（2026-10-11）

#### Engine 分发

`testplat-engine` 保持为独立仓库，并建立 remote 和 `v0.1.x` 版本来源：

- Control 的正式依赖锁定 Engine 的兼容版本或 Git commit。
- 本地开发可以使用显式 editable override，但不得把机器绝对路径、worktree 名或 `sys.path` 注入写进运行时代码。
- CI 和“换一台机器”验收必须从正式依赖来源安装，不依赖开发者现有目录。

这是已批准的目标分发方式；实际 remote URL、首个基线 tag/commit 仍须在 Agent 0 审计后由主线程记录，不能由实现 Agent 猜测。

#### 系统错误记录

批准规则：

- 参数校验、资产不存在、环境配置无效等执行前错误不创建执行记录。
- 已进入真实 HTTP 执行后发生的 timeout、transport、Engine internal error，创建 `failed` 记录并在 `result.error.kind` 中保留分类；HTTP 仍返回对应的 5xx。
- 业务断言失败是正常结果：HTTP 200、`status=failed`，并创建记录。

公开 Engine 异常必须携带 `stage=preflight|execution`，让 Control 能稳定执行这条记录规则；不能根据错误文案或异常类名猜测阶段。

### 0.4 Agent 0 审计结果（2026-10-11）

Agent 0 已完成只读审计，仓库状态未改变。当前仍停在 Engine 来源基线门禁：

- HEAD 为 `e202b2fa77b1bebea4f83a83ee925984d7f420e2`，仅跟踪 `README.md`，无 remote/tag；README 还有未提交修改。
- 可发现 pytest 为 `80 passed / 1 failed`，Ruff 为 49 项；包可构建，但 sdist 会误收嵌套 `testplat-engine/` 历史目录。
- `.venv/`、`.DS_Store`、`__pycache__`、`.pyc` 等生成物必须由根 `.gitignore` 排除；`.venv/pyvenv.cfg` 含本机绝对路径。
- 已确认三个必须在 Phase 2.5 修复的契约风险：`api_id` 可逃逸 `assets_root`、系统异常被吞成业务 `failed`、`HttpResponse` 不能直接 JSON 序列化。
- README/文档存在“生产就绪”、MIT License 和未实现能力等超前声明；外部 `../test_assets` 与公网 `httpbin.org` 也不能成为 clean-machine 门禁依赖。
- 基线整理已提交为本地 `409a049 chore: capture pre-Phase 2.5 engine baseline`；主线程复跑为 `83 passed`、Ruff 全绿、sdist/wheel 内容干净。该提交尚未推送、没有 remote/tag，不能替代正式分发来源。
- Agent 1 Interface 已在 `codex/phase2-5-engine-interface` 提交为 `bee429d feat: define Phase 2.5 engine interface contract`。两轮双轴审查后的 Spec/Standards 阻塞项均已清零；主线程复跑为 `97 passed`、Ruff/build 全绿。该提交尚未推送。

在主线程确认基线纳入清单、测试资产归属、依赖裁剪、许可证表述和 remote 之前，Agent 1–4 继续停止。

## 1. 已选架构

采用单入口深 Module；综合“最小接口”和“安全默认调用”方案。

```text
FastAPI route
  -> ExecutionService                # Control 编排、计时、持久化、HTTP 映射
     -> EnginePort.execute(request)  # 唯一执行 seam
        ├─ PackageEngineAdapter      # 真实 testplat-engine
        └─ FakeEngineAdapter         # 测试/显式验收 harness
```

### 1.1 Interface 目标形状

最终命名可按仓库现有风格调整，但信息和不变量不得漂移：

```python
@dataclass(frozen=True)
class ApiExecutionRequest:
    api_id: str
    assets_root: Path
    environment: Mapping[str, JsonValue]
    params: Mapping[str, JsonValue]
    timeout_seconds: float = 30.0

@dataclass(frozen=True)
class ApiExecutionOutcome:
    status: Literal["success", "failed"]
    request: JsonObject | None
    response: JsonValue | None
    error: JsonObject | None

class EnginePort(Protocol):
    async def execute(self, request: ApiExecutionRequest) -> ApiExecutionOutcome: ...
```

不变量：

- Interface 不接收 SQLAlchemy `Project`，不返回 `ExecutionRecord`。
- Control 拥有数据库模型、记录生命周期和现有 `RunResponse`。
- Engine 拥有资产加载、变量解析、HTTP client、断言和底层异常归一化。
- Control 加载并校验 `config_root` 中的环境配置，再把解析后的环境传给 Engine；Engine 不猜 Control 的目录布局。
- `ApiExecutionOutcome` 必须可 JSON 序列化。
- 业务断言失败返回 outcome，不抛系统异常。
- 系统异常至少携带稳定的 `kind`、`stage` 和安全外显信息；底层 cause 只进入日志链路。
- Phase 2.5 只支持 `api.*`；不提前实现 Case target、进度回调、报告、重试、任务队列或远程 Engine。

### 1.2 错误分类

Engine 的公开异常必须稳定、可判别，不能把所有异常都吞成业务 `failed`：

| kind | 默认 stage | 含义 | Control HTTP | 执行历史 |
|---|---|---|---|---|
| `asset_not_found` | `preflight` | API 资产不存在 | 404 | 不创建 |
| `invalid_asset` | `preflight` | YAML/资产结构无效 | 422 | 不创建 |
| `invalid_environment` | `preflight` | 环境配置缺失或无效 | 422 | 不创建 |
| `timeout` | `execution` | HTTP 请求超时 | 504 | 创建 `failed` |
| `transport` | `execution` | DNS、连接等传输失败 | 502 | 创建 `failed` |
| `unavailable` | `preflight` | Engine 未安装或版本不兼容 | 启动 fail-fast；运行期兜底 503 | 不创建 |
| `internal` | 由抛出点明确提供 | 未分类 Engine 缺陷 | 500 | `execution` 创建；`preflight` 不创建 |

错误对外只暴露安全、可读信息；完整异常栈写日志，不返回给浏览器。

### 1.3 兼容性要求

- 现有前端 `RunResponse`、业务 `success/failed` 语义和历史查询契约不得漂移。
- 删除 `sys.path` 注入、静默 `ImportError` sentinel 和路由内 `EngineAdapter()` 硬编码。
- 默认 provider 使用真实 Engine；Fake 只能由测试或显式验收 harness 注入。
- 缺少正式 Engine 依赖或版本不兼容时，应用启动必须给出明确诊断，不能等到用户点击“执行”才返回模糊 500。
- 新增运行时依赖、发布方式或跨仓库协议变化，必须先由主线程批准。

## 2. 执行编排

```text
Agent 0：Engine 基线只读审计（硬门禁）
                 |
                 v
主线程：用户确认基线提交，并记录 remote/版本来源与已批准错误规则
                 |
                 v
Agent 1：Engine 公共 Interface + 契约测试（串行基础）
                 |
                 +--> Agent 2：Engine 真实实现 --------+
                 |                                      |
                 +--> Agent 3：Control Port/Provider ----+--> Agent 4：跨仓库集成验收
```

- Agent 1 的 Interface commit 必须先独立审查。
- Agent 2、Agent 3 只能从已审查的 Interface commit 开始，可并行。
- 每个仓库独立分支、独立 commit；不得把两个仓库混成一个提交。
- Agent 4 只做契约允许的集成修复、测试和文档，不扩展 Phase 3 功能。
- 主线程负责最终 Spec/Standards 双轴审查、中文 comment pass、推送和 PR 决策。

建议分支：

```text
codex/phase2-5-engine-interface
codex/phase2-5-engine-runtime
codex/phase2-5-control-integration
codex/phase2-5-integration
```

## 3. Agent 0：Engine 基线只读审计

复制下面整段给 Agent 0：

````markdown
你负责 Phase 2.5 启动前的 `testplat-engine` 基线只读审计。只读，不提交、不暂存、不创建分支、不格式化文件。

## 仓库

- `/Users/jiavin/Developer/products/testplat1/testplat-engine`
- 当前已知 `main @ e202b2f` 只跟踪 `README.md`，其余实现大多未跟踪。

## 任务

1. 记录 `git status --short --branch`、`git ls-files`、HEAD 和 remote。
2. 清点所有未跟踪文件，区分：源码、测试、文档、示例、缓存/生成物、可疑嵌套目录。
3. 检查 `.gitignore` 是否缺失；列出应忽略的 `__pycache__`、`.pyc`、`.DS_Store`、虚拟环境、构建产物等。
4. 运行现有可发现测试和静态检查；不得为了让测试通过而修改文件。
5. 审计公开 Interface、配置查找、HTTP client、异常处理和包构建是否自洽。
6. 给出“建议纳入基线 / 建议忽略 / 需要人工确认”的逐路径清单。
7. 提议一个纯基线 commit 的内容和 commit message；不要执行 commit。

## 停止条件

- 发现凭证、个人数据、绝对机器配置或来源不明二进制时立即报告。
- 发现嵌套仓库、历史产物或旧系统复制代码时，不得自行纳入基线。

## 交付

仅报告证据、风险、路径清单、测试结果和建议基线。不写 Phase 2.5 代码。
````

## 4. Agent 1：Engine 公共 Interface 与契约

复制下面整段给 Agent 1：

````markdown
你负责 Phase 2.5 的 `testplat-engine` 公共 Interface 和契约测试。必须从主线程提供的“已确认 Engine 基线 commit”创建独立分支/worktree。

## 开始前

- 记录基线 SHA、工作区状态、Python/uv 版本。
- 若基线 SHA 未提供，或工作区还有来源不明的未跟踪实现，停止。
- 阅读 `docs/architecture/control-engine-interface.md`、现有 `executor.py`、`domain/result.py`、Registry 和 HTTP client。

## 文件所有权

你只拥有 Engine 的公共 Interface、公开异常、导出声明及对应契约测试。不要重写执行实现。

## 交付要求

1. 建立单入口 `execute(request) -> outcome` 的公开 Interface。
2. 使用冻结、类型化、可 JSON 序列化的 request/outcome DTO。
3. 建立稳定异常族和 `kind + stage(preflight|execution)`；异常信息区分安全外显文本与日志原因。
4. 业务 `failed` 是 outcome，不是异常。
5. Interface 不引用 Control、ORM、FastAPI 或数据库模型。
6. 只定义单 API 能力；不要实现 Case、回调、报告、重试、远程 transport。
7. 写契约测试：success、business failed、每种系统错误的 kind/stage、安全信息、非法 outcome、JSON 序列化。
8. 更新 Engine 公共导出和版本说明，声明 `0.1.x` 兼容范围。
9. 新增/修改代码写有价值的中文原因注释；完成后运行完整 comment pass。

## 验收

```bash
uv run ruff check .
uv run pytest -q
uv build
```

先完成自审和验证，再提交一个范围单一的本地 commit。不要推送、不要建 PR。
````

## 5. Agent 2：Engine 真实执行实现

复制下面整段给 Agent 2：

````markdown
你负责在已审查的 Engine Interface 基线上实现真实 `PackageEngineAdapter` 背后的单 API 执行 Module。

## 范围

- Registry/资产加载。
- 已解析环境配置的消费，不再从 `assets_root.parent` 猜 `config_root`。
- 变量解析、HTTP 请求、timeout、断言。
- 底层异常到公开异常族的集中转换。
- HTTP client 生命周期和资源释放。

## 约束

1. 不接触 Control、数据库或 FastAPI。
2. 不 catch-all 为业务 `failed`；未知异常转换为 `internal`，按实际抛出点标注 stage，并保留日志 cause。
3. timeout 必须真正传入 HTTP client。
4. 响应和错误必须可 JSON 序列化；不得泄露密钥、Authorization、Cookie。
5. 使用本地 HTTP test server 或注入的 HttpClient 测试，不访问公网。
6. 保留现有 API YAML 能力；发现格式冲突时先报告，不静默扩展 DSL。
7. 不实现 AuthManager、Case、UI、自动重试、并发队列。
8. 中文注释解释配置所有权、异常分类和资源清理原因；完成 comment pass。

## 测试矩阵

- success。
- 断言失败 outcome。
- 资产不存在/无效。
- 环境 host 缺失。
- timeout。
- transport error。
- 非 JSON 响应。
- 敏感 header 不进入公开结果或日志断言。

## 验收

```bash
uv run ruff check .
uv run pytest -q
uv build
```

提交一个本地 commit，不推送、不建 PR。
````

## 6. Agent 3：Control Port、Provider 与持久化

复制下面整段给 Agent 3：

````markdown
你负责 `testplat-control` 的 EnginePort、provider、环境配置加载、错误映射和执行记录编排。基线必须是已合并 Phase 2.4 的最新 `main`，并使用 Agent 1 已审查的 Engine Interface 版本。

## 先读事实

- `src/testplat_control/api/run.py`
- `src/testplat_control/adapters/engine_adapter.py`
- `src/testplat_control/database/models.py`
- `tests/test_run_api.py`
- `tests/integration/test_api_execution_flow.py`
- `frontend/src/features/executions/` 的现有响应消费

## 交付要求

1. 定义小型 `EnginePort` Protocol，并通过 FastAPI provider/依赖注入获取；路由不得直接构造 Adapter。
2. 实现真实 `PackageEngineAdapter`；Engine 依赖来自主线程批准的正式来源。
3. 删除运行时 `sys.path` 修改和静默 optional import。
4. 默认使用真实 Adapter；Fake 只供测试或显式验收 harness 注入。
5. Control 从 `project.config_root` 加载并校验当前 `env` 配置，把解析结果传给 Engine。
6. Control 负责计时、执行记录、`result_json` 和现有 `RunResponse`；Interface 不接触 ORM。
7. 保持业务 failed=HTTP 200。按批准的错误表映射 404/422/502/503/504/500。
8. 按已批准的 `stage` 规则处理系统错误记录：`preflight` 不创建，`execution` 创建 `failed`；不得自行改变历史语义。
9. 启动时校验 Engine 包和兼容版本；诊断必须说明缺包或版本不兼容，不能延迟成点击后的模糊 500。
10. 现有 OpenAPI response schema、前端 DTO 和历史详情必须保持兼容。
11. 中文注释说明 provider、记录所有权、错误映射和配置 seam；完成 comment pass。

## 测试

- provider 默认选择真实 Adapter。
- Fake 注入不依赖 monkeypatch 路由全局符号。
- 缺包/版本不兼容 fail-fast。
- success、business failed、各系统错误及记录行为。
- `RunResponse` 和 execution detail 的契约回归。

## 验收

```bash
uv run ruff check .
uv run pytest -q -rs
cd frontend
pnpm lint
pnpm test --run
pnpm build
```

提交一个本地 commit，不推送、不建 PR。
````

## 7. Agent 4：跨仓库集成与验收

复制下面整段给 Agent 4：

````markdown
你负责 Phase 2.5 的跨仓库集成与最终候选，不扩展产品功能。

## 输入

- 已审查的 Engine Interface commit。
- Agent 2 Engine runtime commit。
- Agent 3 Control integration commit。
- 主线程批准的 Engine 分发来源和系统错误记录规则。

## 集成步骤

1. 在两个仓库分别创建集成分支；每个仓库保持独立提交历史。
2. 安装 Engine 必须走批准的依赖来源；证明全新 venv 不依赖 `sys.path` 或开发者已有 checkout。
3. 编译并比较 Control 的 OpenAPI：Phase 2.4 已有端点和成功响应结构不得漂移。
4. 使用本地 HTTP test server 跑真实 Engine：success、business failed、timeout、transport、invalid environment。
5. FakeEngine 继续跑完整后端集成测试和双视口浏览器门禁。
6. 至少完成一次真实 Engine 的 UI/API smoke，证明浏览器 -> Control -> Engine -> 本地目标 -> 历史详情全链路。
7. 模拟“换一台机器”：干净 worktree、干净 venv、只按 README 安装和配置即可运行。
8. 更新 Control/Engine README：安装来源、版本、启动、Fake harness、故障排查。
9. 对两个仓库的完整 diff 分别运行中文 comment pass。

## 必须证明

- 缺 Engine 或版本不兼容在启动时给出明确诊断。
- 业务 failed 仍是 HTTP 200。
- 参数/资产/配置错误与 timeout/transport/internal 可区分。
- Fake 与真实 Adapter 通过同一 Interface 契约套件。
- 没有绝对路径、worktree 名、凭证或真实网络依赖进入提交。
- 前端 P2 主题、移动布局和 Phase 2.4 能力没有回退。

## 最终门禁

```bash
# Engine
uv run ruff check .
uv run pytest -q
uv build

# Control
uv run ruff check .
uv run pytest -q -rs
cd frontend
pnpm lint
pnpm test --run
pnpm build
pnpm e2e:gate
```

## 交付格式

1. 两个仓库的基线、候选分支和 commit。
2. 依赖来源与版本锁定方式。
3. Interface 和错误映射摘要。
4. 真实 Engine、FakeEngine、浏览器与 clean-machine 证据。
5. OpenAPI 兼容结论。
6. 已知限制和 Phase 3 前置条件。
7. 两个仓库的中文 comment pass 结论。

不要推送、不要建 PR；由主线程完成独立审查后决定。
````

## 8. 主线程质量审查

Agent 交付后，主线程按以下顺序验收：

1. 检查 Engine 基线来源与提交纯度。
2. 单独审查 Interface：Depth、Seam、错误不变量、版本兼容。
3. 分别做 Engine 和 Control 的 Standards / Spec 双轴审查。
4. 复跑两个仓库的全部门禁，不只接受 Agent 报告。
5. 对真实 Engine 做 mutation 检查：断开 Engine、删环境 host、制造 timeout、返回业务失败，验证精确分类。
6. 检查 OpenAPI、前端 DTO、历史记录和 P2 主题没有回退。
7. 完成跨仓库中文 comment pass。
8. 用户明确授权后再推送、建 PR 或合并。

## 9. Phase 2.5 明确非目标

- Case/场景执行。
- WebSocket 或进度推送。
- AuthManager、登录策略、凭证管理界面。
- 自动重试、任务队列、并发调度。
- 远程 Engine REST/gRPC。
- HTML 报告、导出、趋势图。
- 配置编辑器或在线 YAML 编辑。

发现这些需求时记录为 Phase 3 或后续阶段，不在 Phase 2.5 扩展。
