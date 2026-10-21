# Phase 2.4 Agent 提示词：Web UI

日期：2026-10-10  
目标：基于 `testplat-control` 当前 `main` 的真实 REST 契约，交付可用的 React Web UI。

> 本文是执行提示词，不是实现代码。每个 Agent 必须先检查仓库事实，再在自己的分支或 worktree 中工作。旧系统只能作为只读证据，不能复制业务模型或引入运行时依赖。

## 1. 已确认基线

### 1.1 仓库与版本

- 工作仓库：`/Users/jiavin/Developer/products/testplat1/testplat-control`
- Phase 2.3 基线：`main`，提交 `93dd99e`
- 前端目录：`testplat-control/frontend/`，当前尚未创建
- 已确认技术栈：React + TypeScript + Vite + Ant Design，包管理器使用 pnpm
- 后端：FastAPI + SQLite；前端只调用 `testplat-control`，不得直接调用 `testplat-engine`

### 1.2 当前后端契约

实现前以 `src/testplat_control/api/`、`app.openapi()` 和后端测试为最终事实。当前可用端点如下：

| 方法 | 路径 | 用途 |
|---|---|---|
| `GET` | `/health` | 健康检查 |
| `GET` / `POST` | `/projects` | 项目列表、创建项目 |
| `GET` / `PUT` / `DELETE` | `/projects/{project_id}` | 项目详情、更新、删除 |
| `GET` | `/projects/{project_id}/assets` | 获取资产树 |
| `GET` | `/projects/{project_id}/assets/{asset_id}` | 获取资产元信息 |
| `GET` | `/projects/{project_id}/assets/{asset_id}/params` | 获取 API 参数定义 |
| `POST` | `/projects/{project_id}/assets/{asset_id}/run` | 执行单个 API |
| `GET` | `/projects/{project_id}/execution-records` | 查询执行记录 |
| `GET` | `/projects/{project_id}/execution-records/{record_id}` | 查询执行详情 |

需要特别遵守：

- `POST /run` 的 HTTP 200 既可能是业务 `success`，也可能是业务 `failed`；两种结果都必须展示并刷新历史。
- 参数不合法返回 422；项目或资产不存在返回 404；前端需展示后端 `detail`，不能把所有错误统称为网络错误。
- 参数只支持 `string | number | boolean`。空的非必填输入应省略，不能擅自发送错误类型或 `undefined`。
- 资产树包含目录和多类文件，但当前只有 `api.*` 文件拥有参数与运行契约。
- 执行记录列表可传 `status`、`asset_id`、`order_by`、`skip`、`limit`；响应是数组，没有总数。
- 当前没有 case 运行、配置管理、脱敏规则、记录删除、导出、WebSocket 或上传接口。

### 1.3 Phase 2.4 范围

本阶段必须交付：

1. 项目列表，以及项目的创建、编辑、删除。
2. 项目详情工作台和只读资产树。
3. 单 API 的动态参数表单、执行结果展示。
4. 项目执行历史的状态/资产筛选、列表和详情。
5. 加载、空数据、失败、404、提交中和禁用状态。
6. 桌面端优先且在窄屏可用的响应式布局。

本阶段不实现：

- case 或 UI 自动化运行。
- 配置编辑、上传、在线 YAML 编辑。
- 脱敏开关或假脱敏规则。
- 记录删除、批量删除、导出、趋势图。
- 登录、RBAC、任务队列、重试、WebSocket。
- 为迁就前端而修改后端契约。

若验收需要上述能力，记录缺口并停止扩展；不得创建假按钮、假数据或前端临时协议。

## 2. 执行编排

```text
Agent 1：前端骨架 + 项目管理（串行，先完成）
                 |
                 +--> Agent 2：资产工作台 --------+
                 |                                 |
                 +--> Agent 3：执行与历史 ----------+--> Agent 4：集成、视觉与验收
```

