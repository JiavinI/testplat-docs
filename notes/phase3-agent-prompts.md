# Phase 3 Agent 提示词：串行 API Case 核心

日期：2026-10-12
目标：在 Phase 2.5 的真实 Control–Engine 集成基础上，交付可从 Web 运行、可查看步骤级结果、可进入历史详情的串行 API Case；同时固定未来 UI、DB、Redis、自定义步骤的扩展 seam。

> 本文是执行手册，不是实现代码。第一批只交付 API 步骤；鉴权/凭证、实时进度传输和 UI 步骤各自经过后续设计门禁后实施。所有 Agent 只提交本地 commit，不推送、不建 PR、不合并。

## 0. 已批准范围与裁决

### 0.1 已验证基线

- Engine：远程 `main @ 2dba73e`，`v0.1.0` 指向 `942a66dd127de3647deab88611ada2a9bcc06c90`。
- Control：`main @ da5e26fd43a5257056190ec25c8aee59ec198e97`，依赖锁定 Engine `942a66d`。
- Phase 2.5 终验：Engine 118 tests；Control 123 tests；前端 66 tests、lint、build、e2e 门禁通过。
- 本地 Engine 的 `main` 分支可能落后于 `origin/main`；新 worktree 必须从已核验的 `origin/main`/精确 SHA 创建，不得从旧本地 `main` 猜基线。

Agent 启动时仍须重新记录 remote、基线 SHA 和工作区状态；上述数字是规划证据，不代替本次复跑。

### 0.2 需求冲突裁决

`docs/requirements/MVP-需求规格.md` 把 UI 自动化列为 MVP 后能力；`docs/requirements/补充需求.md` 又把 API+UI 混合场景提升为 P0。两者按以下方式同时落实：

1. Phase 3 第一批建立可注册的 `StepExecutor` seam。
2. 第一批只注册 `api` executor，并完成串行 API Case 的端到端交付。
3. UI、DB、Redis、custom 是受控扩展点，不是第一批可用能力。
4. UI executor 后续单独设计浏览器生命周期、隔离、资源上限、截图/制品和依赖，再接入同一 seam。

不得复制旧系统的业务模型。`/Users/jiavin/Developer/products/testplat-new` 只作为只读需求证据和验收样例来源。

### 0.3 第一批业务语义

- Case 按资产声明顺序串行执行；MVP 不并发。
- 默认 fail-fast：一个步骤失败后，后续未执行步骤以 `skipped` 出现在结果中。
- 断言不满足属于业务失败：HTTP 200，Case `status=failed`，并保存完整执行记录。
- Case 已开始后发生 timeout、transport 或 Engine internal error：公开异常携带安全、可序列化的 `partial_outcome`（已完成步骤、当前失败步骤、后续 `skipped`）；Control 用它保存 `failed` 记录，并按批准的系统错误映射返回 5xx。异常与 partial outcome 不得表达互相冲突的状态。
- 资产不存在、Case 结构无效、环境无效等 preflight 错误不创建执行记录。
- 每个步骤有稳定 `step_id`；步骤输出只能通过显式 alias 进入后续上下文。禁止依赖步骤数组下标形成跨步骤引用。
- 变量解析失败必须指出步骤和变量路径，不得降级为空字符串。
- 公开结果、日志和持久化记录不得包含 Authorization、Cookie、token、password 等秘密。
- 第一批没有 retry、cleanup 补偿、条件分支、循环、队列、后台任务或实时推送。遇到相应 DSL 必须报 `unsupported`/验证错误，不得静默忽略。

### 0.4 Interface 与版本裁决

- Engine 发布线进入 `0.2.x`；Case 契约不得伪装成 `0.1.x` 的内部修改。
- 保留唯一公开执行入口 `execute(request)`。实现可用 overload/判别联合同时接受 `ApiExecutionRequest` 与 `CaseExecutionRequest`，并返回对应 outcome；现有单 API 调用行为保持兼容。
- Case 公共 DTO 至少表达：Case 身份、步骤声明、步骤顺序、输入环境/参数、步骤状态、步骤耗时、请求/响应、安全错误、输出 alias 和 Case 汇总；execution-stage 的公开异常必须能携带同结构的 `partial_outcome`。
- `StepExecutor` 是 Engine 内部端口；Control、FastAPI、SQLAlchemy、WebSocket 类型不得进入 Engine Interface。
- 进度先形成 Engine 内部领域事件语义，但第一批不公开回调或传输 API。事件协议要到 Case 结果稳定后另行批准。

