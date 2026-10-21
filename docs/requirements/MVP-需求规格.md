# 通用测试平台 MVP 需求规格

> **2026-10-17 平台关系更正**：本需求以 [被测平台、GSB 归属与配置导入](platforms-and-projects.md) 为准。GO2、E3E3 本身也是被测网站平台，且各自拥有自己的 GSB；代发后台、拿货端、GO2 管理后台独立登录。旧文中的单一“项目”范围不能直接代替这些业务关系。账号、密码来源、验收与保护边界见该权威记录；实现尚未定稿，未导入配置。历史删库重建说明不适用于现有预览库。

> **2026-10-15 当前需求**：环境、host、变量和测试账号在 Web 按项目维护；账号按“项目 + 环境 + 账号名”定位。登录流程放 Git。先完成 Web 配置与 GSB 冒烟闭环，再逐步加入步骤账号选择、跨系统场景和后期 Web 编辑 case。平台注册 / 登录记录操作人，现阶段不做超管分级与复杂权限。详见 [需求基线](../../notes/phase4-roadmap.md) 和 [4.1 设计稿](../../notes/phase4.1-design-draft.md)。以下 2026-10-14 说明记录 4.0 当时范围，与本次要求冲突处已被取代；新设计不表示已实现。

> **需求变更（2026-10-14，Phase 4.0）**：项目不再在 Web 上新建 / 编辑 / 删除。项目 = Git 管理的用例仓库 `testplat-assets` 中的一个项目目录；平台多人共用，只做“选仓库 → 同步 → 选项目 → 选环境 → 选 api/case 运行 → 看结果”。环境由运行时选择，执行记录绑定 commit SHA。用例通过 Git 提交维护，不在 Web 编辑。本文 §2.1.1、§3.1、§4.1、§5.1、§8 已按此更新；详细设计见 `notes/phase4.0-git-assets-design-draft.md`。
>
> **2026-10-14 需求复核**：部署、平台登录与操作人、分享链接、脱敏开关、排队、优先级的现行结论见 `notes/phase4-roadmap.md`，与本文 §2.2、§7.3、§9 冲突时以其为准。

## 1. 项目背景与目标

### 1.1 项目定位
构建一个**通用的、可跨公司复用的接口自动化测试平台**,核心特性:
- 通用性: 不依赖特定业务/公司的硬编码逻辑
- 可移植性: 换公司后只需调整配置文件,无需修改平台代码
- 渐进式: 先实现最小可用 demo,在使用中逐步完善功能

### 1.2 参考基线
基于现有的 `/Users/jiavin/Developer/products/testplat-new` 项目:
- 保留其六边形架构设计 (domain/engine/application/adapters 分层)
- 参考其 api / scenarios 的接口、场景步骤、账号切换和数据依赖；按新 Engine 能力逐步重写
- 但需要**去除特定业务绑定**(如 GSB、GO2、e3e3 等项目专属逻辑)

### 1.3 架构拆分原则
- **testplat-control**: 控制仓库,负责 Web UI、任务调度、结果展示、配置管理
- **testplat-engine**: 执行引擎仓库,负责纯粹的测试执行(HTTP 调用、断言、变量处理等)

---

## 2. MVP 核心功能范围

### 2.1 必须实现 (P0)

#### 2.1.1 Web 界面 - 项目管理
- [ ] **项目列表页**: 展示用例仓库清单（`testplat.yaml`）中声明的所有项目（只读）
  - 按仓库分组，显示项目名称、当前 commit（短 SHA）、最近同步时间与结果
  - “同步”按钮：拉取仓库最新提交；**不提供**新建/编辑/删除（仓库地址由服务端配置登记）
  
- [ ] **项目详情页**: 选中项目后进入
  - 左侧树形导航: 展示该项目下的 API 契约和测试场景
    - `assets/api/` 下的接口定义文件 (单接口可直接运行)
    - `assets/scenarios/` 下的测试用例文件 (非 flow 的 case 文件)
  - 右侧内容区: 显示选中资产的元信息和执行按钮
  - 环境下拉：可选环境来自项目内 Web 维护的环境，运行时选择；Git 环境文件仅作为初始化模板候选

#### 2.1.2 单接口运行
- [ ] 点击某个 API 文件后,可以:
  - 查看接口定义 (URL、method、请求体结构、成功判定条件)
  - 填写/编辑请求参数 (支持从历史记录加载)
  - 点击"运行"按钮发起单次调用
  - 实时查看执行结果:
    - HTTP 请求详情 (headers、body)
    - HTTP 响应详情 (status、headers、body)
    - 断言结果 (通过/失败)

