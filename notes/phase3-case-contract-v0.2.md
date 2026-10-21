# Phase 3 Case Contract v0.2 裁决稿

日期：2026-10-12
状态：**已批准，供 Agent 1 实施与测试**
依据：`phase3-agent0-case-contract-audit.md` 的只读证据；本文裁决 A1–A8，并取代其中的提议性 schema 和 Golden 期望。

> 本文冻结 Phase 3 第一批的 Case 资产表面语法和可观察语义。第一批只支持串行 API 步骤；它不授权鉴权、凭证、UI/DB/Redis/custom executor、实时进度、重试、cleanup、条件、循环或 Case 编辑器。

## 1. A1–A8 裁决

| 编号 | 裁决 | 理由与约束 |
|---|---|---|
| A1 | Case `id` **文件内必填**，并必须与请求的 `case_id` 及规范路径一致 | 身份不能随展示层的路径算法漂移；沿用 API 资产“文件内 id 与请求 id 一致”的防错不变量 |
| A2 | `output` **可选** | 只做校验或副作用的步骤无需污染上下文；任何跨步引用仍只能指向更早步骤显式声明的 alias |
| A3 | 本批**不新增 step-level 或 Case-level assert DSL** | API 资产已有 `success` 条件；Case 汇总其业务结果。新的断言语法和算子另行设计 |
| A4 | 已声明 alias 存在、但运行时响应路径缺失，属于当前步骤的**业务失败** | 这是资产对上游输出形状的错误，不是 Engine 缺陷；HTTP 200、保存记录、后续步骤 `skipped` |
| A5 | 只接受 `type/api_id/params/output` | 不接受 `kind/use/with/->` 别名，不做静默双键归一；迁移工具不属于本批 |
| A6 | schema-owned 对象的未知字段**严格拒绝** | 防止拼写错误和未实现能力被静默忽略；`params` 的用户数据键不属于“未知字段” |
| A7 | 修正文档漂移：Phase 3 第一批固定 fail-fast，默认值为 `true` 且调用方不可覆盖 | 后续文档工作必须明确旧架构草案的 `fail_fast=False` 已被本文取代 |
| A8 | 结构化 Case UI 和运行只认新 schema | 旧资产可继续作为树中的文件出现，但只显示文件名、路径和“旧格式/不受支持”；不能展示整份 raw 内容、不能运行，也不能偷偷转换 |

## 2. 规范身份与文件路径

Case ID 格式：

```text
case.<segment>[.<segment>...]
segment := [A-Za-z_][A-Za-z0-9_-]*
```

规范映射：

```text
case.demo.login_then_profile
-> {assets_root}/scenarios/demo/login_then_profile.case.yaml
```

规则：

1. `case.` 前缀必需；至少有一个后续 segment。
2. `/`、`\`、空 segment、`.`、`..` 均非法。
3. symlink 或解析后的路径不得逃逸 `assets_root/scenarios`。
4. 文件内 `id` 必须精确等于请求的 `case_id`；不允许大小写或连字符自动归一。
5. 第一批只认 `.case.yaml`，不以根对象是否含 `steps` 猜类型。

## 3. 权威 YAML schema

```yaml
schema_version: "0.2"
id: case.demo.create_then_get
name: 创建并查询资源              # 可选；缺省时 UI 使用 id
desc: 创建资源后用返回 ID 查询      # 可选
tags: [smoke]                    # 可选，默认 []
params:                          # 可选，默认 {}；只允许非秘密默认值
  resource_name: demo
steps:
  - step_id: create
    type: api
    api_id: api.demo.create
    name: 创建资源                # 可选；缺省时 UI 使用 step_id
    params:                      # 可选，默认 {}
      name: "${params.resource_name}"
    output: created              # 可选 alias

  - step_id: get
    type: api
    api_id: api.demo.get
    params:
      resource_id: "${created.body.id}"
