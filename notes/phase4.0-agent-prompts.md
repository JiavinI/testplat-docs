# Phase 4.0 Agent 提示词：Git 用例仓库为数据源

> **2026-10-15 使用范围**：本文只记录 4.0 执行任务，不用于 4.1。环境、测试账号与凭据的来源已调整为 Web / 数据库，登录流程保留 Git；新的需求、拆分与验收以 [Phase 4 基线](phase4-roadmap.md) 和 [4.1 设计稿](phase4.1-design-draft.md) 为准。

权威设计：`notes/phase4.0-git-assets-design-draft.md`（v1，已批准）。
> 所有 Agent 只提交本地 commit，不推送、不建 PR、不建远程仓库、不打 tag、不合并。

## 0. 基线与分工

- Engine：`v0.5.0` = `5231fb4`，**只读，不改**。
- Control：`origin/main @ e5ebb29`（#40）。一律从精确 SHA 建 worktree。
- 旧项目 `/Users/jiavin/Developer/products/testplat-new` 只读；只有 Agent 1 可参考，且只用 `notes/phase4.0-legacy-api-inventory.md` 已整理的信息，不得复制旧 YAML。

| Agent | 模型 | 内容 | 依赖 |
|---|---|---|---|
| 1 | Sonnet | `testplat-assets` 初版（本地仓库） | 用户圈定首批接口 |
| 2 | Sonnet | Control 后端：仓库同步、快照、模型与接口 | 无，可先行 |
| 3 | Sonnet | Control 前端 | 2 通过审查 |
| 4 | Sonnet | 集成：矩阵 §7 + 真实资产仓库跑通 | 1、3 通过审查 |

## 0.1 全体硬约束

1. git 一律 `subprocess` 参数列表调用，禁止 `shell=True`；环境 `GIT_TERMINAL_PROMPT=0`；每次调用有超时。
2. Git 原始 stdout/stderr 可能含带凭据的 URL：不进 HTTP 响应、不进日志、不进异常消息；对外只给失败类别。日志只记仓库键、类别、短 SHA。
3. 仓库 URL 视为敏感：API 响应不返回 URL（只返回仓库键与分支）。
4. 清单与路径：拒绝绝对路径、`..`、指向快照外的符号链接；读取资产前做 `resolve()` + 前缀校验。
5. 测试不连外网：用 `tmp_path` 里 `git init` 的 fixture 仓库与 `file://` URL。
6. 不改 Engine；不新增运行时依赖（不引入 GitPython）；秘密不进任何仓库与快照。
7. 每个 Agent 对完整 diff 做中文 comment pass，只写说明约束的注释。

## 2. Agent 2：Control 后端（可先行）

````markdown
你负责 Phase 4.0 Control 后端。仓库 `/Users/jiavin/Developer/products/testplat1/testplat-control`。
从 `origin/main @ e5ebb29` 新建 worktree `../worktrees/phase4-0-control-backend`，分支 `codex/phase4-0-control-backend`。

## 权威输入
设计稿 §3、§4、§5.1–5.3、§7、§9；本手册 §0.1。先读现有：
`database/models.py`、`api/projects.py`、`api/assets.py`、`api/cases.py`、`api/run.py`、`api/execution.py`、
`services/asset_service.py`、`services/case_asset_service.py`、`services/execution_service.py`（_load_env / _load_auth / cache_scope）、`config.py`、`main.py`。

## 必须交付
1. 服务端配置：`TESTPLAT_REPOS_FILE`（pydantic-settings）→ 解析 §4.1 YAML；键名、分支、`data_dir` 校验；未配置时启动正常、仓库列表为空。
2. `services/repo_sync.py`：mirror clone/fetch、解析分支 SHA、校验清单（§3）、物化快照（§4.3，推荐 `git archive | tar` 或 `git worktree add --detach`，二选一并说明）、每仓库 `asyncio.Lock`、60 秒超时、失败类别 `network|auth|invalid_manifest|timeout`（auth 类别靠 git 退出信息关键词判断即可，判不出归 network）。保留最近 10 个快照。
3. `services/manifest.py`：解析 `testplat.yaml`、项目键规则、路径逃逸检查、环境列表（`config/envs/*.yaml`，忽略 `_` 开头）。
4. 模型（§5.1–5.2，无迁移）：
   - `repos` 表（键、分支、当前 SHA、最近同步时间、最近结果类别）；
   - `projects` 主键改 `"{repo}/{project}"`，存 repo、project 键、name、envs(JSON)、default_env、archived；删除 `assets_root/config_root/env`；
   - `execution_records` 加 `env`、`repo`、`commit_sha`、`operator`（可空，本批恒为 null，API 响应带出）；
   - 启动检测旧表结构 → 清晰报错提示删除数据库文件。
