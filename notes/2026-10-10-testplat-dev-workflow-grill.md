# testplat MVP 开发工作流: Grill / Discovery 会话
日期: 2026-10-10 · 目标: 提取项目结构、技术栈、开发流程和第一阶段任务的完整信息

## 总结 / 关键决策

**Grill 完成度**: 4/4 核心问题已确认 ✅

### 关键决策汇总

| # | 问题 | 决策 | 状态 |
|---|------|------|------|
| Q1 | Git 策略 | 独立 Git Repo (双仓库: engine + control) | ✅ |
| Q2 | 目录结构 | engine 六层架构 + control 前后端分离 | ✅ |
| Q3 | 技术栈版本 | Python 3.11 + React 18 + uv + pnpm | ✅ |
| Q4 | 第一阶段任务 | engine 能执行单个 API (最小验证) | ✅ |

---

### 第一阶段实施计划

**本周目标**: testplat-engine v0.1.0 能执行单个 API

**任务清单**:
1. ✅ 项目初始化 (pyproject.toml + src/ 结构)
2. ✅ domain 层 (Api/Step/Result)
3. ✅ adapters/filesystem (YAML 加载)
4. ✅ engine/executor (单 API 执行)
5. ✅ ports/http_client + plugins/httpx_client
6. ✅ 验证脚本 (run_api 可用)
7. ✅ 测试资产通用化 (基于 httpbin)

**验收标准**:
```python
result = run_api(
    api_id="api.common.health_check",
    registry=registry,
    env="test",
    params={}
)
assert result.status == "success"
```

---

**下一步**: 开始实施 testplat-engine 第一阶段开发!

## Q&A 日志

### 起点
- 已有: 完整需求文档、12 个 grill 决策已确认、参考项目 (testplat-new)
- 需要: 项目结构设计、技术栈版本、开发工作流、依赖顺序、第一阶段任务列表


### Q1 — 项目目录结构与 Git 策略

**提问**:
1. 目录结构如何设计?需要考虑全局需求、后续拓展和维护空间
2. 项目命名: 现在用 testplat1 (内部用 testplat),后续改名
3. Git 仓库:是分开的 git repo 还是 monorepo?需要理解区别和推荐

**用户洞察**:
- 用户未做过类似规划,需要基于全局需求推荐最合理的方案
- 项目最终叫 testplat,但目前用 testplat1 (外层目录),内部使用 testplat 标识
- 后续直接改名 testplat1 → testplat

## 基于昨天 Grill 记录的关键事实

### 已明确的架构决策 (来自 2026-10-09 grill):

1. **架构拆分**: 双仓库 (control + engine)
2. **团队规模**: 3-10 人
3. **部署方式**: 本地 + Docker (混合方案)
4. **权限控制**: MVP 简单方案 (只记录操作人,不做权限限制)
5. **并发执行**: Phase 1 串行 → Phase 2 任务队列 → Phase 3 完全并发
6. **浏览器管理**: 独立会话 / 复用 (可配置)
7. **步骤类型**: api + ui.* (MVP), 插件化扩展 (db/redis/custom)

### 关键技术细节:

**control (testplat-control)**:
- 后端: FastAPI + SQLite
- 前端: React + TypeScript + Ant Design
- 职责: Web UI、任务调度、结果展示、配置管理
- 调用方式: 通过 Python API 调用 engine (不是 HTTP)

**engine (testplat-engine)**:
- 纯 Python 包
- 从 testplat-new 提取: domain/engine/ports/plugins/adapters
- 职责: 纯粹测试执行 (HTTP 调用、断言、变量处理)
- 对外接口: `run_api()`, `run_case()`

**依赖关系** (从文档推导):
```
testplat-control
  ├── pyproject.toml
  │   dependencies:
  │     - testplat-engine >= 0.1.0  # 通过 pip 依赖
```

---

## 重新分析: 为什么文档选择了双仓库 (独立 Git Repo)?

### 原因 1: **职责边界清晰,便于跨公司复用**

**文档 1.1 节明确定位**:
> "换公司后只需调整配置文件,无需修改平台代码"

**推论**:
- **engine**: 可跨公司复用的核心引擎 (纯粹的测试执行逻辑)
- **control**: 特定部署的 Web UI 层 (可定制化)

