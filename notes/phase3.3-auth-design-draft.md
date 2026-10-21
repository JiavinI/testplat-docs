# Phase 3.3 鉴权与凭证：设计稿

> 状态：**已批准**（2026-10-12）。D1–D6 全部按推荐；用户追加要求：登录换 token 与 Cookie 会话（3.3b）在 3.3a 合并后尽快跟进，3.3a 的接口必须为其预留判别式扩展点（§5.3、§9）。
> 执行手册：`notes/phase3.3-agent-prompts.md`。
> 基线：Engine `v0.2.0`（`4715c61`），Control `main`（`c5a904b`，PR #36 合并后）。
> 权威顺序：本稿（批准后）> `notes/phase3-agent-prompts.md` §10 > `docs/architecture/auth-design.md`。

## 1. 为什么不能直接沿用 `auth-design.md`

旧设计与 Phase 3 已定边界冲突，以下部分作废：

| 旧设计 | 冲突 | 本稿做法 |
|---|---|---|
| Engine 持有 `AuthManager`，读 secrets、做路由 | 手册 §10：Control 拥有秘密来源与生命周期 | Control 解析，Engine 只收最小材料 |
| Engine 执行登录（http_form / cookie / OAuth2） | 首批应只做静态 Bearer/header | 登录策略推迟到 3.3b |
| token 明文写入缓存文件，md5 命名 | 磁盘明文秘密 | 首批无缓存 |
| 401 自动重登并重试 | 隐式重试改变断言语义，且需要登录能力 | 推迟 |
| `http_client.headers.update(...)` 修改共享 client | 跨请求、跨身份串号 | 每次请求显式传 headers |

## 2. 代码事实（决定设计的约束）

1. `ApiExecutionRequest` / `CaseExecutionRequest` 只有 `environment`、`params` 等字段；`to_dict()` **原样输出 `environment`（注释：不脱敏）**。凭证放进 `environment` 或 `params` 都会进入快照，必须走独立字段。
2. 公开请求快照用 `_sanitize_headers`，结果用 `redact_json`。两者都**只按键名**脱敏（`authorization`、`x-api-key`、`token` 等）。如果秘密出现在非敏感键下（例如目标服务把 header 回显到 `$.echo.value`），会原样进入结果、记录和 alias。
3. alias 用 `strip_sensitive_json` 移除敏感键；redact.py 注明“凭证传播须等待独立身份/秘密通道”。本稿就是这条通道。
4. Control 的 `Project` 有 `env`、`assets_root`、`config_root`；环境文件为 `{config_root}/envs/{env}.yaml`，已入 git。仓库里没有 secrets 目录，也没有相应的 gitignore 规则。
5. Engine 执行器每次调用 `client.request(..., headers=resolved["headers"])`，不修改共享 client。注入点就在这里，天然隔离。

## 3. 首批目标（3.3a）与非目标

**目标**：API 请求（单 API 运行和 Case 中的 api 步骤）可以按声明的身份携带静态 header 凭证（Bearer token、API key 或任意自定义 header），并且秘密不进入任何公开面。

**非目标**（各自单独批准）：
- 登录接口换 token（http_form）、Cookie 会话、OAuth2、token 缓存、401 重登 → 3.3b
- 前端管理或录入秘密、数据库存储秘密、秘密加密存储
- Web/UI 通道（随 3.5）
- 每个 Case 步骤切换身份以外的动态凭证（例如由上一步响应提取 token 后作为凭证使用）

## 4. 职责边界

```
Control                                         Engine
───────                                         ──────
auth 配置（入 git，无秘密） ─┐
进程环境变量（秘密来源）   ─┼─ 解析 ─▶ AuthMaterial ─▶ 注入 HTTP headers
资产声明的身份              ─┘   （仅 header 名 + 秘密值）  公开面按名 + 按值脱敏
```

- **Control**：读取 auth 配置，按身份查找环境变量，生成 `AuthMaterial`；校验缺失项（预检 422）；从不记录秘密值。
- **Engine**：不读文件、不读环境变量、不知道身份如何配置；只把 `AuthMaterial` 合并进请求 headers，并保证秘密值不出现在公开面。

## 5. 契约草案

### 5.1 Control 侧 auth 配置（入 git，不含秘密）

```yaml
# {config_root}/auth/{env}.yaml
identities:
  admin:
    headers:
      Authorization: "Bearer ${secret:TESTPLAT_ADMIN_TOKEN}"
  partner:
    headers:
      X-Api-Key: "${secret:TESTPLAT_PARTNER_KEY}"
default_identity: admin        # 可选；缺省表示默认匿名
```