#### 2.1.3 场景运行
- [ ] 点击某个 case 文件后,可以:
  - 查看场景元信息 (desc、tags、步骤列表)
  - 点击"运行"按钮执行整个场景
  - 查看执行结果:
    - 各步骤的名称、状态 (通过/失败/跳过)
    - 点击某步骤可展开查看该步骤的请求/响应
    - 场景级断言结果汇总

#### 2.1.4 测试结果展示
- [ ] 执行记录列表
  - 显示历史执行记录 (运行 ID、资产名称、开始时间、耗时、状态)
  - 支持筛选 (按项目/环境/状态)
  
- [ ] 结果详情页
  - 复用 testplat-new 的 HTML 报告能力
  - 展示步骤级的请求/响应内容
  - 失败时高亮失败的断言

#### 2.1.5 Web 运行配置（2026-10-15 新增）
- 自主注册 / 登录并记录操作人；所有登录用户可维护配置。
- 每项目独立新增、编辑、停用环境，维护服务地址、超时和业务参数默认值。
- 按项目、环境和账号名维护多角色测试账号，选择 Git 登录流程，查看 / 编辑凭据。
- 保存后下一次运行使用新配置；已开始的运行固定启动配置，历史保留配置版本与账号名。
- 4.1a 接入现有 GSB 冒烟；4.1b 再增加 case 步骤账号选择。后期支持 Web 修改请求 / 断言，保存方式另定。

### 2.2 暂不实现 (后续迭代)
- UI 自动化 (Playwright 集成)
- 定时任务调度
- 并发执行控制
- 数据库/Redis 连接校验
- 复杂的鉴权策略 (OAuth、多租户等)
- CI/CD 集成

---

## 3. 技术架构设计（早期示意；当前实现与增量方案见 4.1 设计稿）

### 3.1 控制仓库 (testplat-control)

#### 职责边界
- Web 后端 API (推荐 FastAPI)
- 前端 UI (推荐 React + Ant Design / Vue + Element Plus)
- 用例仓库同步（Git fetch）与按 commit 的只读快照；项目由仓库清单派生（只读，无 CRUD）
- 测试资产解析与展示 (读取当前 commit 快照中的 YAML 文件)
- 执行任务调度 (调用 engine 执行器)
- 结果归档与查询

#### 技术栈建议
```
后端: FastAPI + SQLite (存储项目配置和执行记录)
前端: React + TypeScript + Ant Design
通信: REST API / WebSocket (实时推送执行进度)
```

#### 关键数据模型
```python
# 用例仓库（服务端配置登记，Web 只读）
class Repo:
    key: str          # 如 main
    branch: str
    current_sha: str  # 最近一次同步成功的 commit
    last_sync_at: datetime
    last_sync_result: str  # ok / network / auth / invalid_manifest / timeout

# 项目（由仓库清单 testplat.yaml 派生，同步时 upsert）
class Project:
    id: str           # "{repo}/{project}"
    repo: str
    name: str
    envs: list[str]   # 4.1 起由项目内启用的数据库环境派生
    default_env: str | None
    archived: bool    # 清单中已移除的项目，历史仍可查

# 执行记录
class ExecutionRecord:
    id: str
    project_id: str
    env: str          # 运行时选择
    commit_sha: str   # 本次执行所用快照
    asset_type: str  # api / case
    asset_id: str
    status: str  # running/success/failed
    start_time: datetime
    duration: float
    result_path: Path  # 指向结果文件
```

### 3.2 执行引擎 (testplat-engine)

#### 职责边界
- 纯粹的测试执行逻辑 (无 UI、无调度)
- HTTP 客户端封装
- YAML DSL 解析与执行
- 变量渲染、断言引擎
- 结果序列化 (JSON/HTML)

#### 复用 testplat-new 的模块
```
domain/     # 核心领域模型 (Case/Api/Step/Assert)
engine/     # 执行引擎 (executor/resolver/assert_engine)
ports/      # 端口接口 (HttpClient/SecurityContext)
plugins/    # 插件实现 (httpx_client 等)
adapters/filesystem/  # YAML 加载器
```

#### 对外接口 (Python API)
```python
# 执行单个 API
def run_api(
    api_id: str,
    registry: Registry,
    env: str,
    params: dict
) -> ApiResult

# 执行单个 Case
def run_case(
    case_id: str,
    registry: Registry,
    env: str
) -> CaseResult
```

### 3.3 交互流程