```

顶层只允许以下字段：

| 字段 | 类型 | 必填 | 规则 |
|---|---|---|---|
| `schema_version` | string | 是 | 精确为 `"0.2"` |
| `id` | string | 是 | 满足 Case ID 格式并匹配请求/路径 |
| `name` | string | 否 | 非空；仅展示 |
| `desc` | string | 否 | 字符串；仅展示 |
| `tags` | string[] | 否 | 默认 `[]`，元素非空 |
| `params` | JSON object | 否 | 默认 `{}`；执行请求可按顶层键覆盖 |
| `steps` | step[] | 是 | 至少一个，数组顺序即执行顺序 |

第一批不接受任意 `metadata` 容器。需要新的归档/治理字段时，以显式 schema 字段演进，避免把无界对象变成跨仓公共 Interface。

API step 只允许以下字段：

| 字段 | 类型 | 必填 | 规则 |
|---|---|---|---|
| `step_id` | string | 是 | `[A-Za-z_][A-Za-z0-9_-]*`；Case 内唯一 |
| `type` | string | 是 | 第一批精确为 `api` |
| `api_id` | string | 是 | 按既有 API registry 规则解析 |
| `name` | string | 否 | 非空；仅展示 |
| `params` | JSON object | 否 | 默认 `{}`；支持本文第 4 节引用 |
| `output` | string | 否 | alias，格式同 `step_id`；Case 内唯一且不得为 `params` |

未知顶层字段、未知 step 字段、`assert`、`data`、`default_as`、`retry_safe`、`kind`、`use`、`with`、`->` 均在 preflight 严格拒绝。

## 4. 参数与引用语义

### 4.1 参数合并

`CaseExecutionRequest.params` 以顶层键覆盖资产 `params`，得到只读 Case 参数空间。第一批只做浅层覆盖；不做隐式深合并。资产与请求参数都必须是 JSON object。

资产默认值和 Golden fixtures 不得包含 password、token、Cookie、Authorization 或真实个人数据。凭证注入留到独立安全设计。

### 4.2 允许的引用

```text
${params.resource_name}
${created.body.id}
${created.status_code}
```

- `params` 是保留根，读取合并后的 Case 参数。
- 其他根必须是更早步骤显式声明的 `output` alias。
- alias 绑定该 API 步骤的安全响应对象：`status_code`、已脱敏 `headers`、已安全化 `body`。它不包含 request、error 或完整 StepOutcome。
- “安全响应”要求对 header 名及嵌套 JSON 键做大小写不敏感的递归脱敏；至少覆盖 `authorization`、`cookie`、`set-cookie`、`token`、`access_token`、`refresh_token`、`password`、`secret`、`api_key`。脱敏值不能再供后续步骤引用；凭证传播必须等待独立身份/秘密通道。
- `${steps.0...}`、前向 alias、未声明 alias 和自引用均非法。
- 路径第一批只支持点分对象键，不支持数组下标、JSONPath、函数或表达式。

### 4.3 值类型

- 字符串恰好是一个占位符时，保留引用值的 JSON 类型。
- 占位符嵌入普通字符串时，只允许引用 JSON scalar，并按字符串插值。
- 对 object/list 做字符串插值是业务失败；不隐式 JSON stringify。
- `${params.missing}` 可在执行前确定，作为 preflight `invalid_input` 拒绝，不创建记录。`invalid_input` 是 Engine 0.2 新增的稳定 kind，stage 固定为 `preflight`，Control 映射 HTTP 422。
- alias 已合法声明但运行时路径不存在，当前消费步骤返回业务 `failed`：`error.kind=variable_resolution`，错误包含安全的 `step_id` 与引用路径；request 为 `null`，后续步骤 `skipped`。

## 5. 结果与错误不变量

### 5.1 StepOutcome

每个声明步骤恰有一个、且顺序相同的结果：

```text
step_id, name?, type, status, duration_seconds, request?, response?, error?, output?
```

- `success`：`error=null`；可有 request/response；`output` 只回显 alias 名，不复制响应值。
- `failed`：必须有安全 error；可能有 request/response。变量解析失败时 request/response 均为 `null`。
- `skipped`：`duration_seconds=0`，request/response/error 均为 `null`；可用单独 `skip_reason=fail_fast`，不得伪造成失败。
- duration 是有限非负秒数。

### 5.2 CaseExecutionOutcome

```text
case_id, status, duration_seconds, steps, summary
```

`summary` 是由步骤结果计算出的 `{total, success, failed, skipped}`，不得接收调用方自报值。Case `success` 当且仅当全部步骤 `success`；否则为 `failed`。第一批所谓“场景级断言汇总”就是步骤/API success 条件结果的汇总，不是新的 Case assert DSL。

### 5.3 系统错误

timeout、transport、execution-stage internal 继续抛公开 `EngineError`，并携带安全 `partial_outcome`：

1. 已完成步骤保留原结果。
2. 当前步骤为 `failed`，error.kind 与公开异常 kind 一致。
3. 后续步骤全部为 `skipped`。
4. partial outcome 满足完整 Case outcome 的顺序、计数和状态不变量。
5. 异常与当前步骤 error 不得表达不同 kind/stage。

## 6. Golden fixtures G1–G6

Golden 使用本地、无秘密的 API fixtures；不得使用登录/token 场景，不访问公网。Agent 1 先落资产和期望清单/契约测试，Agent 2 再让运行期结果全部变绿。

| ID | 资产设计 | 精确期望 |
|---|---|---|
| G1 `success` | 两步：`create -> output: created`，`get` 使用 `${created.body.id}` | Case success；2 success；summary `2/2/0/0`；第二步收到 ID；无重复 output 快照 |
| G2 `api_assert_failed` | 第一步引用一个 API fixture，其既有 `success` 条件故意不满足；后接一步 | HTTP 200 业务结果；Case failed；首步 `error.kind=assertion`；第二步 skipped；summary `2/0/1/1` |
| G3 `typed_propagation` | 三步传播 number、boolean 或 object；整个字段使用占位符 | C 收到的值保持原 JSON 类型，不被字符串化；3 success |
| G4a `undeclared_alias` | params 引用 `${missing.body.id}` | preflight `invalid_asset`；不发 HTTP；不创建记录 |
| G4b `missing_runtime_path` | alias 合法，但上一步响应没有 `${created.body.missing}` | 上一步 success；消费步骤业务 failed，`variable_resolution`，request=null；后续 skipped；HTTP 200 并保存记录 |
| G5 `unsupported_step` | 中间步骤 `type: ui` | preflight `invalid_asset`，public message 指出不支持的 type；0 次 HTTP；不创建记录 |
| G6a `timeout` | 中间 API 由可控 client 触发 timeout | 抛 timeout/execution；partial outcome 含前步 success、当前 failed、后步 skipped；保存 failed 后由 Control 映射 504 |
| G6b `transport` | 中间 API 由可控 client 触发 transport | 同 G6a，但 kind=transport，Control 映射 502 |

所有 Golden 都必须同时验证步骤顺序、summary、JSON 序列化和敏感字段不出现。G6 不靠 YAML 伪造错误，而由注入 client 或本地 test server 触发。

## 7. 必须落地的负例 M1–M8

| ID | 变异 | 期望 |
|---|---|---|
| M1 | 重复 `step_id` | preflight `invalid_asset` |
| M2 | 重复 `output` | preflight `invalid_asset` |
| M3 | 前向引用 | preflight `invalid_asset` |
| M4 | schema-owned 对象含未知字段 | preflight `invalid_asset` |
| M5 | 省略 `output` 且无人引用 | 合法 |
| M6 | 下标引用 `${steps.0...}` | preflight `invalid_asset` |
| M7 | 文件 id 与请求/路径不一致 | preflight `invalid_asset` |
| M8 | `${params.missing}` | preflight `invalid_input`/422，不创建记录 |

对每个负例断言精确 kind/stage；只断言“抛错”不算通过。

## 8. Control 与文档迁移

1. Control 的 `parse_case_yaml` 不作为执行权威；结构化 Case 详情最终消费 Engine 的规范化模型/结果。
2. `tests/fixtures/case_login.yaml` 及 `${steps.0...}` 测试必须重写为新 schema，不保留双语法。
3. Control 资产树修正 `.case.yaml` 双后缀展示 ID，但不得据此替代文件内 `id` 校验。
4. 旧 Case 文件可以显示文件名/路径，但 run 按钮禁用并给出“旧格式/不受支持”；不展示整份 raw 内容，不自动迁移。
5. `docs/architecture/control-engine-interface.md` 中 `fail_fast=False`、双 execute 入口、回调/WebSocket 直连等均是旧草案。文档修订应引用本裁决和 Engine 0.2 单入口，不只孤立改一个默认值。

## 9. Agent 1 开工门禁

Agent 1 可以开工，但必须遵守：

- 从 Engine `origin/main @ 2dba73e3cdefdea2509fe007693f02ef37d43e80` 建独立 worktree。
- Interface/loader/validator/fixtures/契约测试为本任务范围；不实现真实步骤网络执行。
- `StepExecutor`/registry 是 Engine 内部 seam，不从包顶层公开。
- 不新增运行时 validator 依赖；如认为必须新增，先停止并报告依赖、许可证和替代方案。
- Engine 版本进入 `0.2.0` 开发线但不打 tag、不推送。
- 完整保留 0.1 单 API行为测试；对本文 G/M 编号建立可追踪测试名或 manifest。
- 完成 Ruff、pytest、build、公共导出审计、中文 comment pass 后只提交本地 commit。
