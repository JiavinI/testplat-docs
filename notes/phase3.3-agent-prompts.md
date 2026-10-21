# Phase 3.3a Agent 提示词：静态 header 凭证

日期：2026-10-12
权威设计：`notes/phase3.3-auth-design-draft.md`（已批准，D1–D6 全部按推荐）
目标：API 请求（单 API 运行和 Case 中的 api 步骤）可以按资产声明的身份携带静态 header 凭证，秘密不进入任何公开面；接口为 3.3b（登录换 token、Cookie 会话）预留判别式扩展点。

> 所有 Agent 只提交本地 commit，不推送、不建 PR、不打 tag、不合并。3.3b 不在本手册授权范围内。

## 0. 基线与已定裁决

### 0.1 基线

- Engine：`origin/main @ 4715c61be52ce4e071bde5114a57fbbcad959bf6`，tag `v0.2.0`。本地 `main` 分支已知落后，**必须从 `origin/main` 或精确 SHA 建 worktree**。
- Control：`origin/main @ c5a904b`（PR #36 合并后），依赖锁定 Engine `4715c61`。
- 终验数字（规划证据，不代替复跑）：Engine 271 tests；Control 219 tests；前端 81 tests、lint、build、`e2e:gate`、真实 Engine 矩阵通过。

### 0.2 已批准决策（不得重新开放）

| # | 裁决 |
|---|---|
| D1 | 只做静态 header 凭证；登录、Cookie 会话、OAuth2、token 缓存、401 重登全部不做 |
| D2 | 秘密只来自 Control 服务进程的环境变量；变量名必须匹配 `^TESTPLAT_[A-Z0-9_]+$` |
| D3 | API 资产 `auth: <身份>` / `auth: none`；环境级 `default_identity`；Case 步骤不能覆盖 |
| D4 | 身份/秘密相关错误一律复用 `invalid_environment`（preflight，422，不保存记录） |
| D5 | 凭证 header 与资产 header 同名时凭证优先，公开结果记录被覆盖的 header 名 |
| D6 | Engine 发布 `0.3.0`；Control 约束改为 `testplat-engine>=0.3.0,<0.4` |

### 0.3 全体硬约束

1. 秘密值不得出现在：DTO 的 `repr`/`str`/`to_dict`、公开结果、错误 `public_message`/`log_reason`、执行记录、日志、OpenAPI 示例、前端 DOM 与截图、测试快照文件、commit 内容。
2. 日志沿用 Phase 3 约束：不用 `logger.exception`、不传 `exc_info`、不记录 `log_reason`/`__cause__`/异常消息；只记 kind、stage、ids、异常类型名、身份名、变量名。
3. 测试用秘密使用明显的假值（如 `tp-fake-secret-9f31c2a7`），并把该值作为泄密断言的搜索目标；不得使用真实凭证。
4. 不新增运行时依赖（包括 pydantic 进 Engine）；需要时停止并报告。
5. 不新增数据库表或迁移；执行记录沿用现有结构，只是公开结果里多一个 `auth` 字段。
6. 发现需要改动已批准决策、跨仓协议或错误合同时，停止并报告，不自行决定。
7. 每个实现 Agent 对完整 diff 做中文 comment pass：注释说明边界、不变量和原因，不逐行翻译。

### 0.4 Agent 0 审计结论与裁决（已确认，写入 Agent 1/2 任务）

Q1–Q7 裁决见设计稿 §6.1，Agent 1/2 必须逐条实现并测试。

审计发现的高/中风险项（主线程已在代码中逐一确认）：

