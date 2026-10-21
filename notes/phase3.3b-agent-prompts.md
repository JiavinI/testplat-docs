# Phase 3.3b Agent 提示词：登录鉴权（通用 / 配置型特殊 / 插件型特殊）

日期：2026-10-13
权威设计：`notes/phase3.3b-login-auth-design-draft.md`（v2，已批准：D7–D15、E1–E4）
前置：`notes/phase3.3-auth-design-draft.md`（3.3a）与 `notes/phase3.3-agent-prompts.md` §0.3 全体硬约束，继续有效。
目标：API 请求可以按身份**按需登录**换取鉴权信息并注入；登录流程分三档（通用单步 / 配置型多步 / 代码插件）；登录失败作为独立类别醒目呈现；请求体支持 json / form / query / raw。

> 所有 Agent 只提交本地 commit，不推送、不建 PR、不打 tag、不合并。

## 0. 基线与已定裁决

### 0.1 基线

- Engine：`origin/main @ 8564040add59c027e5d2aa7edb8f56b6693a5960`，tag `v0.3.0`。Agent 0 已确认本地 `main` 等于该 SHA；仍**从 `origin/main` 或精确 SHA 建 worktree**，不要依赖本地分支状态。
- Control：`origin/main @ d29d615`（PR #37 合并后），依赖远程锁定 Engine `8564040`。
- 3.3a 终验数字（规划证据，不代替复跑）：Engine 335 tests；Control 300 tests；前端 101 tests、lint、build、`e2e:gate`、真实 Engine 鉴权矩阵 28/28 通过。
- 旧项目 `/Users/jiavin/Developer/products/testplat-new` 只读，仅作需求证据（`config/envs/_global.yaml` 的 `auth` 段、`src/testplat/plugins/strategies/`、`src/testplat/application/api_auth_provider.py`）。不得复制其业务模型或代码。

### 0.2 已批准决策（不得重新开放）

| # | 裁决 |
|---|---|
| 三档 | 第 1 档通用 = 一个 `steps` + 一条 `extract`（`body_json`/`response_header`，`as: token`）+ `inject` 引用 `${login:token}`；第 2 档 = `generic_flow` 多步/多提取/Cookie/probe；第 3 档 = `LoginStrategy` 插件 |
| D7 | 登录在 auth 配置的 `login.steps` 中描述，不复用 API 资产 |
| D8 | 用到身份时才登录；本次执行内按 (执行, 身份) 保存状态；失效后再次用到自动重登 |
| D9 | 登录失败是独立类别 `login_failed`，归属到触发登录的步骤，后续步骤 `skipped(login_failed)`；返回 200 并保存 |
| D10 | 提取来源 `body_json` / `response_header` / `cookies` / `html_meta`，可多个命名值；插件直接产出命名值 |
| D11 | 资产与登录步骤的请求体类型 `json` / `form` / `query` / `raw`；`multipart` 为 `invalid_asset` |
| D12 | Cookie 身份在本次执行内的所有响应都更新其 jar |
| D13 | 不做跨执行缓存；有效性判断在本批建立（TTL / 响应过期时间 / Cookie expires，固定提前 30 秒） |
| D14 | Engine `0.4.0`；Control 约束 `>=0.4.0,<0.5` |
| D15 | 方案 C：Engine 定义 `LoginStrategy` 协议并内置 `generic_flow`；插件独立包，Control 启动时注册 |
| E1 | 插件发现 = entry point `testplat_engine.login_strategies` **且** Control 配置白名单 `login_plugins` 列出 |
| E2 | 登录超时 / 连接失败也是 `login_failed`（`reason = timeout` / `transport`），不映射 504/502 |
| E3 | 安全余量固定 30 秒 |
| E4 | 本批不做 `vcode_redis` 示例包 |

### 0.3 全体硬约束（在 3.3a §0.3 之上追加）