- Agent 1 合并后再启动 Agent 2 和 Agent 3。
- Agent 2、Agent 3 从同一个 Agent 1 基线提交创建独立分支/worktree，可并行。
- Agent 4 在专门集成分支合并 Agent 2、Agent 3；冲突由 Agent 4 按契约解决。
- 每个 Agent 只修改自己声明的目录。发现必须跨所有权修改时，先在交付报告中提出，由 Agent 4 统一处理。
- 新增运行时依赖前必须列出包名、用途和替代方案，得到主线程确认。已确认的技术栈不等于任意第三方包均已批准。

建议分支名：

```text
codex/phase2-4-foundation
codex/phase2-4-assets
codex/phase2-4-executions
codex/phase2-4-integration
```

## 3. Agent 1：前端骨架与项目管理

复制下面整段给 Agent 1：

````markdown
你负责 testplat-control Phase 2.4 的前端骨架和项目管理。请在独立分支/worktree 中完成实现、测试和自审，不要提交 PR，不要推送远端。

## 工作目录与基线

- 仓库：`/Users/jiavin/Developer/products/testplat1/testplat-control`
- 基线：当前 `main`，应包含提交 `93dd99e`
- 新建目录：`frontend/`
- 技术栈：React + TypeScript + Vite + Ant Design，使用 pnpm

开始前执行并记录：

```bash
pwd
git status --short --branch
git rev-parse --short HEAD
node --version
pnpm --version
```

工作区存在用户改动时保留它们；若与本任务重叠且无法安全绕开，停止并报告。

## 先读事实

至少阅读：

- `src/testplat_control/main.py`
- `src/testplat_control/api/projects.py`
- `src/testplat_control/api/schemas.py`
- `tests/test_api.py`
- `../docs/requirements/MVP-需求规格.md` 的 2.1.1、3.1、4.1

从 `app.openapi()` 核对项目端点和响应字段。源码与说明冲突时以当前后端源码和测试为准，并在交付报告中指出冲突。

## 依赖门禁

先给主线程一份拟新增依赖清单，逐项说明用途、运行时/开发时属性和不采用它的影响；得到确认后再安装。不要顺手加入状态库、CSS 框架、图表库、日期库或 mock 平台。

## 文件所有权

你拥有：

- `frontend/` 工程配置、锁文件和 README
- `frontend/src/app/`
- `frontend/src/shared/`
- `frontend/src/features/projects/`
- `frontend/src/pages/ProjectListPage.tsx`
- `frontend/src/pages/ProjectWorkspacePage.tsx` 的占位骨架

不要实现 `features/assets/` 或 `features/executions/`；只为后续集成保留稳定页面容器。

## 交付要求

1. 初始化可复现的 Vite React TypeScript 工程，配置严格 TypeScript、lint、test、build scripts。
2. 使用 Ant Design 和其图标体系。界面面向高频测试操作：克制、紧凑、可扫描；不要做营销首页、大标题 hero、装饰渐变或层层卡片。
3. 建立应用壳、路由、Query Client 和统一 API client。API 基地址可由 `VITE_API_BASE_URL` 配置；本地开发通过 Vite proxy 访问后端，避免要求后端临时放宽 CORS。
4. API client 必须：
   - 正确拼接路径并编码动态 path segment。
   - 解析 FastAPI 的字符串或数组形式 `detail`。
   - 区分 HTTP 错误、网络错误和响应解析错误。
   - 不吞掉未知异常。
5. TypeScript DTO 必须与 `ProjectCreate`、`ProjectUpdate`、`ProjectResponse` 对齐，不在多个 feature 中重复定义。
6. 实现项目列表、创建、编辑、删除：
   - 显示名称、环境、资产目录、更新时间。
   - 不伪造“最后运行时间”，当前项目响应没有该字段。
   - 创建成功后进入项目工作台；编辑成功刷新详情；删除需二次确认。
   - 表单错误贴近字段展示，提交期间防止重复操作。