| # | 仓库 | 问题 | 要求 |
|---|---|---|---|
| H1 | Engine | `executor.py` 的 `_SENSITIVE_HEADER_NAMES`（6 个名字）与 `redact._SENSITIVE_KEYS` 不一致，也不认自定义头名；凭证放在 `X-Tenant-Token` 之类的头里会原样进入公开结果。该常量上方注释写着“完整值只允许进入日志链路”，与日志硬约束矛盾 | 注入的头名一律按敏感处理，**并且**按值替换，两条各有测试；核实执行器任何日志都不含 header 值，改正该注释；两份敏感名单收敛为单一来源 |
| H2 | Engine | Case 路径 `api_step_executor.py` 构造 `ApiExecutionRequest` 时不带 auth；auth 须经 `case_runner` → `StepExecutionContext` → `preflight_api` → 步骤请求逐层传递，漏一层步骤就匿名执行，目标返回 401 时又表现为普通业务失败 | 用回显目标断言 Case 步骤**确实发出了**凭证 header（不能只断言结果里有 `auth` 字段）；预检路径同样覆盖 |
| H3 | Control | `CaseStepResult`（`case_schemas.py`）未设 `extra="allow"`，步骤新增的 `auth` 会被静默丢弃 | 显式声明 `auth` 字段（不要改成 `extra="allow"`），单 API 结果同理；重新冻结 OpenAPI 快照 |
| M1 | 两仓 | Control `execution_service.py` 两处 `except (TypeError, ValueError)` 把 `str(exc)` 直接作为 422 detail 返回 | Engine：所有新 DTO 构造错误的消息只含字段名/身份名/变量名，绝不含值，加测试。Control：加测试证明凭证相关的 422 detail 不含测试秘密值 |
| M3 | Engine | 同名头覆盖若区分大小写，会同时发出 `Authorization` 与 `authorization` | 按 Q3 不区分大小写，测试实际发出的请求只有一份 |
| L1 | Control | `.gitignore` 未忽略 `.env*` | Agent 2 加上 |
| L2 | Control | 本地目标服务没有回显接口 | Agent 4 增加（Agent 1 在 Engine 测试中用假 client 断言即可） |

## 1. 目标架构

```text
Control
  auth_loader: {config_root}/auth/{env}.yaml + 进程环境变量
    -> AuthTable { identities: {名: StaticHeaderAuth | UnavailableAuth}, default_identity }
  ExecutionService.run_single_api / run_case
    -> EnginePort.execute(request with auth=AuthTable)        # 唯一跨仓 seam，Engine 0.3.x
Engine
  preflight: 按资产 auth 声明解析身份；用到 UnavailableAuth / 未知身份 / 未知 kind -> invalid_environment
  执行: 每次 client.request 显式合并凭证 headers（不修改共享 client）
  公开面: 注入的 header 名一律按敏感处理 + 本次执行全部秘密值按值替换为 ***
```

## 2. 执行顺序与门禁

```text
Agent 0：只读审计 + golden fixtures 提案
  -> 主线程确认
  -> Agent 1：Engine 0.3.0（DTO + 预检 + 注入 + 按值脱敏）
  -> 主线程独立双轴审查
  -> Agent 2：Control 后端（auth 配置加载 + 映射 + OpenAPI 冻结）
  -> 主线程审查
  -> Agent 3：Control 前端（展示身份名与 header 名）
  -> 主线程审查
  -> Agent 4：跨仓集成 + 泄密矩阵 + 干净安装
  -> 主线程终验；推送 / PR / tag 由用户授权
```

建议分支：

```text
codex/phase3-3-engine-auth          (Engine，基于 4715c61)
codex/phase3-3-control-auth-backend (Control，基于 c5a904b)
codex/phase3-3-control-auth-frontend(Control，基于 backend 分支)
codex/phase3-3-integration-engine / codex/phase3-3-integration-control
```

worktree 统一放在 `/Users/jiavin/Developer/products/testplat1/worktrees/phase3-3-*`。

## 3. Agent 0：只读审计

````markdown
你负责 Phase 3.3a 启动前审计。只读：不改文件、不创建分支、不提交。

## 权威输入
- `notes/phase3.3-auth-design-draft.md`（已批准）与 `notes/phase3.3-agent-prompts.md` §0
- Engine `origin/main @ 4715c61`（用 `git show origin/main:<path>` 读取，不要信本地 main）
- Control `origin/main @ c5a904b`