1. token、Cookie 值、登录响应中的任何命名值、`connections` 中的密码，全部视为秘密：不进 DTO 的 `repr`/`to_dict`、公开结果、错误消息、日志、记录、DOM、截图、测试快照、commit。
2. 命名值一经产出，**先**加入本次执行的按值替换集合，**再**构造任何公开快照；测试必须用“目标服务回显 token”的方式验证顺序。
3. 插件拿到的只有 `LoginContext`（专用 client、host URL、params、connections、record 回调）；插件不能访问资产、业务 client、其它身份的状态、Control 对象。
4. 不新增 Engine 核心运行时依赖（仍只有 httpx、pyyaml）；Redis 等依赖只能在插件包。
5. 不新增数据库表或迁移。
6. 日志约束沿用：不用 `logger.exception`/`exc_info`，只记 kind、stage、ids、身份名、策略名、步骤名、异常类型名。
7. 每个实现 Agent 对完整 diff 做中文 comment pass。

### 0.4 Agent 0 审计结论与裁决（已确认，写入 Agent 1–3 任务）

Q1–Q8 与 H1–H5 的裁决见设计稿 §12，Agent 1/2/3 必须逐条实现并测试。摘要：

| # | 仓库 | 要求 |
|---|---|---|
| H1 | Engine | 按值替换集合提升为执行级（挂会话池），快照用全部身份秘密的并集；测试：A 身份 token 被 B 身份 / 匿名步骤响应回显 → `***` |
| H2 | Engine | `HttpResponse` 新增 `cookies`、`text`；`HttpxClient` 暴露 jar；`cookies`/`html_meta` 提取与“所有响应更新 jar”基于此实现 |
| H3 | Engine | 登录子流程内捕获 timeout/transport → `login_failed`，测试断言不产生 504/502 |
| H4 | 三处 | `skip_reason` 枚举化：Engine `case_runner.py` / `interface.py`，Control `case_schemas.py`，前端 `caseParse.ts` |
| H5 | Control | `parse_execution_auth` 失败时记 warning（只含记录 id）；`ExecutionAuth` 扩展为七键 |
| M1/Q5 | Engine | 资产含 `${secret:`/`${login:`/`${login_host}` → `invalid_asset` |
| M2 | Engine | URL 参与按值替换；`query` 类型参数不进 `url`，单独 `request.query`（Q8） |
| M3/Q3 | Engine | 登录步骤与 probe 默认成功 = 状态码 < 400；`follow_redirects=False` |
| M4 | Engine | `body_type` 进公开快照；默认 json 对现有资产零影响（回归测试） |
| M5 | Control | `auth_loader` 扩展 `login` / `connections` / `login_plugins`，其余未知键仍 422 |
| L1 | Engine/Control | `create_engine(login_strategies=...)`；Control 两处调用共用同一份插件列表 |
| L2 | Engine | 更新 `interface.py` 判别联合注释为单一 `LoginAuth` |
| L3 | Engine | `reveal()` AST 白名单加入登录渲染模块，交付中列出全部命中 |
| Q1/Q2 | Control+Engine | inject 值允许字面量/`${login:}`/`${secret:}` 混合；身份级约束“至少一个 `${login:}` 或 `session: cookie`” |
| Q4 | Engine | 显式 `Content-Type` 优先；`raw` 必须有 content type |
| Q6 | Engine | 会话池走内部 kwargs + `StepExecutionContext` |
| Q7 | Engine | 登录记录先缓存后脱敏；登录失败时响应中 ≥ 20 字符的字符串叶子一律 `***` |

## 1. 目标架构