## 1. 目标架构

```text
Control Case route
  -> CaseExecutionService              # 环境、计时、记录、HTTP 映射
     -> EnginePort.execute(request)     # 唯一跨仓 seam，Engine 0.2.x
        -> CaseRunner                   # 顺序、上下文、fail-fast、汇总
           -> StepExecutorRegistry
              -> ApiStepExecutor        # 第一批唯一注册实现
                 -> 已有单 API执行 Module

未来扩展（本批不实现）
  StepExecutorRegistry -> UiStepExecutor / DbStepExecutor / RedisStepExecutor
  CaseRunner events -> Control run lifecycle -> WebSocket 或 SSE
  ExecutionContext -> 已解析且可脱敏的身份材料
```

所有权：

| 边界 | 所有者 | 本批责任 |
|---|---|---|
| Case 资产语法与运行语义 | Engine | 解析、校验、执行、步骤/Case outcome |
| 项目路径、环境选择、HTTP、记录 | Control | 加载项目环境、调用 Engine、持久化、映射响应 |
| Case 展示与交互 | Control frontend | 元信息、运行、步骤结果、历史详情 |
| 凭证原文 | 后续 Control 配置/秘密边界 | 本批不加载、不持久化、不展示 |
| 实时传输 | 后续 Control run lifecycle | 本批不引入 WebSocket/SSE |

## 2. 执行顺序与硬门禁

```text
Agent 0：Case 语法/现状只读审计
  -> 主线程批准最小 Case contract 与 golden fixtures
  -> Agent 1：Engine 0.2 Interface + Case 契约（串行基础）
  -> 主线程独立双轴审查 Interface
  -> Agent 2：Engine CaseRunner + API executor ---------+
  -> Agent 3：Control Case backend ---------------------+ -> Agent 5：跨仓集成
     （Agent 3 可在 Interface 稳定后与 Agent 2 并行）    |
  -> Agent 4：Control Case frontend（基于稳定 OpenAPI）--+
  -> 主线程终验与发布决策
```

硬门禁：

1. Agent 0 未交付、最小 Case contract 未经主线程确认时，Agent 1–5 停止。
2. Agent 1 未通过独立 Spec/Standards 审查时，Agent 2–4 不开始。
3. Agent 3 未冻结 OpenAPI/示例响应时，Agent 4 不开始。
4. 任一 Agent 发现需要新增运行时依赖、数据库迁移、秘密存储、跨仓协议改向或 DSL 扩张，停止并报告，不自行决定。
5. 每个仓库独立分支、独立 commit；Agent 不修改其他 Agent 的 worktree。
6. 每个实现 Agent 对完整 diff 做中文 comment pass：注释解释边界、不变量和原因，不逐行翻译代码。

建议分支：

```text
codex/phase3-case-contract
codex/phase3-engine-case-runtime
codex/phase3-control-case-backend
codex/phase3-control-case-frontend
codex/phase3-case-integration
```

## 3. Agent 0：Case 契约只读审计

复制下面整段给 Agent 0：

````markdown
你负责 Phase 3 Case 契约启动前审计。只读：不改文件、不创建分支、不暂存、不提交。

## 权威范围

- `/Users/jiavin/Developer/products/testplat1/docs/requirements/MVP-需求规格.md`
- `/Users/jiavin/Developer/products/testplat1/docs/requirements/补充需求.md`
- `/Users/jiavin/Developer/products/testplat1/docs/requirements/需求汇总.md`
- `/Users/jiavin/Developer/products/testplat1/docs/architecture/control-engine-interface.md`
- Engine `origin/main @ 2dba73e` 与 `v0.1.0 @ 942a66d`
- Control `main @ da5e26f`
- Control 现有 Case fixtures/parser/tests
- `/Users/jiavin/Developer/products/testplat-new/assets/scenarios/` 下 Case 仅作只读证据

