# Phase 3 Agent 0：Case 契约启动前审计

日期：2026-10-12（Asia/Shanghai）  
范围：只读审计；本文件为唯一写入物。未改业务代码、未建分支、未暂存、未提交。

权威裁决前提（**不重新开放**）：第一批仅 `api` 步骤；Case 串行；默认 fail-fast（失败后后续步骤 `skipped`）。依据：`notes/phase3-agent-prompts.md` L8–L40。

---

## 1. 仓库基线证据

### 1.1 testplat-engine

| 项 | 值 |
|---|---|
| 工作树路径 | `/Users/jiavin/Developer/products/testplat1/testplat-engine` |
| remotes | fetch `https://github.com/JiavinI/testplat-engine.git`；push `git@github.com:JiavinI/testplat-engine.git` |
| 当前分支 / HEAD | `codex/phase2-5-main-bridge` @ `45e711806ffbc892a8000d0e9caee4c7eb0dccea`（clean） |
| 规划基线 `origin/main` | `2dba73e3cdefdea2509fe007693f02ef37d43e80`（`Merge pull request #11 …`）— **与规划一致** |
| 规划基线 `v0.1.0` | annotated tag → `942a66dd127de3647deab88611ada2a9bcc06c90`（tag 对象 `5433b3a…`，peeled 与规划一致） |
| 本地 `main` | `409a049`（**落后** `origin/main`；是其祖先）— 新 worktree 必须从 `origin/main`/`2dba73e`，不得猜本地 `main` |
| dirty | 无 |
| worktrees | 主树 `45e7118`；`worktrees/phase2-5-engine-interface` @ `bee429d`；`phase2-5-engine-runtime` @ `dfdec69`；`phase2-5-integration` @ `942a66d` |

**Engine Case 现状（`origin/main`）**：无 Case 领域模型/加载器/执行入口。公开协议仅为单 API（`src/testplat_engine/interface.py` L58–L125：`ApiExecutionRequest` / `ApiExecutionOutcome` / `EnginePort.execute`）。README 明确「当前基线不提供 Case 编排」。YAML 加载仅 `Api`（`adapters/filesystem/yaml_loader.py`）。

### 1.2 testplat-control

| 项 | 值 |
|---|---|
| 工作树路径 | `/Users/jiavin/Developer/products/testplat1/testplat-control` |
| remote | `git@github.com:JiavinI/testplat-control.git` |
| 当前分支 / HEAD | `main` @ `da5e26fd43a5257056190ec25c8aee59ec198e97`（= `origin/main`）— **与规划一致** |
| Engine 锁定 | `pyproject.toml` L18/L40：`testplat-engine>=0.1.0,<0.2`，git rev `942a66dd127de3647deab88611ada2a9bcc06c90` |
| dirty | 仅未跟踪 `PHASE2_3_REVIEW_REPORT.md`（与 Case 无关） |
| worktrees | 多条 phase2-4 / phase2-5 历史树；另有 `/private/tmp/…` detached |

**Control Case 现状**：展示向松散解析存在；无 Case 执行路径。`YamlParser.parse_case_yaml`（`parsers/yaml_parser.py` L61–L105）+ fixture `tests/fixtures/case_login.yaml` + `tests/test_yaml_parser.py` L57–L103。`ExecutionRecord.asset_type` 已允许 `"case"`（`database/models.py` L39），但 `execution_service` 当前只跑 API。

### 1.3 旧资产只读样本

`/Users/jiavin/Developer/products/testplat-new/assets/scenarios/`：**57** 个 `*.case.yaml`（e3e3 / go2 / shared flows）。仅证据，不复制业务模型。

---

## 2. 证据矩阵：字段与语义差异

图例：✔ 有 / ✖ 无 / △ 部分或语义不同 / → 规划已裁决

### 2.1 Case 顶层