```text
Control
  auth_loader: {config_root}/auth/{env}.yaml + 进程环境变量 + login_plugins 白名单
    -> AuthTable { identities: {名: StaticHeaderAuth | UnavailableAuth | LoginAuth}, default_identity }
  engine_bootstrap: 发现 entry point 插件 ∩ 白名单 -> create_engine(login_strategies=[...])
  ExecutionService.run_single_api / run_case -> EnginePort.execute(request with auth=AuthTable)   # Engine 0.4.x
Engine
  预检: 身份解析 + 策略已注册 + strategy.validate() + host 可解析 + inject 引用可被覆盖
  会话池(每次执行): (身份) -> {client(含 cookie jar), 命名值, expires_at}
  业务步骤: 解析身份 -> 查池 -> 有效则注入/携带 cookie; 否则 登录(generic_flow | 插件) -> probe -> 入池 -> 注入
  登录失败: 当前步骤 failed(kind=login_failed), 后续 skipped(login_failed), HTTP 200, 保存
  公开面: logins[], auth{identity, kind, tier, header_names, overridden_headers, cookie_names, login_ref}
```

## 2. 执行顺序与门禁

```text
Agent 0：只读审计 + fixtures 提案
  -> 主线程确认
  -> Agent 1：Engine 0.4.0
  -> 主线程独立双轴审查
  -> Agent 2：Control 后端（含插件注册、OpenAPI 冻结）
  -> 主线程审查
  -> Agent 3：Control 前端
  -> 主线程审查
  -> Agent 4：跨仓集成 + 假插件包 + 泄密矩阵 + 干净安装
  -> 主线程终验；推送 / PR / tag 由用户授权
```

硬门禁沿用 3.3a：上一 Agent 未通过主线程审查，下一 Agent 不开始；发现需要新增依赖、迁移、跨仓协议改向、重新开放已批准决策时停止并报告。

建议分支与 worktree：

```text
codex/phase3-3b-engine-login            worktrees/phase3-3b-engine-login
codex/phase3-3b-control-login-backend   worktrees/phase3-3b-control-login-backend
codex/phase3-3b-control-login-frontend  worktrees/phase3-3b-control-login-frontend
codex/phase3-3b-integration-engine / codex/phase3-3b-integration-control
```

## 3. Agent 0：只读审计

````markdown
你负责 Phase 3.3b 启动前审计。只读：不改文件、不创建分支、不提交。

## 权威输入
- `notes/phase3.3b-login-auth-design-draft.md`（已批准）与 `notes/phase3.3b-agent-prompts.md` §0
- Engine `origin/main @ 8564040`（用 `git show origin/main:<path>` 读取，不要信本地 main）
- Control `origin/main @ d29d615`
- 旧项目 `/Users/jiavin/Developer/products/testplat-new` 只读证据

## 任务
1. 记录两仓 remote、HEAD、dirty 状态、worktree 列表。
2. Engine 执行器 client 生命周期：列出 `_acquire_client` / `execute_api` / `preflight_api` / `CaseRunner` / `ApiStepExecutor` / `StepExecutionContext` 中引入“每次执行一个会话池、login 身份专用 client”需要改动的全部位置；说明注入 client（测试用）与自建 client 两条路径如何同时满足隔离。
3. 列出所有可能输出请求/响应内容的位置（请求快照、响应快照、错误构造、`partial_outcome`、alias 绑定、`logins`、日志），逐一说明 token / Cookie / 命名值在何处加入按值替换集合才不会泄露。
4. `request.type` 影响面：资产加载、resolver、executor、公开快照、Control API 资产 parser、前端参数表单；给出 json/form/query/raw 各自的发送方式与快照形状。
5. 登录失败归属：对照 `CaseRunner._attribute_system_error`（或等价方法）说明 `login_failed` 如何归属到当前步骤并把后续标为 `skipped(login_failed)`，以及单 API 路径的形状。
6. 占位符作用域：核对资产 `${var}`、auth 配置 `${secret:}` / `${login:}` / `${login_host}` 三套解析不会互相误吞；指出需要加的负向测试。
7. 以旧项目 `_global.yaml` 的 `gsb_token`、`gsb_csrf`、`admin_go2_api`、`df_picker_api`、`cat_api` 五个 profile 为样本，逐个判断能否用 `generic_flow` 配置表达（第 1 或第 2 档）；不能表达的列出缺口。
8. 提议 fixtures：本地目标服务需要的端点（JSON 登录、表单登录、query 登录、CSRF 预取 + Cookie 登录 + probe、短 TTL 登录、回显 token/Cookie 的业务端点、401 端点）、auth 配置样例（三档各至少一个、失败样例）、测试用假插件。
9. 对照设计稿 §6 第 11–21 条，写出由哪个 Agent、哪一层测试覆盖。
10. 只列真正的歧义，不重新开放 §0.2 的决策。