**场景**: 换公司时
- **Monorepo**: 需要 fork 整个仓库,包含所有历史
- **双仓库**: 只需要 `testplat-engine`,重新实现或定制 control

---

### 原因 2: **技术栈隔离,构建工具链完全不同**

| 维度 | testplat-control | testplat-engine |
|------|-----------------|-----------------|
| 语言 | Python (FastAPI) + TypeScript (React) | 纯 Python |
| 构建工具 | npm/pnpm (前端) + pip/poetry (后端) | pip/poetry |
| 开发环境 | 需要 Node.js + Python | 只需要 Python |
| 打包产物 | Web 服务 (uvicorn) + 静态资源 | Python wheel 包 |

**Monorepo 的问题**:
- 需要同时配置 npm + Python 的 workspace 工具
- CI/CD 需要判断哪些改动触发哪些构建
- 开发者需要安装两套工具链 (即使只改 engine)

**双仓库的好处**:
- engine 开发者只需要 Python 环境
- control 开发者需要 Node.js + Python (依赖 engine)
- 构建流程独立,互不干扰

---

### 原因 3: **演进节奏和稳定性要求不同**

**从 grill 记录推导**:

| 模块 | 迭代特点 | 稳定性要求 |
|------|---------|-----------|
| control | UI 迭代快,用户反馈驱动 | 中等 (UI bug 影响体验,不影响测试结果) |
| engine | 核心逻辑稳定,向后兼容优先 | 高 (bug 影响测试结果准确性) |

**场景**: engine 发布了 v1.2.0 (新增 db 步骤支持)
- **Monorepo**: control 也被迫打 v1.2.0 tag (即使没改动)
- **双仓库**: control 可以继续用 engine v1.1.0,不强制升级

---

### 原因 4: **3-10 人团队的分工协作**

**从 Q1/Q2 推导的团队组成**:
- **测试人员** (5-8 人): 主要使用 control (Web UI)
- **核心开发** (1-2 人): 维护 engine 核心逻辑
- **前端开发** (1-2 人): 维护 control UI

**权限需求** (虽然 MVP 简单方案,但长远考虑):
- 测试人员: 只需要 control 代码权限 (改 UI/配置)
- 核心开发: 需要 engine 权限 (不希望测试人员误改核心逻辑)

**双仓库的好处**:
- repo 级别天然隔离权限
- PR 审查范围清晰 (engine PR 只有核心开发审查)
- 减少误操作风险

---

### 原因 5: **开源和商业化潜力**

**从文档定位推导**:
> "通用的、可跨公司复用的接口自动化测试平台"

**潜在场景**:
1. **选择性开源**: 
   - 开源 engine (吸引社区贡献)
   - control 保持私有 (商业版 UI)

2. **分层售卖**:
   - engine 免费 (开源)
   - control 商业授权

3. **定制化服务**:
   - engine 标准版
   - control 为不同客户定制 (不同 UI 风格/功能)

**Monorepo 的问题**: 难以实现选择性开源

---

## 最终结论

**文档选择双仓库 (独立 Git Repo) 的核心原因**:

1. **跨公司复用** (最重要) - engine 是独立内核
2. **技术栈隔离** - Python 纯净 engine vs 混合 control
3. **演进节奏不同** - 稳定 engine vs 快速迭代 control
4. **团队分工** - 3-10 人按职责分 repo
5. **未来扩展** - 开源/商业化潜力

---

## 依赖管理方案

**engine 作为 Python 包**:

```bash
# 发布 engine 到 PyPI (私有)
cd testplat-engine
python -m build
twine upload dist/* --repository testplat-private

# control 依赖 engine
cd testplat-control
# pyproject.toml
[project]
dependencies = [
    "testplat-engine>=0.1.0",
    "fastapi",
    "sqlalchemy",
    ...
]
```

**工作流**:
1. engine 改动 → 发布新版本 (v0.2.0)
2. control 更新依赖版本 → 测试 → 部署
3. 两个 repo 独立的 CI/CD pipeline

---

**用户确认**: ✅ 分析正确

---

## Q1 总结: Git 策略与项目结构

**已确认决策**:

### 1. Git 策略: **独立 Git Repo (双仓库)**

**仓库结构**:
```
testplat-engine/          # 仓库 1: 执行引擎
testplat-control/         # 仓库 2: Web 控制端
```