| 字段/语义 | MVP规格 | 补充需求 | 需求汇总 | 架构 doc | Control parser/fixture | 旧 Case（57） | Phase3 已批准 | 差异要点 |
|---|---|---|---|---|---|---|---|---|
| Case 身份 `id` | 接口用 `case_id`（MVP L148） | 未定 schema | `execute_case(case_id)`（汇总 L411） | `case_id`（架构 L162） | fixture 有 `id`（case_login L2）；parser 提取（yaml_parser L100） | **0/57** 无顶层 `id`（靠路径/`case.…` 命令） | DTO 需 Case 身份（prompts L46） | **路径派生 vs 文件内 id** 未裁决 |
| `name` | 展示场景信息（MVP L50） | — | 展示（汇总 L177） | `StepResult.name`（架构 L106） | fixture/parser 有 | **0/57** | 未强制 | 旧靠 `desc`+文件名 |
| `desc` | ✔（MVP L50） | — | ✔ | — | ✔ | **57/57** | metadata 候选 | 稳定 |
| `tags` | ✔（MVP L50） | — | ✔ | — | ✔（默认 `[]` L103） | **57/57** | metadata 候选 | 稳定 |
| `project` | 项目隔离概念 | — | project+env（汇总 L623） | `ProjectContext.project_id` | ✖ | **57/57** | Control 管项目 | **业务/旧 DSL**；不宜进 Engine Case schema |
| `data.use` | — | — | 测试数据多文件（汇总 L619） | — | ✖ | **56/57** | 本批不引入 | **未来/数据绑定** |
| `default_as` | — | 身份/Context | — | — | ✖ | **56/57** | 鉴权后续门禁 | **未来** |
| `retry_safe` | — | — | — | — | ✖ | **55/57** | 明确无 retry（prompts L40） | **拒绝/unsupported** |
| Case 级 `assert` | 「场景级断言」（MVP L55） | — | ✔（汇总 L188） | `CaseResult.assertions`（架构 L132） | ✖ | **50/57** | 未明确是否第一批 | **需主线程裁决** |
| Case 级 `params` | 单 API 有 params；Case 接口无（MVP L147–152） | `${params…}`（补充 L61） | API params；Case 无显式 | Case 入口无 params（架构 L162–166） | fixture 仅步骤 params | **0/57** 顶层 `params`（经 `data`→`${params…}`） | **要求 Case 级 params**（任务 L133；prompts 输入环境/参数 L46） | 旧用 data 间接注入；新契约应显式 |
| `metadata` | — | — | — | — | ✖ | ✖ | 任务要求覆盖 | **新增** |
| `steps` | ✔ | ✔（kind 示例） | ✔ | ✔ | ✔ 原样透传 | **57/57** | ✔ 顺序串行 | 稳定 |
| 检测 Case 类型 | — | — | — | — | `steps` 键存在→case（L129） | 文件后缀 `.case.yaml` | Engine 严格校验 | Control 启发式脆弱 |

### 2.2 Step 字段

| 字段/语义 | 补充需求示例 | Control fixture | 旧 Case | Phase3 已批准 | 差异 |
|---|---|---|---|---|---|
| 稳定步骤 ID | 无 `step_id`；有 `name` | 无；索引引用 `${steps.0…}`（case_login L15） | 可选 `id`（仅 **4** 次，多用于 UI wait） | **必须 `step_id`**；禁下标跨步引用（prompts L37） | Control fixture **违反**已批准语义 |
| 类型判别 | `kind: api/ui`（补充 L58–65） | 无；隐含 api_id | 由 `use` 前缀（api/flow/ui/db/transform） | **`type=api`**（任务）；seam 按 type 注册 | `kind` vs `type`；旧无显式 type |
| API 引用 | `use: api.order.create` | `api_id:` | `use: api.…`（纯 api 仅 14 步/2 case） | 需 API 资产引用 | 命名三套：`use` / `api_id` / 待定 |
| 输入 | `with:` | `params:` | `with:`（202/206） | params | `with` vs `params` |
| 输出 alias | `->: order` | 无显式 alias；用下标 | `->` 字符串 alias **173**；dict mapping **5**；缺省 **28** | **显式 output alias**；唯入口 | JSON 无法原生表示 `->` |
| 步骤断言 | UI assert 示例 | ✖ | 18 步有 assert；纯 api baseline 有 | 断言失败=业务 failed（prompts L34） | 断言挂在 step 还是仅 API `success`？ |
| `as` 身份 | — | ✖ | 18 | 后续鉴权 | 未来 |
| `wait_until` | — | ✖ | 5（UI） | 无条件/轮询（prompts L40） | unsupported |
| `name` | ✔ | ✔ | **206/206** | 展示用 | 建议保留可选/推荐 |
| flow/ui/db/transform | 混合 P0（扩展 seam） | ✖ | flow **167**，ui **22**，db **2**，transform **1** | 第一批只注册 api；其它 **unsupported** | 不得当本批能力 |