## 交付
基线记录、改动点清单、泄密路径表、`request.type` 影响表、旧 profile 可表达性表、fixture 提案、§6 覆盖映射、待裁决问题。
````

## 4. Agent 1：Engine 0.4.0

````markdown
你负责 Engine Phase 3.3b。仓库 `/Users/jiavin/Developer/products/testplat1/testplat-engine`。
从 `origin/main @ 8564040` 新建 worktree `worktrees/phase3-3b-engine-login`，分支 `codex/phase3-3b-engine-login`。

## 权威输入
设计稿 §1.1、§3、§4、§5、§6、§9、§10、§11、**§12（Agent 0 审计裁决）**；手册 §0（尤其 §0.3 与 **§0.4**）。§0.4 中标注 Engine 的每一项都必须在回传中写明实现位置与测试名。

## 必须交付
1. 公开 DTO（`interface.py` 导出）：`LoginAuth`（含 `tier`）、`LoginFlow`/`LoginStep`/`ExtractRule`/`ProbeSpec`/`Validity`、`LoginStrategy` 协议、`LoginContext`、`LoginResult`；`AuthMaterial` 扩展为含 `LoginAuth`。repr/to_dict/pickle 规则同 3.3a：只暴露策略名、tier、host、步骤名、提取名、header 名。
2. `create_engine(*, login_strategies=())`：注册插件；重名或与内置 `generic_flow` 冲突 → 启动期 `ValueError`（消息只含策略名）。
3. 内置 `generic_flow`：
   - 步骤顺序执行；`body.type` 为 json/form/query/raw；`headers` 与 `path` 可引用 `${login:名字}`、`${login_host}`、`${secret:}` 已由 Control 渲染为 SecretStr。
   - `extract`：`body_json`（简化 JSONPath）、`response_header`、`cookies`（按名取值，或 `format: cookie_header`）、`html_meta`；`optional` 支持。
   - 步骤 `success`：复用断言引擎，另支持 `body_not_contains`、`reject_location_contains`（供 probe 用）。
   - `probe`：登录后立即执行，失败即 `login_failed(reason=probe_failed)`。
4. 会话池：每次 `execute` 一个池；login 身份各自一个 `HttpxClient`（cookie jar 隔离），执行结束统一关闭；匿名 / 静态 header 身份保持每次调用新建。Cookie 身份的业务响应更新其 jar。
5. 按需登录与有效性：业务步骤解析身份后查池；`validity.ttl_seconds` / `expires_from` / Cookie expires 任一到期（提前 30 秒）→ 重登；`logins` 追加新记录，步骤 `auth.login_ref` 指向本次使用的那条。
6. 注入：`inject.headers` 模板用命名值渲染；与资产 header 冲突沿用 Q3（不区分大小写、凭证优先、记录 `overridden_headers`）。
7. 脱敏：命名值（≥ 8）与 jar 中 Cookie 值（≥ 8）**先**入按值替换集合再构造任何快照；登录请求体中的 SecretStr 参数同样替换；`Cookie`/`Set-Cookie` 头继续按名脱敏。
8. 失败语义：`login_failed` 错误（`error.login = {identity, strategy, failed_step, reason}`，reason ∈ assertion / extract_missing / token_too_short / probe_failed / timeout / transport / strategy_error）；归属到触发步骤，后续 `skipped(skip_reason=login_failed)`；单 API 路径返回 `status=failed` + 同形 error。登录阶段的超时/连接失败**不得**抛系统异常。
9. 预检：策略未注册、`validate()` 失败、host 不可解析、`inject` 引用的名字无来源 → `InvalidEnvironmentError`，且在 Case 任何步骤执行前一次性检查。
10. `request.type`：资产 `json`（默认）/ `form` / `query` / `raw`（需 `content_type`）；`multipart` 或未知值 → `invalid_asset`；公开请求快照新增 `body_type`。
11. 公开结果：`logins` 列表；`auth` 扩展为 `{identity, kind, tier, header_names, overridden_headers, cookie_names, login_ref}`，静态 header 身份 `tier=null`、`cookie_names=[]`、`login_ref=null`。
12. 版本 `0.4.0`；`docs/COMPATIBILITY-0.4.md`：新 kind、新错误 kind、新 skip_reason、`auth` 扩展、`request.type`、插件协议与稳定性承诺。
13. 测试：设计稿 §6 第 11–21 条中 Engine 层能覆盖的全部；旧项目五个 profile 的 `generic_flow` 等价配置各一条（用假目标）；测试用假插件（产出命名值、抛异常、超时三种）；`auth=None` 与仅静态 header 的回归。