7. `ProjectWorkspacePage` 只提供页头、返回导航、项目摘要和两个清晰的集成区域：资产区、执行区。不要在这里预写假数据。
8. 所有关键异步视图都有加载、空数据、错误和重试状态；键盘焦点、表单 label、确认弹窗可访问。
9. 为 API client 和项目 CRUD 关键路径写自动化测试。测试通过依赖注入或 mock 请求边界完成，不连接外网和真实生产目录。
10. 代码注释只解释模块责任或非显然决策，使用中文。完成后对整个 diff 做一次中文注释检查。

## 非目标与停止条件

- 不修改后端 Python 文件。
- 不实现资产树、API 执行或执行历史。
- 不引入 Tailwind、Redux、Zustand、ECharts、Monaco 等未批准依赖。
- 若现有后端契约无法完成项目 CRUD，提供最小复现和建议契约，停止相关实现，不自行修改接口。

## 验收

至少完成并报告实际输出：

```bash
cd frontend
pnpm lint
pnpm test --run
pnpm build
```

启动后端和前端，使用真实浏览器完成：项目空态 -> 创建 -> 编辑 -> 删除；检查至少一个桌面视口和一个移动视口，无横向溢出、遮挡或文字截断。保留截图路径。

## 交付格式

最终仅报告：

1. 基线提交与最终提交。
2. 新增依赖及理由。
3. 修改文件与行为摘要。
4. 测试命令、结果和浏览器截图路径。
5. 已知限制、契约缺口。
6. 中文注释检查结论。

先完成自审和验证，再提交一个范围单一的本地 commit。
````

## 4. Agent 2：资产工作台

复制下面整段给 Agent 2：

````markdown
你负责 testplat-control Phase 2.4 的只读资产工作台。请从 Agent 1 已验收的基线提交创建独立分支/worktree，完成实现、测试和自审，不要提交 PR，不要推送远端。

## 开始条件

- `frontend/` 已存在，`pnpm lint`、`pnpm test --run`、`pnpm build` 全部通过。
- 记录 `git status --short --branch` 和 `git rev-parse --short HEAD`。
- 工作区存在用户改动时保留；重叠且无法绕开时停止并报告。

## 先读事实

至少阅读：

- `src/testplat_control/api/assets.py`
- `src/testplat_control/services/asset_service.py`
- `tests/test_assets_api.py`
- `tests/test_asset_service.py`
- `frontend/src/shared/` 中 Agent 1 建立的 API client 和 DTO 约定
- `../docs/requirements/MVP-需求规格.md` 的 2.1.1、4.1

使用真实 fixture 或后端测试确认资产节点的 `id/name/type/path/children/metadata`。不要假定 metadata 一定包含 method、url 或 description；所有 metadata 都必须容错展示。

## 文件所有权

你拥有：

- `frontend/src/features/assets/`
- 对应测试文件

不要修改 app 路由、项目 feature、执行 feature 或 `ProjectWorkspacePage.tsx`。通过导出的组件和类型交给 Agent 4 集成。

## 组件契约

导出一个稳定的资产浏览入口，至少支持：

```ts
type AssetBrowserProps = {
  projectId: string;
  selectedAssetId?: string;
  onSelectAsset: (asset: AssetNode) => void;
};
```

若需要调整该契约，先报告原因，等待主线程确认。

## 交付要求