### 2.3 执行语义

| 语义 | 架构 doc | Phase3 已批准 | 备注 |
|---|---|---|---|
| 串行 | MVP 串行（架构 L478；MVP L368） | ✔ | 已批准，不开放 |
| fail-fast 默认 | `ExecutionConfig.fail_fast: bool = False`（架构 L94） | **默认 fail-fast**（prompts L33） | **文档漂移**；以 Phase3 裁决为准，需后续改架构文，不重新投票 |
| `skipped` | `StepStatus.SKIPPED`（架构 L101） | 失败后补齐 skipped | 对齐 |
| 业务断言失败 | 架构把断言失败塞进 `ExecutionError`（L384–389）又说业务失败非 5xx（L411–418） | HTTP 200 + Case `failed` + 存记录（prompts L34） | 以 Phase3 + Phase2.5 outcome 模式为准 |
| timeout/transport | — | 抛系统异常 + `partial_outcome`（prompts L35） | 对齐 Engine 0.1 kinds |
| preflight 无效资产 | AssetNotFound / YamlParse | 不建执行记录（prompts L36） | 对齐 |
| 变量失败 | — | 必须指出步骤与路径，禁降级空串（prompts L38） | 对齐 |
| 秘密脱敏 | — | 公开结果禁含秘密（prompts L39） | 对齐 |

### 2.4 职责边界冲突（YAML 解析归属）

| 来源 | 说法 |
|---|---|
| 架构 L11 vs L26 | Control：「测试资产解析与展示」；Engine：「YAML DSL 解析」 |
| Phase3 所有权表（prompts L69–L71） | **Case 资产语法与运行语义 → Engine**；Control 只负责路径/环境/HTTP/记录/展示 |
| Control 现状 | `YamlParser` 在 Control，仅展示/启发式 |

结论：运行期严格 Case 契约必须落在 Engine；Control 解析器最多保留**展示用松散摘录**，不得充当执行校验源。

---

## 3. 旧示例字段分类（禁止复制业务专属）

### 3.1 第一批必需（最小 API Case）

来自已批准语义 + 可泛化旧证据（尤其 `*/baseline/readonly_smoke.case.yaml`）：

1. 有序 `steps`
2. 稳定 `step_id`（新硬性要求；旧 `id` 不足）
3. 步骤 `type: api`（显式；禁靠前缀猜）
4. API 资产引用（指向已有 `api.*` 契约）
5. 步骤输入 params（可含 `${…}`）
6. 显式 output alias（跨步唯一入口）
7. Case 级可注入 params（运行入口；替代旧 `data` 间接通道的最小形态）
8. Case 级展示/归档 metadata（至少 `desc`/`tags`；可选 `name`）
9. 步骤失败 → Case `failed` + 后续 `skipped`（行为，非 YAML 字段）

### 3.2 未来扩展（保留 seam，本批遇则校验失败）

| 项 | 旧证据 | 处理 |
|---|---|---|
| `type`/`kind` ∈ {ui, db, redis, custom} | 补充 L51；ui/db 用例 | `unsupported_step_type` |
| `use: flow.*` / flow 复用 | 167 步 | unsupported（无 flow 展开） |
| `use: transform.*` / `collection.*` | shared flows | unsupported |
| `wait_until` | UI 轮询 | unsupported |
| `as` / `default_as` | 身份切换 | 鉴权门禁后 |
| `data` 绑定 | 56/57 | 数据合并门禁后 |
| `retry_safe` / retry | 55/57；prompts 禁 retry | 出现则验证错误 |
| Case 级 `assert`（若本批不做） | 50/57 | 见阻塞问题 |
| dict 形 `->` 多导出 | 5 次（多在 flow） | 本批仅单 alias 字符串 |
| `anon` / `secure` | 各 1 | 未知字段策略见下 |