## 任务

1. 记录两个仓库的 remote、HEAD、dirty 状态、worktree 和相关版本。
2. 列出三份需求文档、架构文档、现有 Control parser/fixture、旧 Case 示例之间所有字段与语义差异。
3. 从旧示例中区分：第一批必需、未来扩展、业务专属；不得建议复制业务专属字段。
4. 提议一个最小 API Case schema，至少覆盖稳定 `step_id`、`type=api`、API 资产引用、params、显式 output alias、Case 级 params/metadata。
5. 为 success、步骤断言失败、变量传播、非法引用、unsupported step、timeout/transport 各提议一个 golden fixture 和预期结果。
6. 明确 JSON/YAML 形状、重复 step_id、alias 冲突、前向引用、未知字段和循环引用的处理。
7. 审计现有 Control Case parser 是否适合复用；逐项给出保留、迁移到 Engine、删除/废弃建议。
8. 列出所有需要主线程裁决的真正歧义，不把已批准的 API-only、串行、fail-fast 决策重新开放。

## 交付

只交付证据矩阵、最小 schema 草案、golden fixture 清单、迁移建议和阻塞问题。引用精确文件与行号；不写实现。
````

Agent 0 已完成。主线程对 A1–A8、schema、引用语义、结果不变量及 G1–G6 的正式裁决见 `notes/phase3-case-contract-v0.2.md`；后续 Agent 以裁决稿为权威，审计报告中的“提议”不再作为可选项。

## 4. Agent 1：Engine 0.2 Interface 与 Case 契约

复制下面整段给 Agent 1：

````markdown
你负责 Engine 0.2 的 Case 公共 Interface、Case 资产模型/校验和契约测试。必须从主线程指定的 Engine `origin/main` 精确 SHA 建独立 worktree。

## 开始前

- 记录基线 SHA、工作区、Python/uv 版本。
- 完整阅读 `notes/phase3-case-contract-v0.2.md`、现有 `testplat_engine.interface`、单 API DTO、公开异常。
- 若批准后的 schema/fixture 未提供，停止。

## 文件所有权

你拥有 Engine 公共 DTO/导出、Case domain model、Case loader/validator、版本元数据和对应契约测试。你不实现步骤网络执行，不修改 Control。

## 交付要求

1. 保持唯一 `execute(request)` seam；为 API/Case 提供明确的静态类型关系，现有单 API 调用继续工作。
2. 定义冻结、可 JSON 序列化的 `CaseExecutionRequest`、`CaseExecutionOutcome`、`StepOutcome`。
3. 固定状态集合和不变量：步骤 `success|failed|skipped`；Case `success|failed`；顺序与输入一致；每个声明步骤恰有一个 outcome。
4. 精确实现裁决稿 schema 和严格校验；不得重新引入 `metadata`、旧键别名或 assert DSL。
5. `skipped` 步骤不得伪造 request/response；失败步骤保留安全错误和可用的执行证据。
6. Case outcome 能表达 Case 汇总、步骤耗时、总耗时和 alias 输出，但不得包含秘密；execution-stage 异常携带安全 `partial_outcome`，preflight 异常不伪造执行结果。
7. 建立内部 `StepExecutor`/registry 端口，接口只携带执行上下文和步骤定义，不携带 FastAPI/ORM/WebSocket。此 Agent 只提供端口与契约测试。
8. 更新版本到批准的 `0.2.0` 开发版本；记录兼容说明，不创建 tag。
9. 有价值的中文原因注释；完成完整 diff 的 comment pass。

## 契约测试

- 现有 Api request/outcome 与异常测试全部保留。
- Case DTO JSON round-trip。
- execution-stage 异常的 `partial_outcome` round-trip，以及异常/结果状态一致性。
- 裁决稿 G1–G6、M1–M8 的加载/拒绝契约；运行期预期可先形成明确的 pending handoff，但不得用伪实现制造通过。
- 步骤顺序、稳定 ID、状态不变量和安全序列化。
- registry 未知 type 的确定性错误。
- mutation：删除必填字段、重复 ID、篡改 alias、加入未知类型时测试必须失败。