5. 接口（§5.3）：路由采用 `/repos/{repo}/projects/{project}/...` 两段式；删除项目 POST/PUT/DELETE；新增 `GET /repos`、`POST /repos/{repo}/sync`；资产 / case / 参数改读当前 SHA 快照；运行接口请求体加必填 `env`（不在 envs → 422）、响应带 `commit_sha`；执行一旦开始全程用同一快照；执行记录列表支持 `env` 过滤。
6. `cache_scope` 改为 `f"{repo}/{project}:{env}"`。
7. 启动时对每个已配置仓库同步一次（失败不阻塞启动，只记类别）。
8. OpenAPI 重新生成写入 `docs/phase4_0/`，附相对 `docs/phase3_3/openapi-phase3_3.json` 的 diff。
9. 测试改造：现有依赖“POST /projects 建项目”的测试改为用 fixture 仓库 + 同步建立项目；提供共用 pytest fixture（`tmp_path` 下 `git init` 资产仓库，提交 api / case / envs / auth，返回 `file://` URL）。3.3a–3.3d 的鉴权测试必须全部保留语义并通过。
10. 新增测试覆盖设计稿 §7 第 1–9 条中后端可测部分；特别是：
    - 第 4 条：执行进行中（用可控的 fake engine 阻塞）触发同步换 SHA → 记录仍为旧 SHA；
    - 第 7 条：`..`、绝对路径、快照内符号链接指向外部 → 拒绝；
    - 第 8 条：同步失败时响应与 caplog 中不出现仓库 URL、git stderr 片段。
11. 文档：更新 Control `README.md` 中“新建项目 / assets_root / config_root”相关用法为“服务端登记仓库 → 同步 → 选环境运行”；`docs/phase4_0/repos-and-snapshots.md`（服务端配置样例、同步语义、快照与并发、失败类别、删除旧库说明）；`scripts/` 中依赖旧项目接口的脚本列出清单（交给 Agent 4，不必本 Agent 修）。

## 禁止
修改 Engine、前端；引入 GitPython 等新依赖；连外网；推送。

## 验收命令
`uv sync --extra dev --locked`、`uv run ruff check .`、`uv run ruff format --check .`、`uv run pytest -q -rs`。

## 回传格式
分支、commit SHA（含父）；改动文件；快照方案选择与理由；测试数（基线 337，新增 / 改写多少）；OpenAPI diff 摘要；受影响 scripts 清单；偏差与未解决问题。
````

## 1. Agent 1：`testplat-assets` 初版（待用户圈定首批）

````markdown
你负责 `testplat-assets` 初版。在 `/Users/jiavin/Developer/products/testplat1/testplat-assets` 本地 `git init`（分支 main）。
远程仓库由用户自建，你**不得** `git remote add` 之外的任何远程操作，也不推送。

## 权威输入
设计稿 §3、§6；接口清单 `notes/phase4.0-legacy-api-inventory.md` 中**用户圈定的首批（只做这些）**：

- 项目 `gsb`（单个项目）。
- 身份：gsb 登录（清单 §3：`unified_login` host，`GET /site/login`，账号字段走 query，另带 `source`；token 取 `$.data.hash` → header `X-TOKEN`，user_id 取 `$.data.userId` → `X-USER-ID`；`validity.ttl_seconds: 14400`）。用 `generic_flow` 单步，登录 host 与业务 host 分开配置；账号密码只写 `${secret:...}`，`source` 先写一个字面量（如 `go2`）并在 README 注明。
- api 2 个：
  - `api.gsb.account.balance`：POST `/api/account/balance`，json，无入参，`$.errorCode == "0"`；
  - `api.gsb.order.search_product`：GET `/api/b2/orders/search/product`，query，入参 `product_keyword` → 参数 `keywords`，`$.errorCode == "0"`。
- case 1 条：`case.gsb.smoke`：balance → search_product（参数在 case 里给一个示例关键字）。
- 环境 `test`：hosts `gsb`（`java.qa.gogoo.cn:8306`，协议以清单为准）与 `unified_login`（`tt.ximgs.net`）。
- 其它接口、系统、流程本批**都不做**。
新格式参考：`testplat-engine/tests/fixtures/login_assets/`、`test_assets/`、Control `docs/phase3_3/control-auth-backend.md`（auth 配置格式）。