## 禁止
- 读环境变量、读文件取秘密；跨执行缓存；401 重登；登录步骤 retry；multipart；新增核心依赖；修改共享 client 状态。

## 验收命令
`uv run ruff check .`、`uv run pytest -q`、`uv build`；`rg -n "reveal\(" src` 并在交付中列出全部命中（只允许注入与登录渲染模块）。

## 回传格式
分支与 commit SHA（含父提交）；改动文件；新公开 API 签名；测试数量与输出摘要；与设计稿的偏差逐条说明；未解决问题。
````

## 5. Agent 2：Control 后端

````markdown
你负责 Control Phase 3.3b 后端。仓库 `/Users/jiavin/Developer/products/testplat1/testplat-control`。
从 `origin/main @ d29d615` 新建 worktree `worktrees/phase3-3b-control-login-backend`，分支 `codex/phase3-3b-control-login-backend`。
在 Agent 1 通过主线程审查后开始。

## 依赖
临时用 `git = "file:///Users/jiavin/Developer/products/testplat1/testplat-engine", rev = "7a7ebb8c47eb3a81e978c7346470618d181d4309"`（Agent 1 已审查 commit）锁定，约束改为 `>=0.4.0,<0.5`，`pyproject.toml` 注明“合并前必须改为远程精确锁定”。不得 path editable。

## 必须交付
1. `auth_loader` 扩展：
   - `identities.<名>.login`：`strategy`（默认 `generic_flow`）、`host`、`steps`、`validity`、`session`、`probe`、`inject`、`params`；与 `headers` 互斥。
   - 模板占位符只允许 `${secret:NAME}`、`${login:名字}`、`${login_host}`；其它 `${` → 422。`${secret:}` 规则沿用 3.3a（白名单、≥ 8、控制字符）；任一变量不可用 → 该身份 `UnavailableAuth`。
   - 档位判定写入 `tier`：恰好一个 step、恰好一条 extract（body_json/response_header，as=token）、inject 只引用 `${login:token}`、无 session/probe → `generic`；其余 `generic_flow` → `configured`；非内置策略 → `plugin`。
   - `inject.headers` 的值可为字面量、`${login:}`、`${secret:}` 任意组合（§12 Q1）；身份级约束为“至少一个 inject 值引用 `${login:}`，或声明 `session: cookie`”；`session: cookie` 时 `inject` 可省略（Q2）。`${login:}` 引用名必须由 extract 声明（插件策略由插件 `validate` 判定，Control 不校验）。字面量注入头同样按名遮蔽。
   - `connections.<名>`：非秘密字段原样；`password` 等含 `${secret:}` 的字段渲染为 SecretStr。
   - 顶层 `login_plugins: [名...]` 白名单。
