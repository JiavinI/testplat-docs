# Phase 3.3c Agent 提示词：验证码 Redis 登录插件

权威设计：`notes/phase3.3c-next-batch-draft.md`（v2，已批准）。本批**不改 Engine**、不改前端、不改 OpenAPI。

## 0. 基线与分工

- Engine：`v0.4.0` = `f1a0c875a8f596e145b8da51fd9ca4bfb809a88a`，只读。
- Control：`origin/main @ 789d6b62d0c7af51f14c1ba08809da5b44937638`。
- 旧项目 `/Users/jiavin/Developer/products/testplat-new` 只读，仅作需求证据（`src/testplat/plugins/strategies/vcode_redis.py`、`config/envs/_global.yaml` 的 `df_admin_api`）。不得复制代码。

| Agent | 建议模型 | 内容 | 依赖 |
|---|---|---|---|
| 1 | Sonnet | 插件包 `plugins/testplat_login_plugins/` + 删除已合入分支 | 无 |
| 2 | Sonnet | Control 连接秘密只阻断引用身份（设计稿 §5） | 无，可与 1 并行 |
| 3 | Sonnet | 集成：本地 Redis + createVCode 目标端点 + 矩阵第 22–30 条 + 泄密搜索动态值 | 1、2 通过审查 |

主线程（我）：每个 Agent 交付后复跑门禁、只读复现、给出裁决；推送 / PR / 合并由用户授权。

## 0.1 全体硬约束

1. 验证码、uuid、登录密码、token、Redis 密码全部视为秘密：不进日志、异常消息、公开结果、测试快照、commit。
2. 日志只记策略名、身份名、步骤名、异常类型名；不用 `logger.exception` / `exc_info`；不记 Redis key、uuid、host。
3. Redis 只读：代码里只允许 `GET`（连接握手的 `AUTH` / `SELECT` / `HELLO` 由客户端库发出，允许）。
4. 不新增 Control 运行时依赖；`redis` 只在插件包的 extras。
5. 不新增数据库表或迁移。
6. 依赖版本精确锁定。
7. 每个 Agent 对完整 diff 做中文 comment pass；只写说明约束的注释。

## 1. Agent 1：插件包

````markdown
你负责 Phase 3.3c 插件包。仓库 `/Users/jiavin/Developer/products/testplat1/testplat-control`。
从 `origin/main @ 789d6b6` 新建 worktree `worktrees/phase3-3c-login-plugins`，分支 `codex/phase3-3c-login-plugins`。

## 权威输入
设计稿 §2（v0.4.0 约束，必须逐条遵守）、§3、§4、§6 第 22–26 条；本手册 §0.1。

## 必须交付
1. `plugins/testplat_login_plugins/`：独立 `pyproject.toml`（hatchling），包名 `testplat-login-plugins`，版本 `0.1.0`；
   依赖 `testplat-engine>=0.4.0,<0.5`；extras `redis = ["redis==<精确版本>"]`；
   entry point `testplat_engine.login_strategies: vcode_redis = "testplat_login_plugins.vcode_redis:VcodeRedisStrategy"`。
   不加入 Control 的 `pyproject.toml` / `uv.lock`。
2. `VcodeRedisStrategy`：`validate()` 与 `login()` 按设计稿 §4.2 实现（扁平点号参数、剩余预算、三步流程、redis-py 异常转内置 `ConnectionError` / `TimeoutError` 且 `from None`、自定义异常类型名、uuid 作为 `vcode_uuid` 命名值返回）。
   - Redis 客户端通过可注入的工厂构造（测试用 fake 客户端），生产默认 `redis.asyncio.Redis`。
   - 不导入 Engine 内部模块（只用 `testplat_engine` 顶层公开导出）。