### 3.3 业务专属（**不得**建议进入平台 Case schema）

- 项目域：`project: go2|e3e3`，`data.go2.items` / `data.e3e3.items` profiles（`single_sku`/`multi_sku`/`coupon`…）
- 身份名：`gsb.b2.primary`、`df_picker.default`、`admin_go2.default`
- 领域 API/flow：`api.gsb.b2.*`、`flow.*.take_return.*`、`aftersale`/`refund`/`take_settle`
- 领域字段：`product_keyword`、`receiver_text`、`express_code`、`send_type`、`supplier_id`、`orderItemId`、`web_order_amount`、`error_code`
- UI/Page：`page.gsb.*`、`ui.open/click/fill/read/dismiss/screenshot`
- 业务辅助函数：`collection_group_by`、`date_add_days` 等

纯 api 可参考形态（仅结构）：`go2/baseline/readonly_smoke.case.yaml`、`e3e3/baseline/readonly_smoke.case.yaml`——但其中的 `data`/`default_as`/业务 api id **仍属样例数据，不入库契约**。

---

## 4. 最小 API Case schema 草案（提议，待主线程批准）

### 4.1 设计原则

- Engine 为语法与运行语义唯一权威；文件格式以 **YAML 资产** 为主，JSON 为等价交换形（DTO/`to_dict`）。
- 第一批只允许 `type: api`；其它 type → preflight `invalid_asset` / 明确 unsupported kind。
- 跨步数据**只**通过显式 alias；禁止 `${steps.<index>…}` 与禁止默认把整步响应塞进上下文。
- 不收录业务专属键；未知键默认拒绝（见 §6）。
- 命名提议偏向 **JSON 友好英文键**，与旧 DSL 的对应关系写在映射表，而不是静默双写。

### 4.2 YAML 形状（权威资产）

```yaml
# 提议文件名：*.case.yaml（或 *.yaml 且根含 schema_version+steps）
schema_version: "0.2"
id: case.demo.login_then_profile   # 可选：若缺省则由相对路径派生，规则需裁决
desc: "登录后拉取用户资料"
tags: [smoke]
metadata:
  owner: platform
params:                              # Case 级默认参数（可被执行请求覆盖）
  username: demo
  password: secret
steps:
  - step_id: login
    type: api
    api_id: api.demo.login           # 引用已有 API 资产 id
    name: 用户登录                     # 可选，展示用
    params:
      username: "${params.username}"
      password: "${params.password}"
    output: token_bundle             # 显式 alias；写入上下文
    # assert: 是否允许 — 见阻塞问题 A3
  - step_id: profile
    type: api
    api_id: api.demo.get_profile
    params:
      user_id: "${token_bundle.body.user_id}"
    output: profile
```

### 4.3 JSON 等价形（DTO / 夹具）

```json
{
  "schema_version": "0.2",
  "id": "case.demo.login_then_profile",
  "desc": "登录后拉取用户资料",
  "tags": ["smoke"],
  "metadata": {"owner": "platform"},
  "params": {"username": "demo", "password": "secret"},
  "steps": [
    {
      "step_id": "login",
      "type": "api",
      "api_id": "api.demo.login",
      "name": "用户登录",
      "params": {
        "username": "${params.username}",
        "password": "${params.password}"
      },
      "output": "token_bundle"
    },
    {
      "step_id": "profile",
      "type": "api",
      "api_id": "api.demo.get_profile",
      "params": {"user_id": "${token_bundle.body.user_id}"},
      "output": "profile"
    }
  ]
}
```

### 4.4 字段表（最小）

| 路径 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `schema_version` | string | 建议必填 | 第一批固定 `"0.2"` |
| `id` | string | 待裁决 | 非空；与路径 id 冲突时以谁为准见 A1 |
| `desc` | string | 否 | 展示 |
| `tags` | string[] | 否 | 默认 `[]` |
| `metadata` | object | 否 | 仅 JSON 标量/嵌套对象；引擎不解释 |
| `params` | object | 否 | Case 默认参数；执行请求可覆盖 |
| `steps` | array | 是 | 长度 ≥ 1；顺序即执行序 |
| `steps[].step_id` | string | 是 | Case 内唯一；`^[A-Za-z_][A-Za-z0-9_]*$` 提议 |
| `steps[].type` | string | 是 | 本批仅 `"api"` |
| `steps[].api_id` | string | type=api 时必填 | 必须能解析到 API 资产 |
| `steps[].name` | string | 否 | UI 展示；缺省可用 `step_id` |
| `steps[].params` | object | 否 | 默认 `{}`；渲染后交给单 API 执行 |
| `steps[].output` | string | 是* | 显式 alias；见 A2 是否允许省略 |

