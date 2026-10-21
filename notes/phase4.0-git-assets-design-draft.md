# Phase 4.0 设计稿：Git 用例仓库为数据源

> **2026-10-15 后续需求覆盖说明**：本文记录 4.0 的历史交付范围。4.1 起环境、host、变量与测试账号改为 Web / 数据库维护，登录流程保留 Git；项目可包含多个被测系统；后期支持 Web 编辑 case。旧项目 api / scenarios 只读参考，按小功能逐步重写。下文 D1/D2/D3/D5、环境来源与秘密来源不能作为 4.1 的要求。详见 [当前需求基线](phase4-roadmap.md) 与 [4.1 设计稿](phase4.1-design-draft.md)。

日期：2026-10-14 · 状态：v1（方向与 §9 三点均已批准）
基线：Engine `v0.5.0` = `5231fb4`；Control main = `e5ebb29`（#40）。

## 1. 已批准方向

| # | 裁决 |
|---|---|
| D1 | 项目 = Git 管理的用例仓库；平台多人共用，只做“选 → 运行 → 看结果”，**不在 Web 新建项目、不编辑用例** |
| D2 | 资产与配置放第三个独立仓库 `testplat-assets`，不放 Control / Engine |
| D3 | 旧项目 `testplat-new` 只读，只取接口基础信息（host、方法、路径、字段、成功判断、登录方式）；不搬资产、不照搬场景结构 |
| D4 | 资产按新 Engine 0.5 **现有能力**编写，先走通架构流程；case 格式随后续功能迭代再丰富 |
| D5 | 秘密不进任何仓库：资产里只写 `${secret:...}`，值在平台服务器 |

## 2. 本批目标流程

选仓库 → 同步 → 选项目 → 选环境 → 浏览 api / case → 运行 → 看结果；历史记录显示 env 与 commit。

**Engine 不改**（Engine 只认一个资产目录和配置目录，快照目录满足）。

## 3. `testplat-assets` 仓库约定

```text
testplat-assets/
├── testplat.yaml              # 仓库清单（必需）
├── projects/
│   └── <project>/
│       ├── api/**.yaml        # Engine 0.5 api 格式（必须有 id）
│       ├── scenarios/**.case.yaml
│       └── config/
│           ├── envs/<env>.yaml
│           └── auth/<env>.yaml   # 现有 auth_loader 格式，秘密只写 ${secret:}
└── README.md                  # 写法说明（随 Engine 能力更新）
```

清单 `testplat.yaml`：

```yaml
schema_version: "1"
projects:
  gsb:
    name: GSB 商家端
    assets: projects/gsb            # 相对仓库根
    config: projects/gsb/config
    default_env: test               # 可选
```

- 项目键只允许 `[a-z0-9_-]{1,64}`；路径必须是仓库内相对路径（拒绝绝对路径、`..`、符号链接逃逸）。
- 可选环境 = `config/envs/*.yaml` 的文件名（不含 `_` 开头的共享文件）。
- 清单非法 → 该仓库同步失败并保留上一个可用 commit，不影响其它仓库。

## 4. Control：仓库与快照

### 4.1 仓库登记（服务端配置，非 Web）

`TESTPLAT_REPOS_FILE` 指向服务端 YAML（不进任何仓库）：

```yaml
data_dir: /var/lib/testplat/repos   # 镜像与快照存放处
repos:
  main:
    url: git@github.com:<org>/testplat-assets.git   # 或本地路径（file://）
    branch: main
```

- 仓库键同项目键规则。Web 上**不能**增删改仓库。
- Git 凭据走服务器自身 SSH agent / credential helper；配置里不写 token。

### 4.2 同步

- 镜像：`{data_dir}/{repo}/mirror.git`（`git clone --mirror` / `git fetch --prune`）。
- 同步 = fetch → 解析 `branch` 当前 SHA → 校验清单 → 更新“当前 commit”。
- 触发：Web“同步”按钮（`POST /repos/{repo}/sync`）+ 启动时同步一次；定时同步留到后续。
- 同一仓库同步串行（进程内锁）；超时 60 秒；失败保留旧 commit，返回原因类别（`network` / `auth` / `invalid_manifest` / `timeout`），不回显 Git 原始输出（可能含 URL 凭据）。
- 所有 git 调用用参数列表（不经 shell），`GIT_TERMINAL_PROMPT=0`。

### 4.3 快照（多人并发的关键）

- 运行前把“当前 commit”物化为只读快照 `{data_dir}/{repo}/snap/{sha}/`（`git archive` 或 `worktree --detach`），已存在则复用。
- 一次执行全程使用同一个 SHA；同步只改变**后续**执行用哪个 SHA，不影响正在跑的。
- 浏览资产树也读当前 SHA 的快照，所以“看到的”=“将要运行的”。
- 快照清理：保留最近 N=10 个 SHA 且无执行在用的才删（本批可只做“保留最近 10 个”）。

## 5. Control：数据模型与接口

### 5.1 项目来自清单

- `projects` 表不再由 Web 写入；改为同步时按清单 upsert，主键 = `"{repo}/{project}"`（稳定、可读），存 name、repo、当前 commit、可选环境、`default_env`。
- 清单里消失的项目标记 `archived`（不删，历史记录仍可查）。
- `assets_root` / `config_root` / `env` 三列移除（由快照 + 运行参数推导）。
- **无迁移**：仓库没有 alembic，当前是 `create_all`。本批直接改模型，旧 SQLite 文件不兼容 → 启动检测到旧表结构时报清晰错误，提示删除数据库文件（开发期数据，可接受）。用户确认见 §9-3。