## 验收

```bash
uv run ruff check .
uv run pytest -q
uv build
```

自审后提交一个范围单一的本地 commit。不要推送、不要建 PR。交付基线/HEAD、diff 摘要、完整门禁结果、兼容说明和 comment-pass 结论。
````

## 5. Agent 2：Engine CaseRunner 与 API StepExecutor

复制下面整段给 Agent 2：

````markdown
你负责 Engine 的串行 CaseRunner 和第一种 `api` StepExecutor。必须从已通过独立审查的 Agent 1 commit 开始。

## 范围

- 串行步骤调度。
- Case 参数、环境和显式 alias 的执行上下文。
- API StepExecutor 复用现有单 API执行 Module。
- fail-fast、skipped 补齐、Case 汇总。
- 执行中系统错误的步骤归属、安全结果和资源清理。

## 约束

1. CaseRunner 编排步骤，不重新实现 HTTP、断言、变量 resolver。
2. registry 只注册 `api`；其他类型由契约层确定性拒绝。
3. 上下文只能写入步骤声明的 alias；写入值取自批准的安全输出模型。
4. 引用在执行步骤前解析；缺失引用产生带 `step_id`/路径的稳定失败，不降级。
5. 业务断言失败触发 fail-fast，后续步骤生成 `skipped`；Case 返回业务 `failed` outcome。
6. timeout/transport/internal 保持现有 `kind + stage`，关联当前步骤并携带安全 `partial_outcome`；不得吞成断言失败。
7. 运行结果不泄露 Authorization、Cookie、token、password；补充嵌套结构脱敏测试。
8. 不实现回调、WebSocket、SSE、队列、重试、cleanup、UI、DB、Redis 或 auth manager。
9. 不访问公网；测试使用本地 HTTP server 或可控 client。
10. 中文注释解释上下文所有权、fail-fast 和异常边界；完成 comment pass。

## 测试矩阵

- 单步/多步 success。
- alias 从前一步传播到后一步。
- 断言失败 + 后续 skipped。
- 首步/中间步 timeout、transport、internal。
- 引用缺失、结果形状不匹配、executor 未注册。
- 单 API 既有回归完整通过。
- mutation：交换步骤、移除 fail-fast、绕开 alias、关闭脱敏时至少一项测试准确失败。

## 验收

```bash
uv run ruff check .
uv run pytest -q
uv build
```

提交一个本地 commit，不推送、不建 PR。报告每个 golden fixture 的实际结果。
````

## 6. Agent 3：Control Case backend

复制下面整段给 Agent 3：

````markdown
你负责 Control 的 Case 资产详情、运行入口、Engine 0.2 适配、执行记录和 OpenAPI。必须从 Control `main @ da5e26f` 的后继精确基线建立独立 worktree，并使用已审查的 Agent 1 Interface。

## 先读

- `src/testplat_control/services/asset_service.py`
- `src/testplat_control/services/execution_service.py`
- `src/testplat_control/ports/engine.py`
- `src/testplat_control/api/assets.py`、`api/run.py`、`api/execution.py`、`api/schemas.py`
- `src/testplat_control/database/models.py`
- 现有 Case parser/fixtures/tests

## 交付要求

1. Control 只负责路径/项目/环境、调用、计时、持久化和 HTTP；Case 语法校验权威归 Engine。
2. 资产 API 能返回 Case 元信息与规范化步骤摘要，但不得在 Control 维护第二套 Case 语义解析器。
3. 提供 Case run endpoint，通过既有 EnginePort 的唯一 `execute(request)` 调用 Case。
4. 复用 `ExecutionRecord(asset_type="case")`；若无需 schema migration，不得制造迁移。确需迁移时立即停止报告。
5. 业务 failed=HTTP 200 并保存；preflight 不保存；execution 系统错误保存 failed，沿用 Phase 2.5 错误映射。
6. `result_json` 保存完整 Case/step outcome；历史详情能稳定返回步骤数据且保留旧 API 详情兼容。
7. OpenAPI 为 Case 详情、run response 和历史 Case result 提供明确 schema，不用无约束 `dict` 掩盖契约。
8. Engine 依赖锁定到主线程批准的 Engine 0.2 commit；不得用本地 path editable 作为正式依赖。
9. 不引入后台任务、WebSocket/SSE、AuthManager 或新秘密字段。
10. 中文注释解释解析所有权、记录生命周期和错误映射；完成 comment pass。

