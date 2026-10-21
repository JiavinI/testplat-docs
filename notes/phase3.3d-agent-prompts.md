# Phase 3.3d Agent 提示词：跨执行登录缓存 + 401 重登重发

权威设计：`notes/phase3.3d-login-cache-reauth-design-draft.md`（v1，已批准）。
> 所有 Agent 只提交本地 commit，不推送、不建 PR、不打 tag、不合并。

## 0. 基线与分工

- Engine：`origin/main @ f1a0c875a8f596e145b8da51fd9ca4bfb809a88a`（`v0.4.0`）。
- Control：`origin/main @ 2dad704293ba5faaf518f1e32603cfc0e1e441e2`（#39）。
- 一律从 `origin/main` 精确 SHA 建 worktree，不依赖本地分支状态。
- 旧项目 `/Users/jiavin/Developer/products/testplat-new` 只读，仅需求证据（`src/testplat/engine/executor.py:1256-1322` 的 401 自愈）。不得复制代码。

| Agent | 建议模型 | 内容 | 依赖 |
|---|---|---|---|
| 1 | Sonnet（若首轮审查返工 ≥2 处并发/脱敏问题，改用 Opus 重做） | Engine 0.5.0：缓存 + 401 重发 + 公开契约 | 无 |
| 2 | Sonnet | Control 后端 + 前端（依赖 Agent 1 已审查 commit） | 1 通过审查 |
| 3 | Sonnet | 集成：目标服务吊销 token / 计数、矩阵第 31–45 条、3.3c 遗留小修 | 2 通过审查 |

Agent 2 后端与前端改动量都小，合为一个 Agent；它依赖 Engine 新字段，不能与 1 并行。

## 0.1 全体硬约束（在 3.3b §0.3、3.3c §0.1 之上）

1. 缓存里的命名值、指纹、HMAC 密钥、generation 全部视为秘密：不进 `repr`/`to_dict`、公开结果、异常消息、日志、测试快照、commit。
2. 日志只记身份名、策略名、`cache=hit|miss|store|invalidate|evict`、`reauth trigger_status=401`、异常类型名；不记 URL、键、指纹、过期时间。不用 `logger.exception` / `exc_info`。
3. 缓存命中路径**必须先**把静态登录秘密与缓存命名值加入执行级替换集合，再构造任何快照（设计稿 §2.5）。
4. 不新增依赖；不新增数据库表或迁移。
5. 每个 Agent 对完整 diff 做中文 comment pass；只写说明约束的注释。

## 1. Agent 1：Engine 0.5.0

````markdown
你负责 Engine 0.5.0。仓库 `/Users/jiavin/Developer/products/testplat1/testplat-engine`。
从 `origin/main @ f1a0c875` 新建 worktree `../worktrees/phase3-3d-engine`，分支 `codex/phase3-3d-engine-cache`。

## 权威输入
设计稿 §1、§2、§3、§4、§7；本手册 §0.1。现有代码先读：
`src/testplat_engine/engine/login_session.py`（SessionPool、过期计算 355 行、_check_values）、
`engine/executor.py`（_execute_api、new_session_pool）、`interface.py`（LoginAuth、AuthTable、_public_auth 七键）、
`engine/auth_injection.py`（public_auth）、`tests/test_auth_interface.py:82`（reveal AST 白名单）。

## 必须交付
1. `engine/login_cache.py`：`LoginCache(max_entries)`，LRU、每键 `asyncio.Lock`、generation、HMAC 指纹（§2.3）。
   reveal 只在本模块一个函数内，并把该模块加入 AST 白名单测试。
2. `create_engine(..., login_cache_max_entries=256)`；Engine 实例持有一个缓存，`new_session_pool()` 传入。
3. `AuthTable.cache_scope`（§2.2 校验，不进 `to_dict`）；`LoginAuth.reauth_on_401: bool = True`（指纹包含它，`to_dict` 可含该布尔）。
4. `SessionPool.ensure` 按 §2.5 三段顺序；写入条件 §2.4；作废按 generation（§2.7）。
5. `_execute_api` 401 重发（§3.2 六步）。Case 路径复用同一逻辑（§3.3）。
6. 公开 `auth` 九键（§4.1），`_public_auth` 与 `public_auth` 同步；版本 `0.5.0`；CHANGELOG 标注破坏性变化。
7. 测试（用 `client_factory` + MockTransport，`_now` 可替换）：
   - 设计稿 §6 第 31–43 条每条至少一个单测（33 用替换时钟，不 sleep）；
   - 32：缓存命中后业务响应回显 token 与回显登录密码 → 均为 `***`；
   - 并发 43：`asyncio.gather` 两个执行，登录端点计数 = 1；
   - generation：A 执行 401 作废前，B 已写入新条目 → A 的作废不删 B 的条目；
   - `login_cache_max_entries=0` → 行为与 0.4.0 一致（每次登录）；
   - 401 重发：POST + json body 两次请求字节相同；公开 request/response 只有第二次；第一次 401 的响应体（放入一个假秘密）不出现在公开结果与日志中；
   - `repr(LoginCache)`、`repr(AuthTable)`、`repr(LoginAuth)` 不含 scope/指纹/值；
   - 日志 caplog：只出现 §0.1-2 允许的字段。
8. 文档：README / 接口文档补“跨执行缓存与 401 重发”，含 §3.4 单步最长耗时。

## 禁止
修改 Control 仓；落盘缓存；公开 `expires_at`；改 `login_failed` reason 枚举；推送。