## 任务
1. 记录两仓 remote、HEAD、dirty 状态、worktree 列表。
2. Engine：确认 API 资产加载对未知字段（如 `auth:`）的现状是忽略、报错还是保留；列出 `Api` 模型、Registry、resolver、executor 中需要改动的点。
3. Engine：列出所有可能输出请求内容的位置（`to_dict`、`repr`、`_build_public_request`、错误构造、`partial_outcome`、alias 绑定、日志），逐一说明凭证值会不会经过。
4. Engine：确认 `HttpxClient` 生命周期（每次 execute 新建 / 注入共享），说明凭证注入如何保证不串号。
5. Control：列出执行记录写入、日志、OpenAPI schema、前端展示请求 headers 的全部位置；确认 Control 的 API YAML parser 遇到 `auth:` 的行为。
6. 提议 golden fixtures：资产（声明 admin / none / 未声明 / 未知身份 / 与凭证同名 header）、auth 配置（正常 / YAML 非法 / 变量名不在白名单）、本地回显目标（把 header 回显到非敏感键）。
7. 对照设计稿 §6 的 10 条安全验收，逐条写出由哪个 Agent、哪一层测试覆盖。
8. 只列真正的歧义，不重新开放 D1–D6。

## 交付
- 基线记录、改动点清单、泄密路径表、fixture 提案、§6 覆盖映射、待裁决问题。
````

## 4. Agent 1：Engine 0.3.0

````markdown
你负责 Engine Phase 3.3a。仓库 `/Users/jiavin/Developer/products/testplat1/testplat-engine`。
从 `origin/main @ 4715c61` 新建 worktree `worktrees/phase3-3-engine-auth`，分支 `codex/phase3-3-engine-auth`。

## 权威输入
设计稿 §2、§4、§5.2–§5.4、§6、§6.1（Q1–Q7 裁决）、§9；手册 §0，**尤其 §0.4 的 H1、H2、M1、M3**，每条都必须在回传中写明对应的实现位置和测试名。

## 必须交付
1. 公开 DTO（`interface.py` 导出）：`SecretStr`、`StaticHeaderAuth`、`UnavailableAuth`、`AuthMaterial`（判别联合）、`AuthTable`；`ApiExecutionRequest.auth` 与 `CaseExecutionRequest.auth`，默认 `None`。
   - `SecretStr`：`repr`/`str` 为 `***`；`to_dict`、`json.dumps`、pickle 显式抛错；只能经 `reveal()` 取值，且全仓只有注入点调用（加测试用 grep/AST 约束）。
   - 请求 `to_dict()` 中 `auth` 只输出身份名、kind、header 名、`default_identity`。
2. API 资产新增可选 `auth` 字段：字符串身份名或 `none`；类型错误为 `invalid_asset`。
3. 预检：按设计稿 §5.3 的顺序解析身份；Case 在任何步骤执行前一次性检查全部步骤。未知身份、`UnavailableAuth`、未知 kind → `InvalidEnvironmentError`（preflight）；`public_message` 只含身份名，`UnavailableAuth` 时附带其 `public_reason`。
4. 注入：在 `client.request` 调用处合并 headers，凭证优先；不修改任何共享 client 状态。
5. 公开面：
   - 注入的 header 名在公开请求中按敏感处理（值为 `***`），无论是否在 `_SENSITIVE_KEYS` 中。
   - 本次执行全部秘密值做按值替换（公开请求、公开响应、错误 `public_message`、`partial_outcome`、alias 绑定值）；替换在现有按键脱敏之后执行，覆盖嵌套字符串中的子串。
   - 长度 < 8 的秘密：构造 `StaticHeaderAuth` 时即拒绝（Control 应先拦截为 `UnavailableAuth`）。
   - 单 API outcome 与每个 Case 步骤结果新增公开字段 `auth: {identity, header_names, overridden_headers} | null`。