- `${secret:NAME}` 只能出现在 header 值中，NAME 限 `[A-Z][A-Z0-9_]*`，并必须带项目前缀白名单（例如 `TESTPLAT_`），防止任意读取进程环境（如 `AWS_SECRET_ACCESS_KEY`）。
- 秘密值来源：Control 服务进程的环境变量（见决策 D2）。

### 5.2 资产侧身份声明

```yaml
# API 资产
auth: admin      # 使用 identities.admin
auth: none       # 显式匿名
# 不写 → 使用 default_identity；仍无则匿名
```

Case 步骤沿用其 API 资产的声明；首批**不支持**在 Case 步骤上覆盖（见决策 D3）。

### 5.3 Engine 接口（新增可选字段，向后兼容）

资产语法归 Engine，Control 不解析 YAML，因此 Control 事先不知道一次执行会用到哪些身份。做法是 Control 传入**整张身份表**，Engine 在预检阶段按资产声明查表：

```python
@dataclass(frozen=True, slots=True)
class StaticHeaderAuth:                   # 3.3a 唯一可用的 kind
    kind: Literal["static_headers"]
    headers: Mapping[str, SecretStr]      # 秘密值；repr/to_dict 永不输出

@dataclass(frozen=True, slots=True)
class UnavailableAuth:                    # Control 解析失败的身份（变量缺失/为空/过短/不在白名单）
    kind: Literal["unavailable"]
    public_reason: str                    # 只含变量名，不含值

AuthMaterial = StaticHeaderAuth | UnavailableAuth   # 3.3b 追加 LoginTokenAuth / LoginCookieAuth

@dataclass(frozen=True, slots=True)
class AuthTable:
    identities: Mapping[str, AuthMaterial]
    default_identity: str | None = None

ApiExecutionRequest.auth: AuthTable | None = None    # None ≡ v0.2.0 行为
CaseExecutionRequest.auth: AuthTable | None = None
```

- 只有被实际执行的资产用到的身份才会被检查：用到 `UnavailableAuth` → 预检 `invalid_environment` 422；未用到的不可用身份不影响执行。
- 身份解析顺序（Engine 负责）：资产 `auth: none` → 匿名；`auth: <名>` → 查表，不存在则 422；未声明 → `default_identity`，再无则匿名。
- Case 预检阶段对**所有步骤**一次性完成身份检查，不允许执行到一半才发现身份不可用。
- `SecretStr`：Engine 自带的最小包装（不引入 pydantic），`__repr__`/`__str__` 返回 `***`，`__reduce__` 与 JSON 序列化显式抛错，只能通过 `reveal()` 取值，且只有注入点调用。

不变量：
- `AuthMaterial` 的 `repr`、`str`、`to_dict` 只暴露 `identity` 与 header **名**；`SecretStr` 不可被 JSON 序列化（显式抛错，防止误写入）。
- 注入后的 header 与资产自身 header 同名时：**凭证优先**，同时在公开结果中记录 `auth.overridden_headers=[名]`（见决策 D5）。
- 注入的 header 名一律按敏感处理，不论是否在 `_SENSITIVE_KEYS` 中。
- **按值脱敏**：本次执行的全部秘密值（长度 ≥ 8）在公开请求、公开响应、错误 `public_message`、alias 绑定值中替换为 `***`。短于 8 的秘密拒绝使用（预检 422），以免误伤正常文本。
- `ApiExecutionOutcome` 与每个 Case 步骤结果新增公开字段 `auth: {identity, header_names, overridden_headers}` 或 `null`，用于 UI 展示“以 admin 身份运行”（Agent 0 裁决 Q1）。

版本：Engine `0.3.0`（新增公开字段）；Control 约束改为 `>=0.3.0,<0.4`（见决策 D6）。

### 5.4 错误语义（不新增 kind）

| 情况 | 阶段 | kind | HTTP | 记录 |
|---|---|---|---|---|
| 资产声明的身份不存在 | preflight | `invalid_environment` | 422 | 不保存 |
| 环境变量缺失、为空，或名称不在白名单 | preflight | `invalid_environment` | 422 | 不保存 |
| 秘密值短于 8 | preflight | `invalid_environment` | 422 | 不保存 |
| auth 配置 YAML 非法 | preflight | `invalid_environment` | 422 | 不保存 |
| 目标返回 401/403 | — | 按断言处理（业务结果） | 200 | 保存 |

`public_message` 只说明身份名与变量名，**不得包含值**；`log_reason` 同样不含值。

## 6. 安全验收矩阵（每条都要有测试）