```
┌─────────────────┐
│  Web Browser    │
└────────┬────────┘
         │ 1. 选择项目/资产,点击运行
         ▼
┌─────────────────────┐
│  testplat-control   │
│  - 解析项目配置      │
│  - 加载资产文件      │
│  - 调用 engine       │
└────────┬────────────┘
         │ 2. 调用执行接口
         ▼
┌─────────────────────┐
│  testplat-engine    │
│  - 加载 YAML 资产    │
│  - 执行 HTTP 请求    │
│  - 运行断言         │
│  - 生成结果报告     │
└────────┬────────────┘
         │ 3. 返回执行结果
         ▼
┌─────────────────────┐
│  testplat-control   │
│  - 保存执行记录      │
│  - 归档结果文件      │
│  - 推送 UI 更新      │
└─────────────────────┘
```

---

## 4. 用户场景与交互设计

### 4.1 首次使用流程

**步骤 0（管理员，一次性）: 登记用例仓库**
```
在服务端配置文件（TESTPLAT_REPOS_FILE）写入仓库地址与分支
→ 启动平台，自动同步一次
```

**步骤 1: 选择项目与环境**
```
用户访问 http://localhost:8000
→ 项目列表显示仓库清单中的项目（含当前 commit）
→ 如有人刚提交了新用例，点击"同步"
→ 注册 / 登录，进入项目详情页，在 Web 配好 test 环境、服务地址与测试账号
→ 在环境下拉中选择 test；环境与账号数据以平台数据库为准
```

**步骤 2: 运行单接口**
```
左侧树形导航显示:
├── api
│   ├── user
│   │   ├── login.yaml
│   │   └── getUserInfo.yaml
│   └── order
│       └── create.yaml

点击 login.yaml
→ 右侧显示接口定义和参数表单
→ 填写非秘密业务参数；被测系统登录凭据从 Web 维护的测试账号读取
→ 点击"运行"
→ 下方展示请求/响应内容和断言结果
```

**步骤 3: 运行场景**
```
左侧树形导航显示:
├── scenarios
│   ├── user
│   │   └── login_success.case.yaml
│   └── order
│       └── create_order.case.yaml

点击 login_success.case.yaml
→ 右侧显示场景信息 (描述、步骤列表、标签)
→ 点击"运行"
→ 下方展示步骤执行进度和结果
```

### 4.2 界面布局草图

```
┌─────────────────────────────────────────────────────────┐
│  TestPlat - 通用测试平台                    [项目: my-api-tests] │
├─────────────┬───────────────────────────────────────────┤
│             │  接口: user/login                          │
│ 资产树       │  ────────────────────────────────────────│
│             │  URL: POST /api/user/login                │
│ api/        │  请求参数:                                 │
│  user/      │  ┌─────────────────────────────────────┐ │
│   login     │  │ username: [__________________]      │ │
│   getUserInfo│  │ password: [__________________]      │ │
│  order/     │  └─────────────────────────────────────┘ │
│   create    │  [ 运行 ]  [ 保存参数 ]                   │
│             │  ────────────────────────────────────────│
│ scenarios/  │  执行结果:                                 │
│  user/      │  ┌─────────────────────────────────────┐ │
│   login_    │  │ 状态: 成功 ✓                         │ │
│   success   │  │ 耗时: 123ms                          │ │
│  order/     │  │                                     │ │
│   create_   │  │ 请求: {...}                          │ │
│   order     │  │ 响应: {...}                          │ │
│             │  │ 断言: ✓ $.code == 0                  │ │
│             │  └─────────────────────────────────────┘ │
└─────────────┴───────────────────────────────────────────┘
```

---

## 5. 配置文件结构 (通用化改造)

### 5.1 用例仓库与项目清单（取代原“control 管理的项目配置”）
```yaml
# 服务端配置（不进任何仓库）：TESTPLAT_REPOS_FILE
data_dir: /var/lib/testplat/repos
repos:
  main:
    url: git@github.com:<org>/testplat-assets.git
    branch: main
```
```yaml
# 用例仓库根目录 testplat-assets/testplat.yaml
schema_version: "1"
projects:
  gsb:
    name: GSB 商家端
    assets: projects/gsb
    config: projects/gsb/config
    default_env: test
```

### 5.2 运行配置（2026-10-15 修订）