1. 调用 `GET /projects/{project_id}/assets` 展示可展开/收起的只读树。
2. 目录和文件使用 Ant Design 图标区分；选中态、hover、键盘操作和窄屏交互清晰。
3. 支持本地过滤：按名称、路径或资产 ID 搜索，同时保留命中节点的祖先路径。不要为本地数据增加不存在的搜索 API。
4. 选中文件时调用资产详情端点，并展示后端真实返回的元信息；目录不得请求参数或运行端点。
5. 明确区分可运行的 `api.*` 文件与当前不可运行的其他文件：不可运行项可以查看元信息，但不显示可点击的运行按钮。
6. 支持加载、空树、404、扫描失败、重试状态。项目切换时清理旧选择，不能短暂显示上一个项目的数据。
7. 不实现上传、编辑、删除或刷新磁盘之外的写操作。
8. 为树转换、搜索保留祖先、选择事件、错误态和项目切换写自动化测试。
9. 复用 Agent 1 的设计 token、API client、错误模型和共享 DTO；不要复制一套平行基础设施。
10. 注释只解释非显然的树过滤或状态同步逻辑，使用中文。完成后检查整个 diff 的中文注释。

## 停止条件

- 需要更改后端响应、资产 ID 规则或 URL 路由。
- 需要新增未批准运行时依赖。
- Agent 1 的共享契约不足且无法通过 feature 内适配解决。

命中停止条件时给出最小证据和建议，不自行跨边界修改。

## 验收

```bash
cd frontend
pnpm lint
pnpm test --run
pnpm build
```

使用真实浏览器验证：树展开/收起、搜索、清空搜索、文件选择、目录选择、错误重试、桌面和移动视口。保留截图路径。

## 交付格式

最终仅报告：基线/最终提交、修改文件、组件导出契约、测试结果、截图路径、已知限制、中文注释检查结论。验证通过后提交一个本地 commit。
````

## 5. Agent 3：单 API 执行与历史

复制下面整段给 Agent 3：

````markdown
你负责 testplat-control Phase 2.4 的单 API 执行、执行历史和执行详情。请从 Agent 1 已验收的同一基线提交创建独立分支/worktree，完成实现、测试和自审，不要提交 PR，不要推送远端。

## 开始条件

- `frontend/` 已存在，基础 lint/test/build 全绿。
- 记录 `git status --short --branch` 和 `git rev-parse --short HEAD`。
- 工作区存在用户改动时保留；重叠且无法绕开时停止并报告。

## 先读事实

至少阅读：

- `src/testplat_control/api/run.py`
- `src/testplat_control/api/execution.py`
- `src/testplat_control/api/schemas.py`
- `tests/test_run_api.py`
- `tests/test_execution_api.py`
- `tests/integration/test_api_execution_flow.py`
- `frontend/src/shared/` 中 Agent 1 建立的 API client 和 DTO 约定

后端测试是交互语义的权威，尤其注意：执行结果 `failed` 仍返回 HTTP 200；422 不产生记录；详情中的 `request` 或 `response` 可以为 null；列表没有 total。

## 文件所有权

你拥有：

- `frontend/src/features/executions/`
- 对应测试文件

不要修改 app 路由、项目 feature、资产 feature 或 `ProjectWorkspacePage.tsx`。通过导出组件交给 Agent 4 集成。

## 组件契约

至少导出：

```ts
type ApiExecutionPanelProps = {
  projectId: string;
  assetId: string;
};

type ExecutionHistoryProps = {
  projectId: string;
  assetId?: string;
};
```

若需要调整契约，先报告原因，等待主线程确认。

## 交付要求

1. 参数表单：
   - 从 `/params` 动态渲染 string、number、boolean。
   - 必填字段在前端提供即时校验，但以后端 422 为最终事实。
   - 空的非必填 string/number 不进入请求；`false` 是合法 boolean 值，不能因 falsy 被遗漏。
   - 切换 asset 时清空旧参数、旧错误和旧结果，避免跨资产污染。
2. 执行：
   - 请求体严格为 `{ "params": { ... } }`。
   - 提交中禁用重复执行并给出进度反馈。
   - HTTP 200 + `status=failed` 作为可查看的执行结果，不进入网络错误态。
   - success 和 failed 都使当前项目/资产的历史查询失效并刷新。