### 5.2 执行记录

`execution_records` 新增：`env`、`repo`、`commit_sha`（40 位）、`operator`（可空字符串，4.0 恒为 null，4.1 平台登录后填写，见 `notes/phase4-roadmap.md` N10）。历史列表与详情显示 env + 短 SHA。

### 5.3 接口变化

| 接口 | 变化 |
|---|---|
| `POST/PUT/DELETE /projects` | **删除** |
| `GET /projects` / `GET /projects/{id}` | 只读；带 `repo`、`commit_sha`、`envs`、`default_env`、`archived` |
| `GET /repos` | 新增：仓库键、分支、当前 SHA、最近同步时间与结果 |
| `POST /repos/{repo}/sync` | 新增：同步，返回新 SHA 或失败类别 |
| 资产树 / 资产详情 / 参数 / case 详情 | 路径不变，内部改读当前 SHA 快照 |
| `POST .../run`（api 与 case） | 请求体新增必填 `env`（须在 `envs` 内，否则 422）；响应带 `commit_sha` |
| 执行记录列表 / 详情 | 带 `env`、`commit_sha`；列表可按 `env` 过滤 |

- `project_id` 含 `/`，路由用 `{project_id:path}` 或改为 `repo` + `project` 两段，Agent 按 FastAPI 习惯选一种并在回传说明（推荐两段：`/repos/{repo}/projects/{project}/...`）。
- 3.3d 缓存作用域改为 `f"{repo}/{project}:{env}"`（不含 SHA：同一身份配置跨 commit 未变时可复用；配置变了指纹自然失效）。

### 5.4 前端

- 项目列表：去掉新建 / 编辑 / 删除；显示仓库、短 SHA、最近同步时间，“同步”按钮（同步中禁用，失败显示类别）。
- 项目详情：环境下拉（默认 `default_env`），运行 api / case 时带上。
- 结果与历史：显示 env、短 SHA。

## 6. 首批资产（`testplat-assets` 初版）

**用户已圈定（2026-10-14）**：只做 gsb —— 登录 + `account/balance` + `order/search_product`，外加一条 smoke case（balance → search_product）。细节见手册 §1。以下为通用约束：

- 每个业务系统 3–6 个核心接口（含登录），只用 Engine 0.5 现有能力；
- 每个系统 1–2 条顺序 steps 的简单 case；
- 一个 `test` 环境；秘密全部 `${secret:}`。
- README 写清当前支持的写法与“暂不支持”（数据驱动、suites、UI 步骤）。

## 7. 验收矩阵（Phase 4.0）

1. 服务端配置一个本地 `file://` 仓库 → 启动后项目列表出现清单中的项目，Web 上无新建入口。
2. 资产仓库新提交一条 case → 点同步 → 树中出现，SHA 更新。
3. 同步失败（仓库地址不可达 / 清单非法）→ 保留旧 SHA，错误只含类别。
4. 执行中触发同步并换 SHA → 正在跑的执行仍用旧快照，记录 `commit_sha` 为旧值。
5. 运行时选 `env`；非法 env → 422；不同 env 用不同 hosts。
6. 执行记录列表与详情显示 env + 短 SHA；按 env 过滤。
7. 清单路径逃逸（`..`、绝对路径、符号链接）→ 拒绝。
8. 秘密：资产仓库与快照中无秘密值；Git 错误输出不进响应与日志。
9. 3.3a–3.3d 鉴权矩阵改用新运行接口后结果不变。
10. 前端 390 / 1440 截图：项目列表（含同步）、环境选择、历史中的 SHA。

## 8. 不在本批

定时同步、Webhook、多分支 / 选历史 commit 运行、Web 上编辑用例、数据驱动 case、suites、UI 步骤、旧格式兼容。
后续批次已于 2026-10-15 调整：4.1 优先平台登录 / 操作人 / Web 配置，分享与脱敏开关后移；运行排队（4.2）见 `notes/phase4-roadmap.md`。
部署环境：阿里云内网服务器，Git 远程为自建 GitLab（SSH 密钥放服务器）。

## 9. 用户裁决（2026-10-14）

1. 平台服务器**能访问 Git 远程** → 本批支持远程 URL（SSH / HTTPS）与 `file://`；测试一律用本地 `file://` fixture 仓库，不连外网。
2. 自建 GitLab 的 `testplat-assets` 仓库**由用户自建**；Agent 1 只在本地 `git init` 并提交，推送由用户授权。
3. **同意**不做数据库迁移，旧开发库删除重建。

## 10. 执行拆分（串行，Sonnet）

| Agent | 内容 | 依赖 |
|---|---|---|
| 1 | `testplat-assets` 初版（首批接口与 case、清单、README） | 用户圈定首批 |
| 2 | Control 后端：仓库同步、快照、模型与接口（§4、§5.1–5.3） | 无（用本地 fixture 仓库测试），可与 1 并行 |
| 3 | Control 前端（§5.4） | 2 通过审查 |
| 4 | 集成：矩阵 §7 + 用 Agent 1 的真实资产仓库跑通 | 1、3 通过审查 |