- Git 保存接口、case、登录流程；不含真实账号密码。现有 config/envs/*.yaml 是初始化模板候选，自动首次导入规则尚未裁决。
- Web 管理各项目自己的环境、api_hosts、超时、配置变量，以及按环境区分的测试账号。
- Control 从数据库取得配置、结合固定 commit 的登录流程，构造 Engine 的 environment / AuthTable；Engine 不读平台数据库。
- 被测系统凭据加密存数据库；平台用户密码保存哈希；平台主密钥和 Git 凭据仍由服务器维护。
- 一个项目可配置多个被测系统地址与账号，用于后续跨系统场景；不强制项目等于一个系统。

### 5.3 API 契约 (通用格式)
```yaml
# my-assets/api/user/login.yaml
name: 用户登录
host: main
method: POST
url: /api/user/login
request:
  type: json
  inputs:
    username: 用户名
    password: 密码
  body:
    username: "${username}"
    password: "${password}"
success:
  path: $.code
  value: 0
```

---

## 6. 开发计划

### Phase 1: 基础框架搭建 (1-2 周)
- [ ] testplat-control 项目初始化
  - FastAPI 后端骨架
  - React 前端骨架
  - SQLite 数据库表设计
- [ ] testplat-engine 项目初始化
  - 从 testplat-new 提取核心模块
  - 去除业务特定逻辑
  - 封装执行接口

### Phase 2: 单接口运行 (1 周)
- [ ] Control: API 资产解析与展示
- [ ] Control: 参数表单渲染
- [ ] Engine: 单接口执行逻辑
- [ ] Control: 结果展示页面

### Phase 3: 场景运行 (1 周)
- [ ] Control: Case 资产解析与展示
- [ ] Engine: 场景执行逻辑 (多步骤编排)
- [ ] Control: 步骤级结果展示

### Phase 4: 结果管理 (3-5 天)
- [ ] 执行记录保存与查询
- [ ] 历史记录列表
- [ ] 报告详情页 (复用 HTML 报告)

### Phase 5: 优化与完善 (持续)
- [ ] 异常处理与错误提示
- [ ] 参数历史记录
- [ ] 资产搜索与过滤
- [ ] 性能优化

---

## 7. 技术债务与风险

### 7.1 复杂度控制
- **风险**: testplat-new 的鉴权体系非常复杂 (多租户、多策略)
- **应对**: MVP 仅支持最简单的 HTTP Basic Auth / Bearer Token

### 7.2 跨平台兼容性
- **风险**: 文件路径在 Windows/Linux/Mac 表现不同
- **应对**: 统一使用 `pathlib.Path`,强制使用绝对路径

### 7.3 并发执行
- **风险**: 多用户同时运行可能冲突
- **应对**: MVP 阶段串行执行,后续版本再加任务队列

---

## 8. 验收标准

MVP 交付时,必须满足:

1. ✅ 服务端登记一个 Git 用例仓库后，Web 界面列出其中的项目；仓库新提交后点“同步”即可看到
2. ✅ 可以浏览项目 API / Case，并维护该项目环境、服务地址及测试账号，运行时选择环境
3. ✅ 可以点击某个 API,填写参数后运行,并看到请求/响应
4. ✅ 可以点击某个 Case,运行后看到步骤级结果
5. ✅ 执行记录保存环境、commit SHA、操作人和配置版本，修改配置后历史保持原值
6. ✅ 换一台机器,只需修改服务端仓库配置,即可正常使用

---

## 9. 后续迭代方向

完成 MVP 后,可根据使用反馈优先级排序:

- **P1**: 定时任务 (cron 表达式配置)
- **P1**: 并发执行控制 (任务队列)
- **P2**: UI 自动化支持 (集成 Playwright)
- **P2**: 数据库校验 (SQL 查询断言)
- **P2**: 更复杂的鉴权策略 (OAuth2、JWT)
- **P3**: 用例编辑器 (可视化编辑 YAML)
- **P3**: Mock 服务集成
- **P3**: 性能测试模式 (压测)

---

## 附录: 与 testplat-new 的差异对比

| 维度 | testplat-new | testplat (通用版) |
|------|--------------|-------------------|
| 定位 | 特定公司的业务测试平台 | 通用的开源测试框架 |
| 架构 | 单体仓库 (CLI) | 双仓库 (control + engine) |
| 鉴权 | 复杂多租户 + 多策略 | 简单 HTTP Auth |
| UI | 仅 CLI + HTML 报告 | Web 管理界面 + CLI |
| 项目绑定 | 硬编码 GSB/GO2/e3e3 | 配置文件驱动 |
| 数据库 | SQLAlchemy 连接池 | MVP 暂不支持 |
| UI 自动化 | Playwright 集成 | MVP 暂不支持 |
| 部署方式 | 本地 CLI | Web 服务 + API |
