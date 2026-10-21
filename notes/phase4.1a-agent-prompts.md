# Phase 4.1a Agent 提示词：Web 配置与 GSB 冒烟闭环

权威设计：`notes/phase4.1-design-draft.md`（v1）+ `notes/phase4-roadmap.md`（2026-10-15）。
> 所有 Agent 只提交本地 commit，不推送、不建 PR、不打 tag、不合并。提交备注用中文。

## 0. 基线与分工

- Engine：`v0.5.0` = `5231fb4`，**只读，不改**（4.1a 不扩步骤 `account`；那是 4.1b）。
- Control：`origin/main @ bb74cb2`（#41 Phase 4.0）。一律从精确 SHA 建 worktree。
- Assets：`testplat-assets @ 80ad73b`（可本地 commit，不推送除非用户授权）。
- 旧项目 `/Users/jiavin/Developer/products/testplat-new` **只读**；不得复制旧 YAML 原文。

| Agent | 内容 | 依赖 |
|---|---|---|
| 1 | Control 后端：平台注册/会话、环境/账号表、配置解析、接入 run | 无，先行 |
| 2 | Assets：提取 `_profiles.yaml`，去掉环境变量秘密引用 | 可与 1 并行；集成前需就绪 |
| 3 | Control 前端：登录页、环境/账号管理、运行页接 DB 环境 | 1 通过审查 |
| 4 | 集成：假服务矩阵 A1–A13 + 真实资产配置跑通 + 截图 | 1、2、3 通过审查 |

## 0.1 全体硬约束

1. 秘密不进 Git、不进日志、不进 HTTP 错误原文、不进 ConfigChange 明细；凭据变更只记「已修改」。禁止 `logger.exception` / `exc_info=True` / 把异常原文回给客户端。
2. 平台用户密码只存哈希，不可查看；被测凭据用认证加密库加密，主密钥 `TESTPLAT_MASTER_KEY`；缺失/错误明确报配置错误，不自动生成新密钥。
3. 会话：同源 HttpOnly、SameSite=Lax cookie，只存 token 哈希，7 天；变更接口校验 Origin；会话不进 Engine AuthTable。
4. 不改 Engine；不引入 Alembic；启动遇不兼容表结构给出删库说明，**禁止自动清空数据库**。
5. 运行配置解析 Module：Control 构造 environment + AuthTable；Engine 继续登录/缓存/HTTP；禁止 Engine 读平台库，禁止 Control 另写 case 执行器。
6. 每个 Agent 对完整 diff 做中文 comment pass，只写说明约束的注释。
7. 验收命令保持既有门禁（ruff / pytest / frontend lint+test+build）；新增用例覆盖设计稿 §8.1 中本 Agent 可测部分。

---

## 1. Agent 1：Control 后端

````markdown
你负责 Phase 4.1a Control 后端。仓库 `/Users/jiavin/Developer/products/testplat1/testplat-control`。
从 `origin/main @ bb74cb2` 新建 worktree `../worktrees/phase4-1a-control-backend`，分支 `codex/phase4-1a-control-backend`。

## 权威输入
- `notes/phase4.1-design-draft.md` §3–§6、§8.1、§9
- `notes/phase4-roadmap.md` N7/N8/N10/N12/N15/N16
- 先读：`database/models.py`、`config.py`、`main.py`、`services/env_loader.py`、`services/auth_loader.py`、`services/execution_service.py`、`services/project_access.py`、`api/run.py`、`api/execution.py`、现有鉴权相关测试

## 必须交付

### A. 平台用户与会话
1. 表 `PlatformUser`、`PlatformSession`（设计稿 §3.1）。
2. `POST /auth/register`、`POST /auth/login`、`POST /auth/logout`、`GET /auth/me`。
3. 配置/执行类接口未登录返回 401；health / register / login 无会话依赖。
4. 现阶段所有登录用户同等能力；无超管字段。

### B. 环境与测试账号持久化
1. 表 `ProjectEnvironment`、`TestAccount`、`ConfigChange`（§3.1）。
2. 唯一键：环境 `(project_id, env)`；账号 `(project_id, env, account_name)`。
3. 环境名、账号名保存后不可重命名；支持新增、编辑、停用（enabled）。
4. 账号名规则 `^[a-z][a-z0-9_]*$`，保留 `none`；展示名/业务角色可用中文。
5. 接口（前缀 `/repos/{repo}/projects/{project}`）：
   - `GET/POST .../environments`
   - `GET/PUT .../environments/{env}`（PUT 带 revision，冲突 409）
   - `GET .../login-profiles`（当前资产 commit 下流程名与所需凭据字段名，不返回凭据）
   - `GET/POST .../environments/{env}/accounts`
   - `GET/PUT .../environments/{env}/accounts/{name}`（PUT 带 revision）
   - `GET .../environments/{env}/accounts/{name}/credentials`（显式查看；禁止缓存头）