3. 结果：
   - 清晰展示状态、耗时、record ID，以及存在时的 request、response、error 和原始 result。
   - 使用可滚动、可复制的 JSON 视图；未知结构必须容错，不能渲染崩溃。
   - 当前没有脱敏配置契约，不实现假脱敏开关；在交付限制中明确记录。
4. 历史：
   - 支持 status、asset_id 筛选和后端允许的排序。
   - 列表没有 total；使用上一页/下一页或“加载更多”时，以返回条数小于 limit 判断末页，不能伪造总页数。
   - 点击记录加载详情；详情的 request/response/result/null/parse_error 都需可展示。
5. 异常：展示 FastAPI `detail`；处理 404、422、500、网络失败和 malformed response，不暴露空白页。
6. 为以下路径写自动化测试：字段类型转换、可选空值省略、false 保留、422、业务 failed、success、查询刷新、列表末页、null request/response、资产切换清理。
7. 复用 Agent 1 的共享 client、错误模型、Query Client 和设计 token。不要再建 Axios/fetch 包装器或重复 DTO。
8. 注释只解释状态隔离、请求序列化等非显然决策，使用中文。完成后检查整个 diff 的中文注释。

## 非目标与停止条件

- 不实现 case 执行、实时进度、重试、删除记录、导出、趋势图或脱敏规则。
- 不修改后端 Python 文件。
- 新增运行时依赖、改变共享契约或发现后端行为与测试冲突时，提供证据并停止相关扩展。

## 验收

```bash
cd frontend
pnpm lint
pnpm test --run
pnpm build
```

使用真实浏览器覆盖：无参数 API、有 string/number/boolean 参数 API、422、业务 failed、业务 success、历史筛选、详情展开、移动视口。保留截图路径；无法稳定触发的状态用确定性测试证明并说明。

## 交付格式

最终仅报告：基线/最终提交、修改文件、组件导出契约、测试结果、截图路径、已知限制、中文注释检查结论。验证通过后提交一个本地 commit。
````

## 6. Agent 4：集成、视觉与端到端验收

复制下面整段给 Agent 4：

````markdown
你负责 testplat-control Phase 2.4 的最终集成和验收。请在独立集成分支/worktree 中工作，不要推送远端，不要创建 PR。

## 输入

- Agent 1 已验收的前端基础提交。
- Agent 2 的资产工作台提交。
- Agent 3 的执行与历史提交。
- Phase 2.3 后端基线提交 `93dd99e`。

先记录各输入 commit、当前 `git status --short --branch`。如果输入提交不是基于相同的 Agent 1 基线，先说明合并拓扑，再进行可追溯的合并。

## 先读事实

重新核对：

- FastAPI `app.openapi()` 的当前路径和 schemas。
- `tests/integration/test_api_execution_flow.py` 的完整流程。
- 三个 Agent 的交付报告和完整 diff。
- `../docs/requirements/MVP-需求规格.md` 的项目管理、单接口运行、结果展示要求。

不要依据旧提示词或口头总结覆盖当前代码事实。

## 集成职责

1. 合并 Agent 2、Agent 3 的提交。保留各自 feature 边界；只在确有需要时修改共享层。
2. 在 `ProjectWorkspacePage` 完成组合：
   - 项目摘要和返回导航。
   - 左侧/抽屉式资产浏览。
   - 中央资产元信息与 API 执行。
   - 同页或清晰 tab 中的执行历史与详情。
3. 只有选中的 `api.*` 文件可显示执行面板。目录和其他文件显示只读信息及当前能力边界。
4. 校正 query key 与失效策略，保证：
   - 项目切换不串数据。
   - 资产切换不串参数和结果。
   - success/failed 执行后历史都刷新。
   - 返回列表再进入时没有明显陈旧状态。