\*若主线程允许「无输出步骤」，可改为可选；但任一后续引用必须仍指向已声明 alias。

### 4.5 与旧 DSL / Control fixture 映射（仅文档，不双写实现）

| 旧/Control | 提议 |
|---|---|
| `use: api.x` / `api_id` | `type: api` + `api_id` |
| `with` / `params` | `params` |
| `->: alias` | `output: alias` |
| `kind`（补充需求） | `type` |
| `${steps.0.response…}` | **禁止** → `${alias…}` |
| `id:`（旧可选步 id） | `step_id`（必填） |

### 4.6 运行结果形状（契约侧最低期望，非实现）

与 prompts L46 / Phase2.5 outcome 模式对齐：

- Case outcome：`status: success|failed`；`steps[]` 含 `step_id`、`status: success|failed|skipped`、耗时、request/response（脱敏）、error、assertions、`output` alias 快照（可选）
- 业务失败：不抛；`status=failed`
- execution-stage 系统错误：抛 `EngineError`（timeout/transport/internal）且带同结构 `partial_outcome`
- preflight：`asset_not_found` / `invalid_asset` / `invalid_environment` — 无记录

---

## 5. Golden fixture 清单（提议）

均建议落在 Engine（批准后由 Agent 1 落地），例如 `tests/fixtures/cases/…` + 对应 API 资产；此处只定义意图与期望，不写实现。

| ID | 场景 | Case 要点 | 期望 |
|---|---|---|---|
| **G1 success** | 两步 API，第二步用第一步 alias | `login`→`output: auth`；`profile` params 引用 `${auth…}`；两步 API `success` 满足 | Case `success`；两步 `success`；无 skipped；上下文含 alias |
| **G2 step assert fail** | 第一步业务断言失败 | 步骤或 API success 条件故意不满足 | Case `failed`；失败步 `failed`；后续步 **全部 `skipped`**；HTTP/记录层按业务失败（非 preflight） |
| **G3 var propagate** | 三步链传递 | A→alias_a→B→alias_b→C | C 请求参数等于渲染后的上游值；成功 |
| **G4 illegal ref** | 引用未声明 alias / 路径不存在 | params 含 `${missing.x}` 或 typo alias | **preflight 或该步执行前** 稳定错误，消息含 step_id + 变量路径；**不**降级空串；不继续后续步（若已开跑则 fail-fast + skipped） |
| **G5 unsupported step** | `type: ui`（或 `flow`）夹在 api 之间 | 第二步 `type: ui` | **preflight `invalid_asset` / unsupported**；不创建执行记录；不执行任何 HTTP |
| **G6 timeout/transport** | 某步 HTTP 超时或传输失败 | mock/运输层注入 | 抛 `EngineTimeoutError` / `EngineTransportError`；`partial_outcome` 含已完成步 + 当前失败步 + 后续 skipped；状态不自相矛盾 |

**附加 mutation 夹具（建议与 Agent 5 对齐，可选列入批准集）：**

| ID | 意图 | 期望 |
|---|---|---|
| M1 重复 `step_id` | 两步同 id | preflight 拒绝 |
| M2 alias 冲突 | 两步相同 `output` | preflight 拒绝 |
| M3 前向引用 | 步骤 1 引用步骤 2 的 alias | preflight 拒绝 |
| M4 未知字段 | 顶层或步骤含 `retry_safe`/`data`/`foo` | preflight 拒绝（若批准严格模式） |
| M5 缺 `output` 却被引用 | — | 非法引用 |
| M6 循环引用 | params 表达式别名环（若渲染模型允许） | preflight 或渲染期拒绝，明确报环 |