2. 插件注册（E1）：`engine_bootstrap` 用 `importlib.metadata.entry_points(group="testplat_engine.login_strategies")` 发现，取与白名单的交集传给 `create_engine(login_strategies=...)`；白名单列出但未安装 → 启动期 `EngineBootstrapError`（消息只含策略名）；安装但未列出 → 不注册并记一条 info 日志。
3. 执行服务：`AuthTable` 构造与透传不变；OpenAPI 新增 `logins`、扩展 `ExecutionAuth`（`kind`、`tier`、`cookie_names`、`login_ref`，仍 `extra="forbid"`）、新错误 kind `login_failed`（含 `login` 子对象）、新 `skip_reason`；冻结示例响应。
4. Fake Engine harness 同步支持 login 身份与 `login_failed`，供前端门禁。
5. 顺带修复 `main.py:23`：`logger.error("Engine 启动诊断失败: %s", exc)` 改为只记异常类型名。
6. 测试：设计稿 §6 中 Control 层条目；档位判定的边界用例；白名单交集逻辑；`connections` 密码不出现在记录/日志/422 detail；OpenAPI 快照。
7. Agent 1 交付裁决（设计稿 §13，Engine 基线 `7a7ebb8`）：
   - U3：`AuthTable` 必须带 `connections`（没有连接信息时传空映射，不是省略字段）。
   - U3：`ExecutionAuth` 扩成七键 `{identity, kind, tier, header_names, overridden_headers, cookie_names, login_ref}`，`extra="forbid"` 保持；`parse_execution_auth` 失败记 warning（只含记录 id），不再静默丢身份。
   - U3：OpenAPI 与冻结示例覆盖 `logins`（只在 Case / 单 API 结果顶层，不在步骤上）、`body_type`（默认 `"json"`）、`query`（`query` 类型请求的独立字段）。
   - A6：`login_failed` 步骤的 `request`/`response` 为 null，身份信息只在 `auth.login_ref` + 顶层 `logins`。
   - A8：无登录时 `logins` 仍输出 `[]`；未声明 `request.type` 的资产结果仍有 `body_type: "json"`。
   - A9：生产入口不得注入共享 `http_client`；只用 `create_engine(login_strategies=...)`。
   - A3：配置文档写明短于 8 的附属值（如 user_id）不会按值脱敏。

## 禁止
- 把秘密写入 `environment`/`params`/数据库/日志/响应；在 Control 发任何登录请求；新增数据库表；解析 Case YAML。

## 验收命令
`uv sync --extra dev --locked`、`uv run ruff check .`、`uv run ruff format --check <本次改动文件>`、`uv run pytest -q -rs`。

## 回传格式
同 Agent 1，另附 OpenAPI 差异摘要与冻结示例路径。
````

## 6. Agent 3：Control 前端

````markdown
你负责 Control Phase 3.3b 前端。基于 Agent 2 分支新建 `codex/phase3-3b-control-login-frontend`，worktree `worktrees/phase3-3b-control-login-frontend`。
在 Agent 2 的 OpenAPI 冻结并通过主线程审查后开始。

## 必须交付
1. `login_failed` 独立样式：与断言失败、系统错误明显区分（独立色板与图标，标题固定为“登录失败”），默认展开登录过程面板；Case 列表与历史列表中该记录也带“登录失败”标记。
2. 登录过程面板：按 `logins` 顺序展示身份、策略、档位（通用 / 配置型 / 插件）、每步请求与响应（已脱敏）、probe 结果、失败原因；同一身份多次登录按 `login_ref` 与步骤关联。
3. 步骤 `auth` 展示扩展：`kind`、`tier`、Cookie 名、`login_ref`；`skipped(login_failed)` 显示“因登录失败跳过”，与 `fail_fast` 区分。
4. 请求体类型：结果视图中按 `body_type` 展示 json/form/query/raw；参数表单不新增录入能力。
5. `e2e:gate`（Fake 后端 + route mock）新增两视口截图：通用登录成功、配置型多步登录成功、插件登录成功、登录失败（断言 / 超时各一）、失效重登（两条 logins）、`skipped(login_failed)`；DOM 不含测试秘密值。
6. 不硬编码任何错误文案，一律展示后端 `detail`/`message`/`reason` 的映射文案（映射表只针对 reason 枚举）。