5. 做一致的错误处理、加载骨架、空态、禁用态和焦点管理。去掉无功能按钮和占位入口。
6. 做一次视觉审查：这是测试运营工具，应紧凑、安静、适合扫描；使用 Ant Design 图标；避免营销式大标题、浮动装饰、嵌套卡片和单一蓝紫色铺满页面。
7. 检查桌面和移动布局。稳定约束树区、工具栏、表格列和 JSON 视图尺寸，不能因 loading、长 ID、长路径或长 JSON 造成布局跳动、遮挡或不可恢复的横向溢出。
8. 补充最少但足够的集成测试，覆盖从项目进入工作台、选择 API、取参数、运行、刷新历史、打开详情的前端链路。
9. 除非属于已批准依赖且确有必要，不新增包；若必须新增，先停止并报告。
10. 不修改后端语义。若联调暴露契约缺口，提交复现步骤、期望/实际结果和建议修复位置，由主线程决定是否另开后端任务。

## 完整验收

后端：

```bash
uv run pytest -q
```

前端：

```bash
cd frontend
pnpm lint
pnpm test --run
pnpm build
```

联调：

1. 启动 FastAPI 和 Vite，记录命令、端口和日志位置。
2. 使用指向仓库 fixture 的测试项目，不使用个人或生产资产目录。
3. 用真实浏览器走通：创建项目 -> 进入项目 -> 浏览/搜索资产 -> 选择 API -> 填参数 -> 执行 -> 查看结果 -> 筛选历史 -> 打开详情 -> 编辑项目 -> 删除测试项目。
4. 至少检查 `1440x900` 和 `390x844` 两个视口并截图。
5. 检查浏览器 console、失败请求、页面溢出和不可见交互；截图之外还要验证关键 DOM 文本和状态。

## 最终审查

- 对 `git diff <Agent-1基线>...HEAD` 做完整自审，不只看冲突文件。
- 核对 Phase 2.4 范围中的每项均有代码和验证证据。
- 核对非目标没有以占位按钮、假数据、硬编码响应混入。
- 所有新增/修改代码中的说明性注释使用中文，并且只保留有决策价值的注释。
- 不把 lint/test/build 通过等同于业务验收；必须报告浏览器联调结果。

## 交付格式

最终报告包含：

1. 合并的 commit 和最终 commit。
2. 用户可见功能清单。
3. 前后端测试、构建、浏览器联调的命令与实际结果。
4. 桌面/移动截图绝对路径。
5. 已知限制和单列的后端契约缺口。
6. 依赖审计与中文注释检查结论。

全部验收通过后提交一个本地集成 commit；不要推送或创建 PR。
````

## 7. 主线程操作顺序

1. 审批 Agent 1 的依赖清单；未审批前不进入安装。
2. 执行 Agent 1，检查其 diff、测试、截图和本地 commit。
3. 以 Agent 1 commit 为共同基线，并行执行 Agent 2、Agent 3。
4. 分别检查 Agent 2、Agent 3 的范围、测试和组件导出契约。
5. 执行 Agent 4 合并与联调；任何后端契约变化另开任务，不混入 UI commit。
6. 主线程最终独立复跑后端测试、前端 lint/test/build，并抽查浏览器关键流程。
7. 通过后再决定是否推送、创建 PR；本文没有授权这些外部操作。

## 8. 最终验收矩阵

| 能力 | 自动化证据 | 浏览器证据 |
|---|---|---|
| 项目 CRUD | API client/表单/缓存测试 | 创建、编辑、删除 |
| 资产树 | 树转换、搜索、切换测试 | 展开、搜索、选择 |
| 参数表单 | 类型、必填、空值序列化测试 | 多类型输入 |
| 单 API 执行 | success/failed/422 测试 | 运行并看结果 |
| 执行历史 | 筛选、分页边界、详情测试 | 筛选并打开详情 |
| 状态隔离 | 项目/资产切换测试 | 切换后无旧数据闪现 |
| 响应式 | 组件边界测试（适用时） | `1440x900`、`390x844` 截图 |
| 回归 | `uv run pytest -q` + 前端全门禁 | console 与网络请求无未解释错误 |