1. 秘密值不出现在：`to_dict`、`repr`、公开结果 JSON、执行记录（SQLite 原始行）、Control 与 Engine 日志、OpenAPI 示例、前端截图和 DOM。
2. 目标服务把 header 回显到**非敏感键**下 → 结果、记录、alias 中均为 `***`。
3. 目标服务把秘密放进错误响应正文 → 失败结果中为 `***`。
4. alias 引用回显的秘密字段 → 拿到 `***` 或键被移除，不能拿到原值。
5. 两个身份交替执行（单 API 与 Case 混跑）→ header 不串号。
6. 白名单外的变量名（如 `HOME`、`AWS_SECRET_ACCESS_KEY`）→ 422，且响应中不出现其值。
7. 缺失变量、空值、短值、未知身份、YAML 非法 → 422 且无记录。
8. `auth: none` 时，即使配置了 `default_identity` 也不携带凭证。
9. 资产 header 与凭证 header 同名 → 凭证生效，公开结果中可见覆盖的 header 名、看不到值。
10. Phase 3 回归：资产未声明 `auth`（或声明 `auth: none`）时，行为与 `v0.2.0` 完全一致。资产声明了身份但项目没有 auth 配置 → 422，不静默匿名（Agent 0 裁决 Q2）。

## 6.1 Agent 0 审计裁决（2026-10-12）

| # | 裁决 |
|---|---|
| Q1 | 公开 `auth` 字段为 `{identity, header_names, overridden_headers}` 或 `null` |
| Q2 | Control 无 auth 配置文件时传 `auth=None`。Engine 在 `auth=None` 下遇到资产声明了具体身份 → `invalid_environment` 422；`auth: none` 或未声明 → 匿名，与 `v0.2.0` 一致 |
| Q3 | 凭证与资产同名 header 按**不区分大小写**判定冲突，凭证优先，只发一份；`overridden_headers` 记资产里的原始写法。同一身份内 header 名仅大小写不同 → Control 配置 schema 错误 422 |
| Q4 | alias 绑定值中凭证值按子串替换为 `***`，键保留；这与按键名移除的 `strip_sensitive_json` 是两种语义，须在 `redact.py` 注明 |
| Q5 | 按值替换的集合 = 每个原始变量值 + 每个渲染后的完整 header 值。≥ 8 位检查只针对原始变量值。一个 header 允许多个 `${secret:}`，每个变量单独检查、单独加入替换集合 |
| Q6 | `auth: null` 视为未声明；`auth: ""` 或非字符串 → `invalid_asset`。`none` 是保留名：配置中出现名为 `none` 的身份 → Control 加载即 schema 错误 422（不标为不可用） |
| Q7 | `default_identity` 指向不存在的身份 → Control 加载即 422；指向不可用的身份 → 真正用到时由 Engine 报 422 |

## 6.2 Agent 1 交付裁决（2026-10-12，Engine `f631e2e`）

| # | 裁决 |
|---|---|
| A1 | 接受 `StaticHeaderAuth.secret_values: Sequence[SecretStr]`：Engine 只拿得到渲染后的整段头值，原始变量值必须由 Control 提供才能落实 Q5。Control **必须**为每个身份传入该身份 header 模板中引用的全部变量原始值（去重，每个 ≥ 8），不得依赖“未传时把整段头值当原始秘密”的兜底，兜底只为直接调用 Engine 的场景保留 |
| A2 | 凭证 header 模板**必须**至少含一个 `${secret:}`，否则 Control 加载配置即 schema 错误 422。与身份相关但不含秘密的头（如租户 ID）本批不支持，应写在资产 headers 中 |
| A3 | 接受：`kind` 带默认值放字段末尾；`AuthTable` 接受任意字符串 kind，未知 kind 在用到时预检 422 |
| A4 | 接受：Case 中途因系统错误失败的步骤，`auth` 与 `request` 一样为 `null`（Phase 3 已定的形状）；前端对该步骤显示“—”。已知缺口，非安全问题 |
| A5 | 接受：按值替换不改 JSON 键名；`public_message`/`log_reason` 额外做按值替换兜底 |
| A6 | 接受：`Api.to_dict()` 不含 `auth`（领域模型内部表示，不是公开面） |

## 6.3 Agent 2 交付裁决（2026-10-12，Control `12a8d6a`）