## 测试

- Case 资产详情与非法资产。
- success、business failed、preflight error、execution timeout/transport/internal。
- 每种情况精确验证 HTTP、是否建记录、record 状态和 step result。
- API/Case 历史筛选、分页、排序、详情兼容。
- OpenAPI snapshot/结构断言：既有 Phase 2.5 API 不漂移。
- Fake 与真实 Engine DTO 使用同一契约。

## 验收

```bash
uv run ruff check .
uv run pytest -q -rs
cd frontend
pnpm lint
pnpm test --run
pnpm build
```

提交一个本地 commit，不推送、不建 PR。交付 OpenAPI diff 和示例 JSON，供 Agent 4 固定前端契约。
````

## 7. Agent 4：Control Case frontend

复制下面整段给 Agent 4：

````markdown
你负责 Phase 3 第一批的 Case 前端体验。基线必须包含已审查的 Agent 3 OpenAPI/示例响应；若后端契约仍在变化，停止。

## 产品范围

- 资产树中选择 Case 后展示 metadata、tags 和步骤摘要。
- Case 可执行；API 资产继续使用现有动态参数表单。
- 执行完成后展示 Case 汇总、每步状态/耗时，步骤可展开查看 request/response/error。
- 历史列表和详情支持 `asset_type=case`，旧 API 记录不回退。

## 交互规则

1. 沿用用户已选的 P2 sage-green/editorial 主题，不重做视觉体系。
2. 状态文案和颜色同时表达结果，不能只依赖颜色；`skipped` 与 `failed` 明确区分。
3. 默认突出首个失败步骤；展开详情保持用户可控，不一次渲染所有大 JSON。
4. 系统错误展示安全的 `public_message` 和运行 ID；不展示日志原因、栈或秘密。
5. 同步执行期间提供明确 loading/禁用态；本批不伪造步骤实时进度。
6. 390x844 不横向溢出；长 step_id、URL、JSON 可换行/局部滚动。
7. 1440x900 保持工作台信息密度；资产过滤/选中、项目 CRUD、API 执行和历史全部回归。
8. 不添加 WebSocket client、轮询、Auth UI、Case 编辑器或 UI step 控件。
9. TypeScript 类型来自已冻结契约；不要以 `any` 绕过 Case outcome。
10. 代码中的中文注释只解释复杂映射/响应式原因；完成 comment pass。

## 测试与视觉证据

- Case metadata 和步骤摘要。
- success、failed+skipped、system error、malformed response。
- 历史 Case 详情和旧 API 详情。
- loading、防重复提交、错误恢复。
- 390x844 与 1440x900 两组截图，存入 `frontend/screenshots/phase3/`，命名区分 case-success/case-failed/history。

## 验收

```bash
pnpm lint
pnpm test --run
pnpm build
pnpm e2e:gate
```

提交一个本地 commit，不推送、不建 PR。报告测试数、截图路径、浏览器控制台/网络错误和 comment-pass 结论。
````

## 8. Agent 5：跨仓集成候选

复制下面整段给 Agent 5：

````markdown
你负责 Phase 3 第一批的跨仓集成候选与验收，只修复已批准 Case 契约内问题。

## 输入

- 已审查的 Agent 1 Engine Interface commit。
- Agent 2 Engine runtime commit。
- Agent 3 Control backend commit。
- Agent 4 frontend commit。
- Agent 0 的 golden fixtures。

## 集成步骤