6. 账号/环境修改同一事务递增环境 revision；ConfigChange 记录操作人与字段名，凭据不落原值。
7. Project.envs / default_env 若仍作响应字段，由 Web 环境查询派生，**Git 同步不得覆盖** Web 环境/账号/启停。

### C. 登录流程与配置解析
1. 支持读取资产 `config/auth/_profiles.yaml`（设计稿 §4.1 形状）；提取纯数据构造入口，复用 `auth_loader` 校验与 SecretStr。
2. 对每个启用账号：按所选 login_profile 解析 `${secret:字段名}` → 该账号凭据字典；AuthTable 的 identity **用账号名**（如 `gsb`），禁止混成全局字典或写回 `os.environ`。
3. 运行配置解析 Module：入参项目+环境+固定 commit → 返回 Engine environment、AuthTable、有效超时/参数、可持久化非秘密摘要。
4. 参数浅层覆盖：请求 params > 环境 case_params > case 资产 params。
5. HTTP 超时：请求显式 timeout_seconds > 环境 timeout_seconds > 默认 30；区分「省略」与「显式 30」。
6. api_hosts 只来自数据库环境。
7. 未引用账号未配全不阻断；被引用账号缺失/停用 → 预检失败（复用 UnavailableAuth 思路）；**禁止回退** `TESTPLAT_GSB_*`。
8. `cache_scope` 继续 `repo:project:env` 编码（与现实现一致）；身份为账号名。

### D. 执行记录增量
1. 复用 operator（写入当前平台用户）；增加 `config_revision`、`account_revisions`、非秘密配置快照；保留 env、repo、commit_sha。
2. 历史详情不读「当前」环境补全。

### E. 配置与安全
1. `TESTPLAT_MASTER_KEY`（及会话相关设置）经 pydantic-settings；密钥问题明确报错。
2. 启动检测不兼容表 → 清晰说明需显式删库；不自动删。
3. OpenAPI 写入 `docs/phase4_1a/`，附相对 phase4_0 的 diff 摘要。
4. 文档：更新 README 中「环境/秘密来自 Git/服务器」相关表述；新增 `docs/phase4_1a/` 说明注册登录、环境账号、主密钥、revision 冲突。

### F. 测试
1. 假服务证明：host、超时、参数、登录凭据字段真实进入请求。
2. 覆盖 §8.1 中后端可测项，至少：A1（会话）、A2（项目隔离）、A3（新建 pre）、A5（改配置生效）、A6（改密码不命中旧缓存指纹）、A7（运行中编辑不影响已开始执行）、A8（同步不覆盖 Web）、A9（未引用不阻断 / 引用缺失失败）、A10（历史不变）、A11（409）、A12（重启后密文仍可用、日志无明文）、A13 回归既有 Engine 契约相关 Control 测。
3. 旧 auth_loader 历史格式夹具保留兼容；平台管理账号路径不读服务器 `TESTPLAT_GSB_*`。

## 禁止
修改 Engine、前端、`testplat-assets`（除非夹具内临时仓库）；推送；自动删库；把完整凭据字典序列进列表/摘要/错误响应。

## 验收命令
`uv sync --extra dev --locked`、`uv run ruff check .`、`uv run ruff format --check .`、`uv run pytest -q -rs`。

## 回传格式
分支、commit SHA（含父）；改动文件；表结构与加密方案；login-profiles 解析入口说明；测试数（基线与新增）；OpenAPI diff 摘要；偏差与未解决问题。
````

---

## 2. Agent 2：Assets `_profiles.yaml`

````markdown
你负责 Phase 4.1a 资产调整。仓库 `/Users/jiavin/Developer/products/testplat1/testplat-assets`，基于 `80ad73b` 本地提交，不推送。

## 权威输入
设计稿 §4.1、§4.2；现有 `projects/gsb/config/auth/test.yaml`。