| # | 裁决 |
|---|---|
| B1 | 接受：请求 DTO 构造失败的 422 一律使用固定文案 `执行请求无效：参数或环境配置不合法`，日志只记异常类型名。原先回显 `str(exc)` 的路径实际只会由程序缺陷触发，丢失的诊断信息有限 |
| B2 | 接受并纳入契约：变量值含 `\r`、`\n` 或 NUL → 该身份 `UnavailableAuth`；模板字面量含这些字符 → schema 错误 422（防 header 注入） |
| B3 | 接受：配置缺 `identities` → schema 错误 422 |
| B4 | 接受：`ExecutionAuth` 使用 `extra="forbid"`。Engine 的公开 `auth` 恰好三个键，新增键属于契约变更，必须升 Engine 次版本，而 Control 锁定 `<0.4`，不会被静默触发。代价是届时 Case 详情的 typed `result` 解析失败为 null，升级 Engine 时必须同步改这个模型 |
| B5 | 接受：冻结示例中两条错误文案来自 Fake Engine，由 Agent 4 用真实 Engine 重新生成并替换 |
| B7 | （Agent 3，Control `d80cf63`）A4 的显示规则放宽为统一口径：Case 步骤**只有确实发出了请求**才显示身份或“匿名”；skipped、`request` 为 null（如 variable_resolution）、系统错误归属的失败步骤一律显示“—”。单 API 结果的 null 只有一种含义，显示“匿名” |
| B8 | 接受前端门禁用 route mock 与冻结示例产生身份场景（同 Phase 3 对前端门禁的裁决）。因 mock 数据本身不含秘密，前端门禁的 DOM 秘密检查只是形式检查；真实的身份展示与 DOM 泄密检查由 Agent 4 在真实 Engine 上完成 |
| B6 | 确认占位符划分：语法上是 `${secret:[A-Za-z0-9_]+}` 的占位符，名字不在白名单（如 `HOME`、小写 `home`）→ `UnavailableAuth`，且白名单检查先于读取环境变量，不读其值；不符合占位符语法（如 `${secret:a-b}`、`${other:X}`、未闭合的 `${`）→ schema 错误 422 |

## 7. Agent 分工（详见执行手册）

| Agent | 仓库 | 内容 |
|---|---|---|
| 0 | 契约 | 冻结 §5 DTO、配置 schema、golden fixtures 与 §6 用例清单 |
| 1 | Engine | `AuthMaterial`、注入、按值脱敏、公开 `auth` 字段，发布 0.3.0 |
| 2 | Control 后端 | auth 配置加载、白名单解析、预检错误、Case 身份映射、日志约束 |
| 3 | Control 前端 | 结果与历史中显示身份名和 header 名；不提供秘密录入 |
| 4 | 集成 | 真实 Engine + 本地回显服务的泄密矩阵、浏览器截图、干净安装 |

主线程保留：秘密来源与白名单裁决、版本合同、最终泄密审查、推送/PR/tag 授权。

## 8. 已批准的决策

| # | 问题 | 推荐（已采纳） | 备选 |
|---|---|---|---|
| D1 | 首批范围 | 只做静态 header 凭证（3.3a） | 同时做 http_form 登录换 token |
| D2 | 秘密来源 | Control 进程环境变量 + `TESTPLAT_` 前缀白名单 | gitignore 的 `{config_root}/secrets/{env}.yaml`；或两者都支持 |
| D3 | 身份声明粒度 | API 资产 `auth:` + 环境级 `default_identity`；Case 步骤不覆盖 | Case 步骤可写 `auth:` 覆盖 |
| D4 | 错误 kind | 复用 `invalid_environment`（不改错误合同） | 新增 `invalid_auth` |
| D5 | header 冲突 | 凭证优先并公开记录被覆盖的 header 名 | 冲突即预检 422 |
| D6 | 版本 | Engine 0.3.0，Control 约束 `>=0.3.0,<0.4` | Engine 0.2.1（新增字段视为兼容） |

## 9. 3.3b 预告：登录换 token 与 Cookie 会话（3.3a 合并后立即设计）

3.3a 只做到下列预留，不实现任何登录逻辑：

- `AuthMaterial` 是带 `kind` 的判别联合；Engine 对未知 kind 报预检 `invalid_environment`，不静默降级为匿名。
- 已知的 3.3b 设计方向（届时单独批准）：
  - 登录请求复用已有 API 资产（例如 `login_api: api.auth.login`），用户名/密码以 `SecretStr` 作为该资产的 params 传入，复用现有执行器、断言与脱敏。
  - `LoginTokenAuth`：从登录响应按 JSONPath 提取 token，按 header 模板注入；提取到的 token 运行期加入按值脱敏集合。
  - `LoginCookieAuth`：需要**每次执行、每个身份独立**的 cookie jar。当前执行器每次 `execute` 新建 `HttpxClient`，Case 内跨步骤不保留 cookie；注入的共享 client 则有跨身份串号风险。3.3b 需引入按 (运行, 身份) 隔离的 client/jar，这是 3.3b 的主要结构改动。
  - 同一次 Case 运行内每个身份最多登录一次（内存缓存，运行结束即丢弃）；跨运行缓存、磁盘缓存、401 重登继续推迟。
  - 登录失败的语义（业务失败还是预检错误、是否保存记录）在 3.3b 契约中裁决。