---

## 6. 校验与错误处理提议

| 规则 | 提议行为 | 阶段 |
|---|---|---|
| JSON/YAML 形状 | YAML 资产与 JSON DTO 同构；`output` 对应旧 `->`；加载后归一到同一内部模型 | preflight |
| 重复 `step_id` | 拒绝；列出冲突 id | preflight |
| alias 冲突（重复 `output`） | 拒绝；即便中间有失败也可能在装载期检出 | preflight |
| 前向引用 | 静态扫描 `${alias…}` / `${params…}`；alias 必须定义在**更早**步骤；`${params…}` 仅 Case/请求 params | preflight |
| 未知字段 | **严格拒绝**（推荐）；展示层可另做宽松摘录，但不得喂给 Engine | preflight |
| 未知/非 api `type` | unsupported → `invalid_asset`（或专用 kind，需与 0.2 错误表对齐） | preflight |
| 循环引用 | 在依赖图上检环；报涉及的 step_id/alias | preflight |
| 空 `steps` / 缺必填 | `invalid_asset` | preflight |
| 非法 `api_id`（文件不存在） | `asset_not_found` | preflight |
| 变量运行期缺失（响应缺路径） | 该步失败或执行错误；消息含 step_id+路径；fail-fast | execution（业务或 internal——**见 A4**） |
| 下标引用 `${steps.N…}` | 直接视为非法语法 | preflight |

---

## 7. Control Case parser 复用审计

文件：`testplat-control/src/testplat_control/parsers/yaml_parser.py`  
测试：`tests/test_yaml_parser.py`  
夹具：`tests/fixtures/case_login.yaml`  
调用方：`services/asset_service.py` L205（`parse_yaml` 整文件入 metadata）；`api/run.py` 仅 API 路径使用 `parse_api_yaml`。

| 组件 | 现状 | 建议 | 理由 |
|---|---|---|---|
| `parse_api_yaml` | 提取 API 展示字段 | **保留（Control）** | 单 API 工作台仍需要；与 Engine API 模型大致对齐 |
| `parse_case_yaml` | 透传 `steps`，抽 id/name/desc/tags；**无校验** | **废弃为执行源**；短期可标 deprecated | 无 `step_id`/`type`/`output`；接受旧 `api_id`+下标引用，与 Phase3 冲突 |
| `parse_yaml` 启发式 `steps∈root ⇒ case` | 用于资产树 metadata | **保留但收窄用途=展示**；勿当契约 | flow 文件亦有 `steps`，存在误判风险 |
| `case_login.yaml` fixture | `api_id` + `${steps.0.response.user_id}` | **删除或重写**为批准 schema 的展示样例 | 直接违反 prompts L37 |
| Case 相关 parser 测试 L57–L103 | 锁定旧形状 | **迁移/重写** 到 Engine 契约测试；Control 仅留「展示摘录」测 | 执行语义归 Engine |
| `AssetService._extract_file_metadata` | 解析失败写 `parse_error` 不中止 | **保留** 展示韧性 | 与 Engine 严格 preflight 分离 |
| `AssetService` 文件 id 生成 L150–156 | `Path.stem` → `login_success.case.yaml` ⇒ id `….login_success.case` | **修复纳入 Control Case 展示工作项**（非 Engine 契约） | 双后缀 bug；与旧 `case.*` 命令不一致 |
| Case 执行 / 校验 / 变量 / fail-fast | Control 无 | **迁移到 Engine 0.2**（Case loader+validator+CaseRunner） | prompts 所有权表 |
| 双份 YAML 解析逻辑 | Control + 未来 Engine | **禁止双实现执行校验**；Control 最多调 Engine 校验 API 或只读摘录 | 防漂移 |

**复用结论：** 现有 Control Case parser **不适合**作为 Phase 3 执行契约实现。可保留 API 展示解析；Case 执行契约从零在 Engine 按批准 schema 建立。Control 侧改为：展示摘录（宽松）+ 调用 Engine 执行（严格）。

---

## 8. 需主线程裁决的阻塞问题

**不重新开放：** API-only 第一批、串行、默认 fail-fast、Engine 拥有 Case 语法、`step_id`+显式 alias、禁下标引用、业务失败 vs 系统错误分界（prompts 已批）。