## 验收命令
`uv run ruff check .`、`uv run ruff format --check .`、`uv run mypy src`（若仓库已配置）、`uv run pytest -q`。

## 回传格式
分支、commit SHA（含父）；改动文件；测试数（0.4.0 为 401，新增多少）；设计稿逐节对照（做了 / 偏差及理由）；未解决问题。
````

## 2. Agent 2：Control 后端 + 前端

````markdown
你负责 Phase 3.3d 的 Control 改动。仓库 `/Users/jiavin/Developer/products/testplat1/testplat-control`。
从 `origin/main @ 2dad704` 新建 worktree `../worktrees/phase3-3d-control`，分支 `codex/phase3-3d-control`。

## 权威输入
设计稿 §4、§5；本手册 §0.1。Engine 用 Agent 1 已审查 commit `9fef3a82e8269b2b30e536b31b9347cc7683e092`。

## 必须交付
1. `pyproject.toml`：`testplat-engine>=0.5.0,<0.6`；`[tool.uv.sources]` 临时
   `git = "file:///Users/jiavin/Developer/products/testplat1/testplat-engine", rev = "<SHA>"`，注释“合并前必须改为远程精确锁定”。不得 path editable。更新 `uv.lock`。
2. `auth_loader.load_auth(..., cache_scope=None)` → `AuthTable(cache_scope=...)`；`execution_service._load_auth` 传 `f"{project.id}:{project.env}"`。
3. `reauth_on_401` 加入 `_LOGIN_KEYS`，非布尔 → 422（消息不含值）。
4. 执行记录读取兼容旧 `auth`（缺两键 → `null` / `false`）。
5. OpenAPI 重新生成，diff 写入 `docs/phase3_3/`。
6. 前端：`types.ts`、`AuthSummary`、`LoginProcessPanel` / 执行历史按设计稿 §5-5；新增单测覆盖 cache / reauthenticated / 旧记录三种。
7. 文档：设计稿 §5-6。
8. 测试：loader（cache_scope 传入、reauth_on_401 校验）、service（项目 id + env 拼 scope）、旧记录兼容。

## 禁止
修改 Engine 仓；在 Control 侧做任何缓存；推送。

## 验收命令
Control 根：`uv run ruff check .`、`uv run ruff format --check .`、`uv run pytest -q`；
`frontend/`：`npm run lint`、`npm test -- --run`、`npm run build`。

## 回传格式
分支、commit SHA（含父）；改动文件；测试数（3.3c 为 Control 330 / 前端 106）；OpenAPI diff 摘要；偏差；未解决问题。
````

## 3. Agent 3：集成

````markdown
你负责 Phase 3.3d 集成验收。从 Agent 2 已审查 commit `6cb508a`（Control，`codex/phase3-3d-control`） 新建 worktree `../worktrees/phase3-3d-integration`，分支 `codex/phase3-3d-integration`。

## 必须交付
1. `scripts/local_target_server.py`：
   - 登录端点计数（按身份），`/__inspect` 返回计数；
   - 吊销开关：`/__inspect/config {"revoke_current_token": true}` → 已签发 token 全部失效，业务端点对失效 token 返回 401；
   - `always_401` 开关（第 39 条）；登录失败开关（第 41 条）；
   - 新签发 token 继续写入 issued-secrets 文件。
2. 新矩阵 `scripts/login_cache_real_engine_matrix.py`：设计稿 §6 第 31–43 条，行号 L31–L43。
   - 33 用短 ttl 真实等待（≤ 5 秒）；
   - 37 用两个项目；
   - 43 用两个并发 HTTP 请求打 Control。
   接入 `scripts/run_real_engine_matrix.sh`，输出目录 `frontend/screenshots/phase3_3d-real-engine/`。
3. 泄密搜索（第 44 条）：动态目标包含所有签发过的 token（含被吊销的旧 token）；后端日志额外断言不含 `fingerprint`、`cache_scope` 原值。
4. 截图：执行详情“复用登录”与“已重新登录并重发”各 390 / 1440 两张，DOM 搜索证据。
5. 3.3c 遗留小修：设计稿 §9 三条。
6. 第 45 条：3.3a 28/28、3.3b 25/25、3.3c 12/12、Phase 3 回归、浏览器全部不变。

## 禁止
修改 Engine 与 Control `src/`（发现问题停下报告）；推送。

## 回传格式
分支、commit SHA（含父）；矩阵逐行结果；泄密证据摘要（动态目标数、意外命中）；截图清单；偏差；BLOCKED 项（Engine 远程锁定需用户推送授权）。
````

## 4. 主线程审查清单

- Agent 1：
  - 缓存命中路径的替换集合顺序（读代码 + 跑第 32 条单测）；
  - generation 作废竞态测试真实存在且有意义；
  - 指纹包含 host_url、params、连接（插件档）、SecretStr 值，且改任一项都失效；
  - reveal AST 白名单只多 `login_cache.py` 一处；
  - 401 第一次响应不进公开结果、不进日志（只状态码）；
  - `max_entries=0` 回到 0.4.0 行为；Engine 全量测试复跑。
- Agent 2：scope 拼接、旧记录兼容、前端三态、OpenAPI diff 只多两个字段与 `reauth_on_401`。
- Agent 3：临时目录复跑完整矩阵；核对计数断言不是“≥”而是精确值；泄密动态目标包含被吊销 token。
- 发布顺序：Engine PR → 用户授权打 `v0.5.0` → Control 改远程精确锁定 → Control PR。