3. 记录与脱敏按 §4.3：record name 为 `create_vcode` / `redis_get` / `login`；验证码、uuid、密码在记录里写 `"***"`；Redis 一步只记连接名、key 前缀、found。
4. 测试（插件包自带 `plugins/testplat_login_plugins/tests/`）：
   - 成功路径：fake Redis + httpx `MockTransport`，经 Engine 公开入口 `create_engine(login_strategies=[VcodeRedisStrategy()])` 跑一个单 API，断言 inject 生效、`logins` 三步、公开结果中验证码（用 4 位与 6 位各一次）、uuid、密码、token、Redis 密码全部不出现。
   - 失败映射：设计稿 §6 第 25 条每一项一条测试，断言 `error.login.reason` 与公开面里的异常类型名。
   - 预算：`timeout_seconds` 很小时 → `timeout`。
   - 只读：fake 客户端记录方法名，只出现 `get` 与关闭方法。
   - `validate()`：缺必填、未知键、非法 body_type 各一条，消息不含参数值。
   - AST：`reveal()` 只出现在一个指定模块。
   - 本机有 `redis-server` 时：起临时端口的 `redis-server --save "" --appendonly no`，写入验证码，跑一次真实 GET，并用 `INFO commandstats` 断言无写命令（测试自身写验证码用独立连接，统计前 `CONFIG RESETSTAT`）。没有 `redis-server` 时 skip 并在输出注明。
5. `plugins/testplat_login_plugins/README.md`：安装（`uv pip install './plugins/testplat_login_plugins[redis]'`）、Control 白名单 `login_plugins`、设计稿 §3 的配置样例、失败原因对照表、“短值不按值替换”的说明。
6. 顺带删除已合入的 3.3b 分支（用户已同意）：
   ```bash
   cd /Users/jiavin/Developer/products/testplat1/testplat-engine
   git branch -d codex/phase3-3b-engine-login codex/phase3-3b-integration-engine
   git push origin --delete codex/phase3-3b-engine-login
   git fetch --prune
   cd ../testplat-control
   git branch -d codex/phase3-3b-control-login-backend codex/phase3-3b-control-login-frontend codex/phase3-3b-integration-control
   git push origin --delete codex/phase3-3b-integration-control
   git fetch --prune
   ```
   必须用 `-d`；任何一条报“未合并”就停下并在回传中说明，不得改用 `-D`。

## 禁止
- 修改 Engine 仓、Control `src/`、前端、OpenAPI；Redis 写命令；网络访问（Redis 只连 127.0.0.1）；推送本批分支。

## 验收命令
在插件目录：`uv venv` 后 `uv pip install -e '.[redis]' pytest pytest-asyncio`（Engine 从 `git+https://github.com/JiavinI/testplat-engine.git@f1a0c875a8f596e145b8da51fd9ca4bfb809a88a` 安装），`uv run pytest -q -rs`；
在 Control 根：`uv run ruff check .`、`uv run ruff format --check plugins/`、`uv run pytest -q`（确认 Control 自身测试不受影响）。

## 回传格式
分支与 commit SHA（含父提交）；改动文件；redis 精确版本；测试数量与输出摘要（含 skip 原因）；分支删除结果；与设计稿的偏差逐条说明；未解决问题。
````

## 2. Agent 2：Control 连接秘密只阻断引用身份

````markdown
你负责 Phase 3.3c Control 后端改动。仓库 `/Users/jiavin/Developer/products/testplat1/testplat-control`。
从 `origin/main @ 789d6b6` 新建 worktree `worktrees/phase3-3c-control-connections`，分支 `codex/phase3-3c-control-connections`。

## 权威输入
设计稿 §5、§6 第 27 条；本手册 §0.1。

## 必须交付
1. `services/auth_loader.py`：
   - `_parse_connections` 不再让秘密不可用的连接导致整次 `InvalidEnvironmentError`；返回已渲染连接与“不可用连接名”集合。
   - 删除 `load` 中 `conn_reasons` 整次 422 的分支；改为：login 身份 `params` 中任一**字符串值**（非 SecretStr）等于某个不可用连接名 → 该身份为 `UnavailableAuth`，原因只含连接名；日志只含身份名与连接名。
   - 不可用连接不进入 `AuthTable.connections`。
   - 同时修正 `_parse_connections` docstring（现有文字自相矛盾）。
2. 测试：
   - 连接秘密缺失 → 引用它的插件身份不可用（422，detail 只含身份名 / 连接名），同项目其它身份（静态 header、generic_flow）执行正常。
   - 两个连接一个可用一个不可用，只阻断引用不可用者的身份。
   - 未被任何身份引用的不可用连接 → 不影响任何执行，只记日志。
   - detail、日志、记录中不出现连接字段的值与秘密。
   - 原 `connections` 整次 422 的测试按新语义改写，并在回传中列出。
3. `docs/phase3_3/control-auth-backend.md`：补充新语义，以及“引用不存在的连接由插件在 login() 报 strategy_error，Control 不检查”。