**选择理由** (已分析确认):
1. **跨公司复用** - engine 是独立可移植的内核
2. **技术栈隔离** - Python 纯净 vs 混合技术栈  
3. **演进节奏不同** - 稳定核心 vs 快速迭代 UI
4. **团队分工** - 3-10 人按职责分 repo
5. **未来扩展** - 开源/商业化潜力

### 2. 依赖管理: **engine 作为 Python 包**

```bash
# testplat-engine 发布
cd testplat-engine
python -m build
twine upload dist/* --repository testplat-private

# testplat-control 依赖
# pyproject.toml
[project]
dependencies = ["testplat-engine>=0.1.0"]
```

### 3. 项目命名

- 当前目录: `testplat1` (临时名称)
- 后续改名: `testplat1` → `testplat`
- 仓库命名:
  - `testplat-engine`
  - `testplat-control`

---

## Q2 — 目录结构设计 (双仓库各自的内部结构)

### 二次审查 (基于 testplat-new 真实结构)

参考了 testplat-new 的实际目录结构后,发现初版方案有多处遗漏和不合理之处。

#### testplat-engine 修正 (❌ 问题 → ✅ 修正)

**❌ 问题 1**: 遗漏了 `application/` 层 (鉴权、脱敏、观察者)
**❌ 问题 2**: 遗漏了 `shared/` 层 (变量渲染、工具函数)  
**❌ 问题 3**: `plugins/` 下应该有 `strategies/` 子目录 (鉴权策略)
**❌ 问题 4**: `ports/` 缺少 `browser_provider.py`

**✅ 修正后的 testplat-engine 结构**:

```
testplat-engine/
├── pyproject.toml
├── README.md
├── src/
│   └── testplat_engine/
│       ├── domain/          # 领域模型
│       │   ├── case.py, api.py, step.py, assert_rule.py, result.py
│       ├── application/     # 应用层 ← 新增
│       │   ├── auth.py      # 鉴权协调
│       │   ├── redaction.py # 脱敏逻辑
│       │   └── observers.py # 观察者 (执行进度通知)
│       ├── engine/          # 执行引擎
│       │   ├── executor.py, resolver.py, assert_engine.py
│       │   └── registry.py  # 步骤执行器注册表
│       ├── ports/           # 端口接口
│       │   ├── http_client.py
│       │   ├── security_context.py
│       │   └── browser_provider.py  ← 新增
│       ├── plugins/         # 插件实现
│       │   ├── strategies/  # 鉴权策略 ← 新增子目录
│       │   │   ├── bearer.py, form.py, oauth2.py
│       │   ├── httpx_client.py
│       │   └── playwright_driver.py
│       ├── adapters/filesystem/
│       │   ├── yaml_loader.py, registry.py, report_builder.py
│       └── shared/          # 共享工具 ← 新增
│           ├── variable_renderer.py
│           └── utils.py
├── tests/
└── docs/
```

---

#### testplat-control 修正 (❌ 问题 → ✅ 修正)

**❌ 问题 1**: `data/` 不应在项目根目录 (运行时数据应可配置路径)
**❌ 问题 2**: 缺少 `config/` 示例文件
**❌ 问题 3**: 缺少 `scripts/` (数据库初始化等)
**❌ 问题 4**: 前端构建产物 (`frontend/dist/`) 没有规划
**❌ 问题 5**: API 路由应该拆分更细 (projects/assets/executions/results)

**✅ 修正后的 testplat-control 结构**:

```
testplat-control/
├── pyproject.toml, package.json, README.md
├── docker-compose.yml, Dockerfile
│
├── backend/                 # FastAPI 后端
│   ├── main.py, config.py
│   ├── api/                 # REST API (拆分更细)
│   │   ├── projects.py      # 项目管理
│   │   ├── assets.py        # 资产浏览
│   │   ├── executions.py    # 执行管理
│   │   └── results.py       # 结果查询 ← 独立拆出
│   ├── models/              # SQLAlchemy 模型
│   │   ├── project.py, execution_record.py, base.py
│   ├── services/            # 业务逻辑
│   │   ├── asset_service.py, execution_service.py, result_service.py
│   ├── db/database.py
│   └── utils/
│
├── frontend/                # React 前端
│   ├── src/
│   │   ├── App.tsx, main.tsx
│   │   ├── pages/           # 页面组件
│   │   │   ├── ProjectList.tsx, ProjectDetail.tsx, ExecutionResult.tsx
│   │   ├── components/      # 通用组件
│   │   │   ├── AssetTree.tsx
│   │   │   ├── ResponseViewer.tsx  # 支持脱敏开关
│   │   │   └── StepTimeline.tsx
│   │   ├── api/client.ts
│   │   └── types/index.ts
│   ├── public/, index.html
│   ├── vite.config.ts       # Vite 构建工具
│   └── tsconfig.json
│
├── config/                  # 配置文件示例 ← 新增
│   ├── config.example.yaml
│   └── README.md
│
├── scripts/                 # 维护脚本 ← 新增
│   ├── init_db.py           # 初始化数据库
│   └── migrate.py
│
└── .gitignore
    # 忽略: data/, logs/, frontend/dist/
```

**运行时目录** (用户指定路径,如 `/var/lib/testplat/`):
```
/var/lib/testplat/           # 或环境变量 TESTPLAT_DATA_DIR
├── data/testplat.db         # SQLite 数据库
├── logs/                    # 日志文件
└── reports/                 # HTML 报告归档
```

---

### 关键改进总结

| 问题 | 修正 | 原因 |
|------|------|------|
| engine 缺少 application/ | ✅ 添加 | 鉴权、脱敏是应用层逻辑,不是领域层 |
| engine 缺少 shared/ | ✅ 添加 | 变量渲染等工具函数需要独立层 |
| plugins/ 结构不完整 | ✅ 添加 strategies/ | 鉴权策略是插件化的 |
| data/ 在项目根目录 | ✅ 移到运行时路径 | 支持灵活部署,便于备份 |
| 缺少 config/ 示例 | ✅ 添加 | 用户需要配置模板 |
| 缺少 scripts/ | ✅ 添加 | 数据库初始化、迁移脚本 |
| API 路由粗糙 | ✅ 细化为 4 个模块 | projects/assets/executions/results 职责更清晰 |
| 前端构建工具未定 | ✅ 使用 Vite | 现代、快速的构建工具 |

---

**修正后的结构更符合 testplat-new 的六边形架构,且为跨公司复用做好了准备。**


---

**用户确认**: ✅ 同意修正后的目录结构

---

## Q2 总结: 目录结构设计

**已确认的目录结构**:

### testplat-engine (纯 Python 包)
- ✅ 六层架构: domain / application / engine / ports / plugins / adapters / shared
- ✅ 使用 src/ 布局 (避免导入问题)
- ✅ plugins/strategies/ 子目录 (鉴权策略)
- ✅ tests/ 与源码分离

### testplat-control (FastAPI + React)
- ✅ backend/ 和 frontend/ 并列
- ✅ API 路由拆分: projects / assets / executions / results
- ✅ 运行时数据外置 (data/logs/reports 通过环境变量配置)
- ✅ 提供 config/ 示例和 scripts/ 维护工具
- ✅ 前端使用 Vite 构建

---

## Q3 — 技术栈版本与依赖


**用户确认**: 
- ✅ Python 3.11+
- ✅ Node.js 18+
- ✅ 依赖管理: **uv** (Python) + **pnpm** (Node.js)

---

### 确认的技术栈版本

#### testplat-engine (Python)
```toml
[project]
name = "testplat-engine"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "pydantic>=2.0",
    "httpx>=0.27",
    "playwright>=1.40",
    "pyyaml>=6.0",
    "jinja2>=3.1",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

**使用 uv 管理**:
```bash
# 开发环境
uv venv
uv pip install -e ".[dev]"

# 发布
uv build
```

---

#### testplat-control (FastAPI + React)

**后端 (Python)**:
```toml
[project]
dependencies = [
    "testplat-engine>=0.1.0",
    "fastapi>=0.115",
    "uvicorn[standard]>=0.30",
    "sqlalchemy>=2.0",
    "alembic>=1.13",
]
```

**前端 (TypeScript + React)**:
```json
{
  "packageManager": "pnpm@9.0.0",
  "dependencies": {
    "react": "^18.3.0",
    "react-dom": "^18.3.0",
    "antd": "^5.20.0",
    "axios": "^1.7.0",
    "react-router-dom": "^6.26.0"
  },
  "devDependencies": {
    "typescript": "^5.5.0",
    "vite": "^5.4.0",
    "@vitejs/plugin-react": "^4.3.0"
  }
}
```

**使用 pnpm 管理**:
```bash
# 安装依赖
pnpm install