6. 版本：`0.3.0`；`docs/COMPATIBILITY-0.3.md` 说明新增字段、默认行为与 `v0.2.0` 一致、3.3b 预留的 kind 扩展规则。
7. 测试：覆盖设计稿 §6 中 Engine 层能覆盖的全部条目（1 的 Engine 部分、2、3、4、5、8、9、10），以及未知 kind、Case 预检一次性失败、`auth=None` 回归。
8. Agent 0 审计项：
   - H1：注入头名敏感化与按值替换各自独立测试（用 `X-Tenant-Token` 这类不在任何名单里的头名）；两份敏感名单收敛为单一来源；核实并改正“完整值只允许进入日志链路”注释，加日志捕获测试。
   - H2：auth 沿 `case_runner` → `StepExecutionContext` → `preflight_api` → 步骤请求传递；用记录实际请求的假 client 断言 Case 每个步骤发出的 header 正确，两个身份交替时不串号。
   - M1：每个新 DTO 的构造错误消息不含秘密值（参数化测试，搜索测试秘密值）。
   - M3 / Q3：资产头与凭证头大小写不同时只发出一份，`overridden_headers` 为资产原始写法。
   - Q2：`auth=None` + 资产声明具体身份 → 422；Q4：alias 中凭证子串为 `***` 且键保留，`redact.py` 注明与 `strip_sensitive_json` 的区别；Q5：一个 header 多个秘密时每个都被替换；Q6：`auth: null` 等同未声明，`auth: ""` 为 `invalid_asset`。

## 禁止
- 读环境变量、读文件取秘密、实现任何登录逻辑、引入缓存、修改共享 client、新增依赖、改错误 kind 集合。

## 验收命令
`uv run ruff check .`、`uv run pytest -q`、`uv build`；另执行 `rg -n "reveal\(" src` 并在交付中列出全部命中。

## 回传格式
分支与 commit SHA（含父提交）；改动文件列表；新公开 API 签名；测试数量与命令输出摘要；与设计稿的偏差（逐条说明原因）；未解决问题。
````

## 5. Agent 2：Control 后端

````markdown
你负责 Control Phase 3.3a 后端。仓库 `/Users/jiavin/Developer/products/testplat1/testplat-control`。
从 `origin/main @ c5a904b` 新建 worktree `worktrees/phase3-3-control-auth-backend`，分支 `codex/phase3-3-control-auth-backend`。
在 Agent 1 通过主线程审查后开始。

## 依赖
临时用 `git = "file:///Users/jiavin/Developer/products/testplat1/testplat-engine", rev = "f631e2e3f9e2bad146a9182062ae0d99ea5c62b5"`（Agent 1 已审查 commit）锁定，约束改为 `>=0.3.0,<0.4`，并在 `pyproject.toml` 注明“合并前必须改为远程精确锁定”。不得用 path editable，不得修改 sys.path。

## 必须交付
1. `services/auth_loader.py`：读取 `{config_root}/auth/{env}.yaml`（不存在 → `AuthTable` 为 `None`，行为与 Phase 3 一致）。
   - schema：`identities.<名>.headers.<Header>: "<模板>"`，`default_identity` 可选；未知字段报错。
   - 模板只允许 `${secret:NAME}` 占位，可与字面量拼接（如 `Bearer ${secret:X}`）；NAME 必须匹配 `^TESTPLAT_[A-Z0-9_]+$`。
   - 变量缺失、为空、展开后秘密 < 8 字符、NAME 不在白名单 → 该身份生成 `UnavailableAuth(public_reason=只含变量名)`，**不在 Control 侧直接报错**，由 Engine 在用到时报 422。
   - YAML 非法、schema 错误、`default_identity` 指向不存在的身份 → 直接 `InvalidEnvironmentError`（preflight 422，不保存）。
   - 身份名与 header 名校验：身份名 `^[a-z][a-z0-9_]*$`；header 名符合 RFC 7230 token。