## 验收命令
`pnpm install --frozen-lockfile`、`pnpm lint`、`pnpm test --run`、`pnpm build`、`pnpm e2e:gate`（先启动 Fake 后端与 `pnpm dev`）。

## 回传格式
commit SHA、改动文件、测试数量、截图清单、偏差与未解决问题。
````

## 7. Agent 4：跨仓集成

````markdown
你负责 Phase 3.3b 跨仓集成验收。
Engine：从 Agent 1 已审查 commit 建 `codex/phase3-3b-integration-engine`；Control：从 Agent 3 已审查 commit 建 `codex/phase3-3b-integration-control`。

## 必须交付
1. 本地目标服务新增端点：JSON 登录、表单登录、query 登录、CSRF 预取（html_meta）+ Cookie 登录 + 受保护 probe 页、短 TTL 登录（响应含 expiresIn）、回显 token/Cookie 的业务端点、401 端点、`raw` 请求体回显。
2. 测试用假插件包（放在 `tests/plugins/testplat_login_fake/`，以本地 path 安装到矩阵环境，不发布）：声明 entry point，产出命名值；用它验证 E1 的交集逻辑（白名单缺失时不生效）。
3. 真实 Engine 矩阵（扩展 `scripts/run_real_engine_matrix.sh`，配置写临时目录）：设计稿 §6 第 11–21 条全部；旧项目五个 profile 的等价配置对本地目标跑通；3.3a 鉴权矩阵与 Phase 3 回归不变。
4. 浏览器身份展示（真实 Engine，390/1440）：通用登录、配置型 Cookie 登录、插件登录、登录失败（断言 / 超时）、失效重登、`skipped(login_failed)`、历史详情；每张截图对 DOM 搜索测试秘密值。
5. 泄密全文搜索：后端日志（DEBUG）、SQLite 原始行、矩阵报告、DOM 证据、docs；搜索目标含密码、token、Cookie 值、`connections` 密码。
6. 干净安装证据：新目录、新 uv 缓存；Engine 远程锁定需用户推送授权，之前标记 BLOCKED。
7. 截图写入 `frontend/screenshots/phase3_3b-real-engine/`；重新生成的旧截图必须还原。

## 验收命令
两仓全部门禁 + 真实矩阵 + `e2e:gate`。

## 回传格式
两仓分支与 SHA、门禁输出摘要、矩阵报告与泄密搜索证据路径、BLOCKED 项、已知缺口。
````

## 8. 主线程审查清单

1. 审 Agent 0 结论：会话池接入点、泄密路径表、旧 profile 可表达性、占位符作用域。
2. Agent 1：独立双轴审查；重点核对 `reveal()` 调用点、命名值入集合与快照构造的先后顺序、会话池关闭与异常路径、按需登录与失效重登、`login_failed` 归属形状、登录阶段超时不抛系统异常、`request.type` 对现有资产零影响、插件只拿到 `LoginContext`。
3. Agent 2：档位判定、白名单交集、`connections` 密码、OpenAPI 差异、`main.py:23`。
4. Agent 3：`login_failed` 是否足够醒目且不与其它失败混淆；DOM 秘密搜索。
5. Agent 4：亲自复跑两仓门禁与真实矩阵；亲自做 SQLite / 日志 / DOM 泄密搜索；核对假插件在白名单缺失时确实未注册。
6. 向用户报告 BLOCKED 项；只有用户明确授权后才推送、建 PR、打 tag 或合并。
7. 3.3b 合并后评估下一批：`vcode_redis` 示例包（E4）、401 重登、跨执行缓存、multipart。