# 开发
pnpm dev

# 构建
pnpm build
```

---

## Q3 总结: 技术栈版本与依赖管理

**已确认**:
- Python 3.11+ (现代特性 + 性能)
- Pydantic v2 (验证速度快)
- FastAPI + uvicorn (高性能异步)
- React 18 + Ant Design 5 (现代 UI)
- Vite 5 (极速构建)
- **uv** (Python 依赖管理,极快)
- **pnpm** (Node.js 依赖管理,节省空间)

---

## Q4 — 开发工作流与第一阶段任务


**用户明确**:

1. **代码复用策略**: 
   - ✅ 参考 testplat-new,但不直接照搬
   - ✅ 根据通用设计和可扩展性重新设计
   - ❌ 去除业务特定逻辑 (GSB/GO2/e3e3)

2. **第一阶段验收标准**:
   - ✅ **能执行单个 API** (最小目标)
   - ❌ 不需要 UI (暂缓)
   - ❌ 不需要场景运行 (暂缓)

3. **测试资产**:
   - ✅ 使用简单 API 作为测试
   - ✅ 参考: `/Users/jiavin/Developer/products/testplat-new/assets/api/gsb/b2/account/balance.yaml`
   - 改造为通用格式 (去除 GSB 业务绑定)

---

### 第一阶段任务分解 (基于用户确认)

#### 目标: **engine 能执行单个 API**

**任务优先级**:

```
Priority 0 (本周必须完成):
├── testplat-engine
│   ├── 1. 项目初始化 (pyproject.toml + src/ 结构)
│   ├── 2. domain 层 (Api/Step/Result 领域模型)
│   ├── 3. adapters/filesystem (YAML 加载器)
│   ├── 4. engine/executor (单 API 执行逻辑)
│   ├── 5. ports/http_client (HTTP 请求接口)
│   ├── 6. plugins/httpx_client (httpx 实现)
│   └── 7. 验证脚本 (run_api 函数可用)
│
└── 测试资产准备
    ├── 通用化 balance.yaml (去除 GSB 业务绑定)
    └── 准备 config/envs/test.yaml
```

**暂缓 (Phase 2)**:
- ❌ application 层 (鉴权、脱敏) - 单 API 暂不需要
- ❌ UI 步骤 (Playwright) - 不在第一阶段
- ❌ testplat-control - engine 验证通过后再开始

---

### 测试资产通用化改造

**原始 (testplat-new 的 balance.yaml)**:
```yaml
name: 查询账户余额
desc: GSB B2 公共账户余额查询接口...
host: gsb              # ← 业务绑定
method: POST
url: /api/account/balance
success:
  path: $.errorCode
  value: "0"
request:
  type: json
  inputs: {}
```

**通用化改造后**:
```yaml
# test_assets/api/common/health_check.yaml
name: 健康检查接口
desc: 通用的健康检查接口,用于验证平台基础能力
host: test_server      # ← 通用化,在 config 中配置
method: GET
url: /health
success:
  path: $.status
  value: "ok"
request:
  type: json
  inputs: {}
```

**配置文件**:
```yaml
# test_assets/config/envs/test.yaml
api_hosts:
  test_server: https://httpbin.org  # 使用公开测试接口
```

**验证目标**:
```python
# 验证脚本能成功执行
result = run_api(
    api_id="api.common.health_check",
    registry=registry,
    env="test",
    params={}
)
assert result.status == "success"
```

---

## Q4 总结: 开发工作流与第一阶段任务

**已确认的第一阶段范围**:

1. **核心目标**: engine 能执行单个 API ✅
2. **代码策略**: 参考但不照搬,重新设计 ✅
3. **测试资产**: 通用化的简单 API (基于 httpbin 等公开接口) ✅
4. **验收标准**: Python 调用 `run_api()` 成功返回结果 ✅

**暂缓功能**:
- testplat-control (Phase 2)
- UI 自动化 (Phase 2)
- 鉴权和脱敏 (Phase 2)
- 场景运行 (Phase 2)

---

**第一阶段任务清单已明确,可以开始实施!**