2. `run_single_api` 与 `run_case` 把 `AuthTable` 传给 Engine；Fake Engine harness 同步支持（用于前端门禁）。
3. OpenAPI：结果与历史详情新增 `auth` 字段（身份名、header 名、被覆盖的 header 名，可为 null）；冻结示例响应，供 Agent 3 使用。
4. 日志：只记身份名、变量名、kind；`auth_loader` 中任何异常只记类型名。
5. 测试：设计稿 §6 中 Control 层条目（1 的记录/日志/OpenAPI 部分、6、7、10），并对 SQLite 原始行和捕获日志做秘密值全文搜索。
6. Agent 0 审计项与裁决（设计稿 §6.1、手册 §0.4）：
   - H3：`CaseStepResult` 与单 API 结果 schema 显式声明 `auth` 字段（不得改成 `extra="allow"`），重新冻结 OpenAPI 快照，并测试 `auth` 能从 Engine 结果完整透传到响应和历史详情。
   - M1：测试凭证相关的 422 detail 不含测试秘密值；如果 `str(exc)` 有带出值的可能，改为安全文案。
   - Q2：没有 auth 配置文件时传 `auth=None`，不传空表。
   - Q3：同一身份内 header 名仅大小写不同 → schema 错误 422。
   - Q5：一个 header 模板允许多个 `${secret:}`，每个变量单独检查长度与白名单。
   - Q6：配置里出现名为 `none` 的身份 → schema 错误 422。
   - Q7：`default_identity` 指向不存在的身份 → 加载即 422；指向不可用的身份 → 交给 Engine 在用到时报错。
   - L1：`.gitignore` 增加 `.env*`。
7. Agent 1 交付裁决（设计稿 §6.2，Engine 基线 `f631e2e`）：
   - A1：构造 `StaticHeaderAuth(headers=..., secret_values=...)` 时，`secret_values` 必须是该身份所有 header 模板中引用变量的原始值（去重），每个 ≥ 8；不得省略、不得依赖 Engine 的兜底。测试断言：传给 Engine 的 `secret_values` 与模板引用的变量一一对应；当原始值短于 8 时生成 `UnavailableAuth`，Engine 构造不会被触发。
   - A2：header 模板不含任何 `${secret:}` → 配置 schema 错误 422。
   - 构造 Engine DTO 时如果仍抛 `TypeError`/`ValueError`，响应 detail 不得含秘密值（M1），加测试覆盖。

## 禁止
- 把秘密写入 `environment`/`params`、数据库、日志或响应；新增数据库表；提供秘密录入 API；解析 Case YAML。

## 验收命令
`uv sync --extra dev --locked`、`uv run ruff check .`、`uv run ruff format --check <本次新增/修改的文件>`、`uv run pytest -q -rs`。

## 回传格式
同 Agent 1，另附 OpenAPI 差异摘要与冻结的示例响应路径。
````

## 6. Agent 3：Control 前端

````markdown
你负责 Control Phase 3.3a 前端。基于 Agent 2 分支新建 `codex/phase3-3-control-auth-frontend`，worktree `worktrees/phase3-3-control-auth-frontend`。
在 Agent 2 的 OpenAPI 冻结并通过主线程审查后开始。

## 必须交付
1. 单 API 结果、Case 步骤结果、历史详情显示“身份：admin”或“匿名”；可展开查看注入的 header 名与被覆盖的 header 名。不显示任何值，不提供录入秘密的 UI。
2. 预检 422 中身份相关的错误按现有错误展示方式呈现，文案来自后端 `public_message`。
3. `e2e:gate` 增加身份相关截图（390 / 1440 两个视口）：有身份、匿名、header 覆盖、身份不可用 422；断言 DOM 中不出现测试秘密值。
4. 沿用现有主题与组件，不引入新依赖。
5. 契约来源：Agent 2 冻结的 `docs/phase3_3/openapi-phase3_3.json` 与 `docs/phase3_3/examples/`（Control `12a8d6a`）。注意：
   - 单 API 运行响应与历史详情有顶层 `auth`；Case 步骤的 `auth` 在每个步骤上。
   - `auth` 为 null 时显示“匿名”；但 Case 中**因系统错误失败的步骤**（`error.stage == "execution"` 且 kind 为 timeout/transport/internal）`auth` 也为 null，这时显示“—”，不要显示“匿名”（设计稿 A4）。
   - 示例里两条错误文案来自 Fake Engine（B5），前端不得硬编码任何错误文案，一律展示后端 `detail`/`message`。