### A1. Case `id` 来源

- 旧资产 0/57 无文件内 `id`；Control fixture 有；路径派生存在 `.case` stem bug。  
- **请裁：** (i) 文件内必填 `id`；(ii) 仅路径派生；(iii) 可选文件内，缺省路径派生——冲突时谁赢。

### A2. `output` 是否每步必填

- 旧 28/206 步无 `->`。  
- **请裁：** 每步强制 `output`，或允许无输出步骤（不可被引用）。

### A3. 第一批是否包含步骤级 / Case 级 assert DSL

- 旧大量 Case 级 assert；API 资产已有 `success`。  
- **请裁：** (i) 本批仅 API `success`；(ii) 允许步骤 `assert[]`；(iii) 另加 Case 级 `assert[]`。  
- 影响 G2 夹具写法与 DTO。

### A4. 响应路径缺失的错误分类

- 渲染时 alias 存在但 JSON 路径缺失：业务 `failed` 还是 `internal`？  
- **请裁：** 归业务失败（推荐，利于 fail-fast+记录）或系统错误。

### A5. 表面语法选型（键名）

- 候选：补充需求 `kind`+`use`+`with`+`->` vs 本草案 `type`+`api_id`+`params`+`output`。  
- **请裁：** 批准一套；是否提供只读旧键别名（**不推荐**本批双键）。

### A6. 未知字段策略

- 严格拒绝（推荐）vs 忽略未知（便于渐进迁旧资产）。  
- 旧资产几乎都带 `data`/`default_as`/`retry_safe`——严格模式下**不能**直接跑旧仓 Case（符合「不复制旧系统」）。  
- **请裁：** 确认严格拒绝。

### A7. 架构文档 `fail_fast=False` 漂移

- 非产品决策重开；**请裁：** 是否授权后续文档 PR 改为「默认 true / Phase3 固定 fail-fast」，避免 Agent 误读。

### A8. Control 展示层对未批准键的态度

- 资产树是否继续原样展示旧 `use`/`->` 文件，或仅支持新 schema？  
- **请裁：** Phase3 UI 只保证新 schema；旧键显示为 raw metadata 即可。

---

## 9. 交付摘要（给主线程）

1. **基线核验通过**：Engine `origin/main=2dba73e`，`v0.1.0→942a66d`；Control `main=da5e26f` 锁 Engine `942a66d`。本地 Engine `main` 落后，新建树须用 `origin/main`。  
2. **差异核心**：四套来源在步骤类型键、API 引用键、params/`with`、alias/`->`、Case id、Case 断言、fail-fast 默认值上不一致；Control fixture 的下标引用与已批准语义冲突。  
3. **旧资产**：仅 2 个纯 api Case；其余依赖 flow/ui/业务数据——只可作证据，不可当第一批夹具来源。  
4. **最小 schema**：§4 草案（`schema_version`/`step_id`/`type=api`/`api_id`/`params`/`output`/Case `params`+`metadata`）。  
5. **Golden**：G1–G6 + mutation M1–M6。  
6. **迁移**：Control Case parser 不作执行权威；Engine 新建严格契约；重写/删除 `case_login` 旧形。  
7. **阻塞**：A1–A8 待主线程批准后方可启动 Agent 1。

---

## 10. 关键引用索引

- `notes/phase3-agent-prompts.md` L8–L40, L69–L71, L110–L141  
- `docs/architecture/control-engine-interface.md` L11–L30, L89–L180, L384–L418, L478  
- `docs/requirements/补充需求.md` L40–L111  
- `docs/requirements/MVP-需求规格.md` L49–L55, L147–L152, L338–L341, L368  
- `docs/requirements/需求汇总.md` L104–L106, L411–L418, L470–L478, L613–L620  
- Control `parsers/yaml_parser.py` L61–L131；`tests/fixtures/case_login.yaml` L1–L15  
- Control `services/asset_service.py` L150–L210  
- Engine `interface.py`（origin/main）L58–L214  
- 旧样例：`…/go2/baseline/readonly_smoke.case.yaml`；`…/go2/order/full_pay/items_search_visible.case.yaml`