## 必须交付
1. 新增 `projects/gsb/config/auth/_profiles.yaml`：流程名例如 `gsb_login`，内容从现有 gsb 登录提取；`${secret:username}` / `${secret:password}`（字段名，非 `TESTPLAT_GSB_*`）；`host: unified_login`；`source: go2` 保持字面量。
2. 处理旧 `config/auth/test.yaml`：改为指向说明「运行时凭据由 Web 维护」或仅保留兼容说明/最小兼容夹层（与 Agent 1 约定一致）；**不得**保留真实秘密或继续要求 `TESTPLAT_GSB_*` 作为平台主路径。
3. `config/envs/*.yaml` 保留为示例模板；README 写明：运行时环境/账号以 Web 为准，同步不覆盖。
4. API 的 `auth: gsb` 与 smoke case **保留**（4.1a 仍靠账号名 `gsb` 对接）。
5. 更新 README：登录流程、Web 凭据字段、不再使用的环境变量名。
6. 自检：`scripts/validate.py`（或等价）在 Engine 0.5 下仍能加载；若 validate 依赖旧 auth 文件，按 Agent 1 兼容策略调整并说明。

## 禁止
写入任何账号密码；推送；改 Engine/Control。

## 回传格式
commit SHA；目录树；profiles 内容要点；与旧 test.yaml 关系；validate 输出。
````

---

## 3. Agent 3：Control 前端

````markdown
你负责 Phase 4.1a Control 前端。从 Agent 1 已审查 commit 新建 worktree `../worktrees/phase4-1a-control-frontend`，分支 `codex/phase4-1a-control-frontend`。

## 权威输入
设计稿 §6、§8.1；Agent 1 的 `docs/phase4_1a/` OpenAPI。

## 必须交付
1. 注册 / 登录 / 注销；未登录引导登录；顶栏显示当前操作人。
2. 项目详情：环境管理（新增/编辑/停用；hosts、超时、case_params）；测试账号管理（账号名、展示名、业务角色、login_profile、凭据；密码可点显示/编辑）。
3. revision：保存带 revision；收到 409 提示刷新后再保存。
4. 运行页：环境下拉来自该项目**启用的数据库环境**；无环境时提示去配置；账号缺失提示账号名与环境。
5. 执行结果/历史：显示操作人、env、commit 短 SHA、config_revision（若有）。
6. 不用 Monaco、配置继承树、拖拽编辑器、单独「测试登录」按钮。
7. 单测覆盖：登录态、环境 CRUD、账号凭据查看、409、运行带 env、未登录 401 引导。

## 禁止
修改后端业务逻辑与 Engine；推送。

## 验收命令
`frontend/`：`npm run lint`、`npm test -- --run`、`npm run build`。

## 回传格式
commit SHA；改动文件；测试数；偏差。
````

---

## 4. Agent 4：集成

````markdown
你负责 Phase 4.1a 集成。从 Agent 3 已审查 commit 新建 `codex/phase4-1a-integration`（worktree `../worktrees/phase4-1a-integration`）。

## 必须交付
1. 矩阵脚本覆盖设计稿 §8.1 A1–A13；本地假服务证明配置进入请求；不把「真实 GSB 可达」写成自动化必过。
2. 用 Agent 2 资产 + Web API 建 test（及可选 pre）环境、账号名 `gsb`、login_profile `gsb_login`，跑既有 API / smoke；目标不可达时允许 transport 失败，但记录须含 operator、env、commit、config_revision。
3. 同步回归：改 Web 配置后同步 Git，Web 数据不被覆盖。
4. 截图 390/1440：登录、环境管理、账号管理、运行选环境、历史含操作人/版本 → `frontend/screenshots/phase4_1a/`。
5. 泄密检查：响应、日志、SQLite 中无明文密码/token；列表接口不批量返回密码。

## 禁止
修改 Engine 与 Control `src/`、前端 `src/`（发现问题停下报告）；推送。

## 回传格式
分支与 SHA；A1–A13 逐条结果；端到端证据；截图清单；BLOCKED 项。
````

---

## 5. 主线程审查清单

- Agent 1：会话与 401；加密/哈希分家；revision 409；同步不覆盖 Web；不回退服务器秘密；运行冻结配置；OpenAPI/文档齐全。
- Agent 2：`_profiles.yaml` 字段名秘密；无真实凭据；`auth: gsb` 仍在。
- Agent 3：登录闭环；环境/账号表单；运行下来自 DB。
- Agent 4：假服务矩阵 + 不宣称真实 GSB 业务通过；泄密检查。
- 发布：Control PR、assets 推送均需用户另行授权。4.1b 另开批次，本批不改 Engine schema。