## 验收命令
`pnpm install --frozen-lockfile`、`pnpm lint`、`pnpm test --run`、`pnpm build`、`pnpm e2e:gate`（先启动 Fake 后端 `uv run uvicorn testplat_control.harness_fake:app --host 127.0.0.1 --port 8000` 与 `pnpm dev`）。

## 回传格式
commit SHA、改动文件、测试数量、截图清单、偏差与未解决问题。
````

## 7. Agent 4：跨仓集成

````markdown
你负责 Phase 3.3a 跨仓集成验收。
Engine：从 Agent 1 已审查 commit 建 `codex/phase3-3-integration-engine`；Control：从 Agent 3 已审查 commit 建 `codex/phase3-3-integration-control`。

## 必须交付
1. 在 `scripts/local_target_server.py` 增加回显端点：把请求 headers 回显到非敏感键下、把秘密放进错误响应正文。
2. 真实 Engine 矩阵（扩展 `scripts/run_real_engine_matrix.sh`，配置仍写临时目录）：设计稿 §6 全部 10 条，其中 5（两身份交替）要在单 API 与 Case 混跑中验证。
3. 泄密全文搜索：对后端日志、SQLite 文件、矩阵报告、截图所在页面的 DOM 搜索测试秘密值，结果写入证据文件。
4. Phase 3 回归：无 auth 配置的项目跑原有真实矩阵，结果不变。
4.1 用真实 Engine 重新生成 `docs/phase3_3/examples/` 中的错误响应示例，替换 Fake Engine 的措辞（设计稿 B5），并保持 `tests/test_auth_examples.py` 通过。
4.2 header 注入防护（B2）：变量值含 `\r\n` 时身份不可用、请求不发出。
4.3 浏览器身份展示（B8，替代前端门禁中的 route mock）：用真实 Engine + 临时 `auth/test.yaml` + 假秘密环境变量，在 390 / 1440 两个视口截图：单 API 有身份、匿名、header 覆盖、身份不可用 422；Case 混合身份；Case 系统错误步骤的“—”（该步骤必须在截图可视区域内，Agent 3 的同名截图里它在折叠线以下）；历史详情。每张截图同时对页面 DOM 搜索测试秘密值，结果写入证据文件。
4.4 截图写入新目录 `frontend/screenshots/phase3_3-real-engine/`；跑门禁时被重新生成的旧截图必须还原，不得提交。
5. 干净安装证据：在新目录、新 uv 缓存下安装；Engine 远程锁定需要主线程取得用户推送授权后才能完成，在此之前标记 BLOCKED 并说明。

## 验收命令
两仓全部门禁 + 真实矩阵 + `e2e:gate`。

## 回传格式
两仓分支与 SHA、门禁输出摘要、矩阵报告与泄密搜索证据路径、BLOCKED 项、已知缺口。
````

## 8. 主线程审查清单

1. 审 Agent 0 结论，确认 fixtures 与 §6 覆盖映射。
2. Agent 1：独立双轴审查；重点核对 `reveal()` 调用点、全部输出路径的按值脱敏、Case 预检一次性检查、判别联合对未知 kind 的处理、`auth=None` 时与 `v0.2.0` 逐字段一致。
3. Agent 2：核对白名单正则、`UnavailableAuth` 与直接 422 的划分、OpenAPI 差异、SQLite 原始行与日志的秘密搜索。
4. Agent 3：截图与 DOM 秘密搜索。
5. Agent 4：亲自复跑两仓门禁与真实矩阵，亲自做泄密全文搜索。
6. 向用户报告 BLOCKED 项；只有用户明确授权后才推送、建 PR、打 tag 或合并。
7. 3.3a 合并后立即起草 3.3b（登录换 token、Cookie 会话）设计稿，见设计稿 §9。