## 必须交付
1. `testplat.yaml`（§3 清单）、`projects/<项目>/{api,scenarios,config/envs,config/auth}`。
2. 首批 api：只用 Engine 0.5 已支持字段；每个有 `id`；必填标记按新格式写。
3. 每个项目 1–2 条顺序 steps case。
4. `config/envs/test.yaml`（hosts 用清单里的域名）；`config/auth/test.yaml`，所有秘密 `${secret:...}`。
5. `README.md`：目录约定、api / case / env / auth 写法、当前**不支持**的写法、新增资产的步骤、秘密环境变量名清单（只列名）。
6. 自检：用 Engine 0.5 加载全部资产（写一个 `scripts/validate.py`，只做加载与 case 引用检查，不发请求），在回传中贴结果。

## 禁止
复制旧项目 YAML 原文；写入任何账号、密码、token 值；推送。

## 回传格式
commit SHA；目录树；每项目接口 / case 数；需要的秘密变量名清单；validate 输出；无法用现有能力表达而舍弃的接口及原因。
````

## 3. Agent 3：Control 前端

````markdown
你负责 Phase 4.0 前端。从 Agent 2 已审查 commit `393d2ba`（`codex/phase4-0-control-backend`）新建 worktree `../worktrees/phase4-0-control-frontend`，分支 `codex/phase4-0-control-frontend`。

## 权威输入
设计稿 §5.3、§5.4；Agent 2 的 `docs/phase4_0/` OpenAPI。

## 必须交付
1. API 层与类型按新接口改（两段式路由、`env`、`commit_sha`、`/repos`）。
2. 项目列表：删除新建 / 编辑 / 删除（含 `ProjectFormModal`）；按仓库分组，显示短 SHA、最近同步时间与结果；“同步”按钮（进行中禁用，失败显示类别中文文案）；archived 项目灰显。
3. 项目详情：环境下拉（默认 `default_env`，记住上次选择于 localStorage），运行 api / case 时带 `env`。
4. 结果与历史：显示 env、短 SHA；历史支持按 env 过滤。
5. 单测覆盖：同步成功 / 失败、环境选择传参、旧字段移除、历史 env + SHA 显示。

## 禁止
修改后端、Engine；推送。

## 验收命令
`frontend/`：`npm run lint`、`npm test -- --run`、`npm run build`。

## 回传格式
commit SHA；改动文件；测试数（基线以 Agent 2 交付时为准）；偏差。
````

## 4. Agent 4：集成

````markdown
你负责 Phase 4.0 集成。从 Agent 3 已审查 commit 新建 `codex/phase4-0-integration`。

## 必须交付
1. 矩阵脚本 `scripts/phase4_0_matrix.py`，覆盖设计稿 §7 第 1–10 条；资产仓库用临时目录 `git init` 的副本（`file://`），第 2 / 4 条在矩阵中真实提交新 commit。
2. 把 3.3a–3.3d 的矩阵脚本与 `local_target_server.py` 迁到新接口（两段式路由 + `env`），结果数不变；Agent 2 回传的受影响 scripts 清单全部处理。
3. 用 Agent 1 的 `testplat-assets` 本地仓库（commit `80ad73b`）做一次端到端：服务端配置 → 同步 → 列出项目 → 选 test 环境 → 运行一个 api 与一个 case（目标服务不可达时允许失败，但须是 transport 类失败且记录含 env + SHA）。
4. 截图 390 / 1440：项目列表（含同步）、环境选择、历史 env + SHA，写入 `frontend/screenshots/phase4_0/`。
5. 泄密检查：响应、日志、SQLite 中无仓库 URL 凭据与 git stderr。

## 禁止
修改 Engine 与 Control `src/`、前端 `src/`（发现问题停下报告）；推送。

## 回传格式
分支与 SHA；矩阵逐条结果；旧矩阵迁移后结果数；端到端证据；截图清单；BLOCKED 项。
````

## 5. 主线程审查清单

- Agent 2：git 调用无 shell、带超时；URL / stderr 不外泄（读代码 + caplog 测试）；快照前缀校验与符号链接测试真实有效；执行固定 SHA 的并发测试有意义；旧鉴权测试未被删改语义。
- Agent 1：无秘密值；validate 全部通过；没有照抄旧 YAML。
- Agent 3：创建入口全部移除；env 必传。
- Agent 4：临时目录复跑矩阵；旧矩阵结果数不变。
- 发布：Control PR 由用户授权；`testplat-assets` 推送到用户自建仓库由用户授权。