## 禁止
- 修改 Engine、前端、OpenAPI 结构（若 OpenAPI 快照测试因此失败，停下报告）；新增依赖；推送。

## 验收命令
`uv sync --extra dev --locked`、`uv run ruff check .`、`uv run ruff format --check <本次改动文件>`、`uv run pytest -q -rs`。

## 回传格式
分支与 commit SHA（含父提交）；改动文件；改写的旧测试清单；测试数量与输出摘要；偏差与未解决问题。
````

## 3. Agent 3：跨仓集成

````markdown
你负责 Phase 3.3c 集成验收。在 Control 仓从 Agent 2 已审查 commit 新建 `codex/phase3-3c-integration`，worktree `worktrees/phase3-3c-integration`，并 merge（非 rebase）Agent 1 已审查分支。

## 权威输入
设计稿 §3、§4、§6 全部；3.3b 矩阵脚本 `scripts/run_real_engine_matrix.sh`、`scripts/login_real_engine_matrix.py`、`scripts/auth_leak_search.py`、`scripts/local_target_server.py`。

## 必须交付
1. `local_target_server.py` 新增：`POST /createVCode`（返回新 uuid，并把 4 位验证码写入矩阵 Redis 的 `{prefix}{uuid}`）、`POST /vcode/login`（form；校验验证码与 userKey，成功返回 token 与 userId）、回显 `X-TOKEN` / `X-USER-ID` 的业务端点；可配置让 createVCode 不写 Redis（用于 key 不存在）。
2. 矩阵脚本：临时端口启动 `redis-server --save "" --appendonly no --requirepass <假密码>`，trap 中关闭；以本地 path 安装 `plugins/testplat_login_plugins[redis]`；新增 auth 配置夹具（设计稿 §3 等价配置 + 连接秘密缺失的变体 + 白名单未列 vcode_redis 的变体）。
3. 新增矩阵条目覆盖设计稿 §6 第 22–28 条；Redis 不可达用一个未监听端口的连接；Redis 超时用 `DEBUG SLEEP` 或 `CLIENT PAUSE`（只对矩阵 Redis）。第 26 条用 `INFO commandstats` 在矩阵结束时断言没有写命令（目标服务写验证码用的连接不计，统计方式在回传中说明）。
4. 泄密搜索（第 29 条）：目标服务把每次签发的 token、session、uuid、验证码追加写入 `$WORK/issued-secrets.txt`（临时目录，不进证据目录）；`auth_leak_search.py` 读取后并入搜索目标；证据只记动态目标数量；动态目标为 0 时脚本失败；搜索在 trap 清理前执行，覆盖 SQLite 文件字节与逐行。
5. 浏览器（真实 Engine，390 / 1440）：`vcode_redis` 登录成功、Redis 不可达导致的登录失败两张场景各两视口；DOM 秘密搜索包含动态目标。截图写入 `frontend/screenshots/phase3_3c-real-engine/`；不得改写 `phase3_3/`、`phase3_3b/`、`phase3_3b-real-engine/` 的已提交文件（若脚本会重生成，运行后还原并在回传说明）。
6. 3.3a 28/28、3.3b 25/25、Phase 3 回归不变（第 30 条）。

## 禁止
- 修改 Engine、前端源码、OpenAPI；Redis 绑定非 127.0.0.1；推送。

## 验收命令
Control 全部门禁 + 插件包测试 + `scripts/run_real_engine_matrix.sh` + 前端 `pnpm lint` / `pnpm test --run` / `pnpm build`。

## 回传格式
分支与 SHA；门禁输出摘要；新矩阵条目逐条 PASS/FAIL；commandstats 证据；泄密搜索证据路径与动态目标数量；截图清单；被还原的文件；已知缺口。
````

## 4. 主线程审查清单

1. Agent 1：复跑插件测试；核对 `redis` 异常转换是否 `from None`、记录里验证码 / uuid 是否为 `***`、只读断言是否真实有效（不是空跑）、未导入 Engine 内部模块；核对分支删除结果。
2. Agent 2：复跑 Control 门禁；只读复现“一个连接不可用、其它身份正常”；确认 OpenAPI 未变。
3. Agent 3：亲自复跑矩阵（输出重定向到临时目录）；抽查泄密搜索的动态目标确实非空；核对旧截图目录未被改写。
4. 向用户报告；推送 / PR / 合并等用户授权。