1. Engine/Control 分别建集成分支，保留独立历史，记录全部 cherry-pick/merge 来源。
2. Control 锁定 Engine 0.2 候选的远程完整 SHA；干净 venv 安装不得依赖 sibling checkout。
3. 运行 Engine golden fixtures 和真实本地 HTTP target：多步 success、alias、assertion failed、timeout、transport。
4. 运行浏览器 -> Control -> Engine -> 本地 target -> SQLite history -> history detail 全链路。
5. mutation 验收：重复 step_id、未知 type、缺失 alias、前向引用、移除 fail-fast、返回嵌套秘密；验证精确拒绝/脱敏。
6. 编译比较 OpenAPI：Phase 2.5 既有端点与响应不漂移；Phase 3 新结构有明确 schema。
7. 复跑 API 单执行、项目 CRUD、资产树、历史分页/排序以及 390/1440 浏览器门禁。
8. 干净 worktree/venv 按 README 安装，证明 Engine 0.2 来源与版本；不得借用开发机 editable path。
9. 更新两个仓库 README/兼容说明，只写已实现能力。
10. 对两个仓库最终完整 diff 做中文 comment pass。

## 停止条件

- Case schema 或跨仓 DTO 需要改变：停止并回到 Interface 审查。
- 需要数据库迁移、新运行时依赖、秘密存储、后台运行或实时传输：停止并请主线程裁决。
- UI/DB/Redis/custom step 被测试资产实际要求：作为下一批缺口记录，不在集成阶段扩张。

## 最终门禁

```bash
# Engine
uv run ruff check .
uv run pytest -q
uv build

# Control
uv sync --extra dev --locked
uv run ruff check .
uv run pytest -q -rs
cd frontend
pnpm install --frozen-lockfile
pnpm lint
pnpm test --run
pnpm build
pnpm e2e:gate
```

## 交付格式

1. 两仓基线、分支、HEAD、工作区状态。
2. Engine 版本和 Control 完整 SHA 锁定。
3. Case schema/状态/错误语义摘要。
4. 单测、真实 Engine、mutation、浏览器、clean-machine、OpenAPI 证据。
5. 截图路径和响应式结论。
6. 已知限制，明确区分 auth、progress transport、UI step。
7. 两仓中文 comment-pass 结论。

不要推送、不要建 PR、不要打 tag；由主线程终验后决定。
````

## 9. 主线程质量审查

Agent 交付后，主线程按顺序执行：

1. 审查 Agent 0 证据和最小 schema；先固定 golden fixtures。
2. 对 Agent 1 做独立 Spec/Standards 双轴审查，重点检查 Interface 深度、状态不变量、兼容和扩展 seam。
3. 分别审查 Agent 2/3/4；只对高风险修订重复完整双轴审查。
4. 亲自比较 Engine/Control 完整 diff、依赖锁、OpenAPI 和生成物。
5. 复跑两仓门禁及真实本地 HTTP Case 矩阵，不只接受 Agent 报告。
6. 亲自做 malformed/mutation、敏感信息和历史记录语义检查。
7. 检查 P2 主题、390/1440 截图、浏览器 console/network 和 Phase 2.5 回归。
8. 对最终完整 diff 做中文 comment pass。
9. 向用户报告阻塞问题；只有用户明确授权后才推送、建 PR、打 tag 或合并。

## 10. 后续批次（本手册不授权实现）

### Phase 3.3：鉴权与凭证

在 Case core 稳定后单独设计。Control 拥有配置/秘密来源和生命周期；Engine 只接收已解析、最小化、可脱敏的执行身份材料。第一批策略建议从静态 Bearer/header 开始。秘密存储、登录策略、缓存、401 重登均需安全审查和用户批准。

### Phase 3.4：进度事件与传输

先冻结 `case_started/step_started/step_finished/case_finished` 领域事件，再设计 Control 的 run lifecycle、断线重连与最终状态权威。WebSocket 与 SSE 必须在真实交互需求和后台执行模型明确后选择；Engine 不依赖 FastAPI/WebSocket。

### Phase 3.5：UI StepExecutor

单独批准 Playwright 依赖、浏览器池/隔离、超时、制品、截图脱敏和资源上限。通过既有 registry 接入，不修改 CaseRunner 的核心顺序/上下文语义。旧 `testplat-new` Case 只用作验收样例。
