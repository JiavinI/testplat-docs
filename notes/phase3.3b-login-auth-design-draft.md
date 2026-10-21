# Phase 3.3b 登录鉴权：设计稿 v2

> 状态：**已批准**（2026-10-13）。D7–D15 见 §10；E1–E4 全部按推荐（§11）。执行手册：`notes/phase3.3b-agent-prompts.md`。
> 基线：Engine `origin/main @ 8564040`（tag `v0.3.0`），Control `origin/main @ d29d615`（PR #37 合并后）。
> 前置设计：`notes/phase3.3-auth-design-draft.md`（3.3a，已实现）。本稿沿用其边界与裁决，只在明确写出的地方扩展。
> 需求证据：旧项目 `/Users/jiavin/Developer/products/testplat-new`（只读）。

## 1. 用户要求（2026-10-13 定稿）

1. 登录方式分**通用**与**特殊**两层，通用层覆盖“调登录接口成功后拿到鉴权信息、后续请求带上”，特殊层为项目专属登录（验证码、签名等）预留扩展。
2. 一个 Case 可用多个角色；**用到某角色时**检查是否已有有效鉴权信息，没有或失效才登录；不预先登录所有角色。
3. 登录失败必须作为**独立、醒目的失败类别**呈现，不得被断言失败、系统错误等掩盖。
4. 鉴权信息的提取方式多样，必须分通用（可配置的提取规则）与特殊（插件直接产出）。
5. 所有常见请求体类型都要支持。

架构裁决（用户已同意）：**方案 C**——Engine 定义登录策略插件协议并内置通用策略；特殊策略以独立包或项目目录中的插件实现，由 Control 在创建 Engine 时注册。Engine 核心不依赖 Redis、浏览器等外部系统。

### 1.1 三档划分（2026-10-13 用户确认）

| 档 | 分类 | 定义 | 实现 |
|---|---|---|---|
| 1 | **通用** | 一个登录请求（`steps` 恰好一项）+ 一次提取（`extract` 恰好一条，`source` 为 `body_json` 或 `response_header`，`as: token`）+ 一处注入（`inject.headers` 引用 `${login:token}`） | `generic_flow` 单步形式，纯配置 |
| 2 | **特殊（配置型）** | 多步流程、CSRF 预取、Cookie 会话、probe、多个提取值、`cookies`/`html_meta` 提取来源 | `generic_flow` 多步形式，纯配置，不写代码 |
| 3 | **特殊（代码型）** | 验证码走 Redis、签名、任何配置表达不了的流程 | `LoginStrategy` 插件（方案 C） |

第 1 档与第 2 档共用同一套配置语法；“通用”的简单性靠文档中的标准模板与 Control 校验提示体现。旧项目对应：`http_form` → 第 1 档；`cookie_session` → 第 2 档；`vcode_redis` → 第 3 档。

## 2. 旧项目证据（决定通用层边界）

| 旧实现 | 分类 | 要点 |
|---|---|---|
| `http_form` | 通用 | 请求体位置 `json`/`data`/`params`；`token_path` 取 token；`extras_paths` 取附属值（user_id）；注入模板 `X-TOKEN: "{token}"` 等 |
| `cookie_session` | 通用骨架 | 多步 flow（预取 CSRF/JSESSIONID → 登录 → probe）；提取来源 `body_json`/`cookies`/`html_meta`/`response_header`；步骤级 `success` 与 `retry`；登录后 `probe` 验证会话 |
| `vcode_redis` | 特殊 | createVCode → Redis 读验证码 → 表单登录；密码来自 secrets 额外字段 |
| 按需登录 | 通用 | `ApiAuthProvider.headers()` 先查缓存，无则登录（`api_auth_provider.py:29-60`） |
| 有效性 | 通用 | TTL + `safety_margin_s`；Cookie 型靠 `probe` |
| 401 自愈 | 通用 | 作废缓存、重登、只重发一次（`executor.py:1256-1320`）——**本批不做**，见 §9 |
| 身份切换 | 通用 | 资产步骤 `as: <身份>`（30 处） |

## 3. 代码事实（当前两仓）

1. Engine 只用 `json=` 发请求体，`request.type` 被忽略。
2. 生产路径每次 API 调用新建 `HttpxClient`，Case 步骤之间不保留 Cookie，也不存在跨执行串号。
3. `HttpxClient` 用 `dict(response.headers)` 读响应头，多个 `Set-Cookie` 合并；Cookie 必须走 httpx cookie jar。
4. 简化 JSONPath 只支持 `$.a.b` 点路径。
5. 非 `preserve_types` 模式下未解析的 `${var}` 原样发出；登录请求必须禁止。
6. 3.3a 的 `AuthMaterial` 是带 `kind` 的判别联合，Engine 对未知 kind 预检 422；`AuthTable` 接受任意字符串 kind。
7. 3.3a 的公开 `auth` 字段恰好三键，Control `ExecutionAuth` 为 `extra="forbid"`（B4）。

## 4. 职责边界

```
Control                                               Engine 核心
───────                                               ──────────
auth 配置（入 git，无秘密）  ─┐                        LoginStrategy 协议 + 注册表
进程环境变量（秘密来源）     ─┼─ 解析 ─▶ LoginAuth DTO ─▶ 内置策略：generic_flow
插件注册（方案 C）           ─┘   （策略名、flow/params、          （第 1 档单步 / 第 2 档多步、提取、probe）
                                     提取规则、注入模板）           会话池：按 (执行, 身份) 保存鉴权状态
                                                                   按需登录 + 有效性判断
                                                                   公开面：登录记录、login_failed、动态脱敏
                                                      ▲
特殊策略插件包（独立 pip 包或项目目录）────────────────┘
  实现 LoginStrategy，自带 Redis 等依赖；Control 启动时注册到 Engine
```

- **Control**：读配置、读环境变量、构造 DTO、注册插件；不发 HTTP、不拿 token/Cookie 原值。
- **Engine 核心**：执行登录、保存会话状态、注入、脱敏；只依赖 httpx。
- **插件**：拿到 `LoginContext`（隔离的 HTTP client、已解析的账号材料、策略参数、`connections` 配置），返回命名值和/或 Cookie。插件不能访问资产、不能触碰业务步骤。

## 5. 契约草案

### 5.1 Control 侧 auth 配置

```yaml
# {config_root}/auth/{env}.yaml
identities:
  admin:                                   # 3.3a 静态 header，保持不变
    headers:
      Authorization: "Bearer ${secret:TESTPLAT_ADMIN_TOKEN}"

  buyer:                                   # 第 1 档通用：单步登录换 token（标准模板；user_id 这条 extract 使其成为第 2 档，仅作示例）
    login:
      strategy: generic_flow               # 省略时默认 generic_flow
      host: main                           # 登录目标 host（来自环境配置 api_hosts）
      steps:
        - name: login
          method: POST
          path: /api/user/login
          body: { type: json, fields: { username: "buyer001", password: "${secret:TESTPLAT_BUYER_PASSWORD}" } }
          success: { path: $.code, value: 0 }
          extract:
            - { source: body_json, path: $.data.token, as: token }
            - { source: body_json, path: $.data.userId, as: user_id, optional: true }
      validity:
        ttl_seconds: 14400                 # 可选；从响应取过期时间见 §5.4
      inject:
        headers:
          X-TOKEN: "${login:token}"
          X-USER-ID: "${login:user_id}"

  seller:                                  # 第 2 档特殊（配置型）：多步 Cookie 会话
    login:
      host: admin
      steps:
        - name: bootstrap
          method: GET
          path: /login
          extract:
            - { source: html_meta, name: csrf-token, as: csrf }
        - name: submit
          method: POST
          path: /login
          headers: { Referer: "${login_host}/login" }
          body: { type: form, fields: { _csrf: "${login:csrf}", username: "seller001", password: "${secret:TESTPLAT_SELLER_PASSWORD}" } }
          success: { status_code: 302 }
      session: cookie                      # 后续请求携带该身份 cookie jar
      probe:                               # 可选；登录后立即验证
        method: GET
        path: /nx/withdrawView
        success: { status_code: 200, body_not_contains: '<form id="loginFrom"' }
      inject:
        headers:
          X-CSRF-Token: "${login:csrf}"

  df_admin:                                # 第 3 档特殊（代码型）：插件策略
    login:
      strategy: vcode_redis                # 必须已注册，否则预检 422
      host: daifa_admin
      params:                              # 原样交给插件；插件自行校验
        create_vcode_path: /createVCode
        redis_connection: redis_main
        redis_key_prefix: "gsb:vcode:prefix:"
        username: "admin01"
        password: "${secret:TESTPLAT_DF_ADMIN_LOGIN_PASSWORD}"
      inject:
        headers:
          X-TOKEN: "${login:token}"
          X-USER-ID: "${login:user_id}"

default_identity: admin

connections:                               # 插件可用的非秘密连接信息；密码仍走 ${secret:}
  redis_main: { host: 10.0.0.5, port: 6379, db: 2, password: "${secret:TESTPLAT_REDIS_MAIN_PASSWORD}" }
```

校验规则（Control 加载期，违者 schema 错误 422）：
- 一个身份只能是 `headers`（3.3a）或 `login`。
- `${secret:}` 规则完全沿用 3.3a（白名单、≥ 8、控制字符、语法错误即 422）。模板中可出现的占位符只有 `${secret:NAME}`、`${login:名字}`、`${login_host}`；其它 `${` 一律 422。
- `inject.headers` 的值可为字面量、`${login:名字}`、`${secret:NAME}` 任意组合；每个 login 身份至少一个 inject 值引用 `${login:}` 或声明 `session: cookie`（Q1/Q2）。引用的名字必须在某个 `extract.as` 中声明（插件策略例外，由插件 `validate` 判定）。
- `steps[].body.type ∈ {json, form, query, raw}`；`multipart` 本批不支持。
- `session: cookie` 与 `inject` 可同时出现（Cookie + CSRF 头是常见组合）。
- Control 加载时按 §1.1 判定每个 login 身份的档位（`tier: generic | configured | plugin`），作为非秘密信息写入 DTO 并在公开 `auth`/`logins` 中展示，便于用户一眼看出某身份是否超出了“通用”范围。

### 5.2 资产侧身份声明

沿用 3.3a：API 资产 `auth: <身份>` / `auth: none` / 未声明走 `default_identity`。Case 步骤仍不能覆盖（D3）。

### 5.3 Engine 接口（0.4.0）

```python
class LoginAuth:                       # kind = "login"
    strategy: str                      # "generic_flow" 或插件名
    host: str
    flow: LoginFlow | None             # generic_flow 必填；插件策略可为 None
    params: Mapping[str, str | SecretStr]   # 插件参数；SecretStr 值全部进入按值替换
    secret_values: Sequence[SecretStr]      # 同 A1
    validity: Validity                 # ttl_seconds / expires_from
    session_cookie: bool
    probe: ProbeSpec | None
    inject_headers: Mapping[str, Template]  # 字面量 + ${login:名字} 片段
    kind: Literal["login"] = "login"

class LoginStrategy(Protocol):         # 插件协议（Engine 公开）
    name: str
    def validate(self, auth: LoginAuth) -> None: ...        # 预检期，不发请求
    async def login(self, ctx: LoginContext) -> LoginResult: ...

class LoginContext:                    # Engine 交给策略
    http: HttpClient                   # 该身份专用、本次执行内存活的 client（自带 cookie jar）
    host_url: str                      # 已解析的登录 host
    params: Mapping[str, str | SecretStr]
    connections: Mapping[str, Mapping[str, str | SecretStr]]
    record: Callable[[PublicRequest, PublicResponse], None]   # 登录过程记录（Engine 负责脱敏）

class LoginResult:
    values: Mapping[str, str]          # 命名值；Engine 全部视为秘密加入按值替换集合（≥ 8 的）
    expires_at: float | None           # 策略若能得知过期时间
```

- `AuthMaterial = StaticHeaderAuth | UnavailableAuth | LoginAuth`。
- `create_engine(*, login_strategies: Sequence[LoginStrategy] = ())`：Control 启动时把已安装的插件传入。内置 `generic_flow` 总是注册。
- 新 DTO 的 `repr`/`to_dict`/pickle 规则与 3.3a 相同，只暴露策略名、host、步骤名、提取名、header 名。

### 5.4 执行模型

```text
预检（零副作用）
  └ 解析本次执行用到的身份；对 login 身份：策略已注册、策略 validate() 通过、host 可解析、
    inject 引用的名字可被 extract/插件产出覆盖。失败 → invalid_environment 422，不保存。
业务步骤 N
  └ 解析身份 → 查会话池[(执行, 身份)]
       ├ 有且有效 → 直接注入 / 携带 cookie
       ├ 无或失效 → 登录（generic_flow 或插件）→ probe（若配置）→ 写入会话池 → 注入
       └ 登录失败 → 本步骤 failed，error.kind = login_failed；后续步骤 skipped(skip_reason=login_failed)
```

**有效性判断**（本批只在单次执行内生效）：
- `validity.ttl_seconds`：自登录成功起算，提前 30 秒判失效（固定 safety margin）。
- `validity.expires_from: { source: body_json, path: $.data.expiresIn, unit: seconds | epoch_ms | epoch_s }`：从登录响应取过期时间，优先于 ttl。
- Cookie 身份：jar 中会话 Cookie 的 `expires` 过期即失效。
- 都未配置 → 本次执行内一直有效。
- 失效后再次用到 → 自动重新登录（这是“按需检查有效”的另一半）。

**会话池**：每次执行一个池，执行结束整体关闭。login 身份各自一个 `HttpxClient`（cookie jar 随之隔离）；匿名、静态 header 身份保持现状（每次调用新建）。Cookie 身份的所有响应（含业务步骤）都更新其 jar（v1 D12 保留）。

### 5.5 公开结果

- 新增 `logins: [ { identity, strategy, status, duration_seconds, steps: [ {name, request, response} ], probe, error } ]`，按发生顺序；请求/响应全部脱敏。
- 公开 `auth` 字段扩展为 `{identity, kind, header_names, overridden_headers, cookie_names, login_ref}`，`login_ref` 指向本步骤使用的那次登录在 `logins` 中的下标（同一身份多次重登时可区分）。
- 这是 Q1 三键结构的契约变更；Control `ExecutionAuth` 同步改（B4）。

### 5.6 错误语义

| 情况 | 结果 | HTTP | 记录 |
|---|---|---|---|
| 策略未注册 / validate 失败 / host 不可解析 / inject 引用缺失 | `invalid_environment` | 422 | 不保存 |
| 登录步骤 success 不满足、提取不到值、probe 失败 | 当前步骤 failed，`error.kind = login_failed`，`error.login = {identity, failed_step, reason}`；后续 `skipped(login_failed)` | 200 | 保存 |
| 登录步骤超时 / 连接失败 | **同上归为 `login_failed`**，`reason = timeout` / `transport`，不映射为 504/502（避免登录失败被当成系统错误） | 200 | 保存 |
| 插件抛出任意异常 | `login_failed`，`reason = strategy_error`，只记异常类型名 | 200 | 保存 |
| 业务请求 401/403 | 按断言处理，不重登（§9） | 200 | 保存 |
| 业务请求超时 / 连接失败 | 沿用 Phase 3 系统错误 | 504 / 502 | 保存 failed |

契约扩展：错误 kind 增加 `login_failed`；`skip_reason` 增加 `login_failed`。前端对 `login_failed` 使用独立的醒目样式，并直接展开登录过程面板。

### 5.7 请求体类型（资产 DSL，影响所有 API）

`request.type`：
- `json`（默认）→ `json=body`
- `form` → `data=body`（一层字符串键值）
- `query` → `params=body`（一层字符串键值；与 URL 自带 query 合并）
- `raw` → `content=body`（字符串），需 `request.content_type`
- `multipart` → 本批不支持，声明即 `invalid_asset`

登录 flow 的 `steps[].body.type` 使用同一枚举与同一实现。

## 6. 安全验收矩阵（在 3.3a 十条基础上新增）

11. 登录请求中的密码、登录响应中的 token/附属值、`Set-Cookie` 会话值，不出现在公开结果、记录、日志、DOM、截图。
12. 业务响应回显 token 或 Cookie 值（非敏感键下）→ `***`；alias 引用 → `***`。
13. 同一 Case 内两个 login 身份交替 → 各自携带自己的 token/Cookie，不串号；匿名步骤不携带 Cookie。
14. 两次独立执行之间不共享鉴权状态（第二次一定重新登录）。
15. 按需登录：身份首次用到的步骤之前没有发出任何登录请求；同一身份第二次用到不再登录。
16. 失效重登：`ttl_seconds` 很短时，后续步骤触发重新登录，`logins` 出现两条，`login_ref` 区分。
17. 登录失败（断言 / 提取 / probe / 超时 / 连接 / 插件异常）→ 200、`login_failed`、归属到触发步骤、后续 `skipped(login_failed)`、登录过程可见且脱敏。
18. 插件策略：注册的假插件（测试用）产出命名值并被注入、被脱敏；未注册的策略名 → 422 且无请求发出。
19. `connections` 中的密码不出现在任何公开面。
20. 请求体 `form` / `query` / `raw` 的公开快照中秘密为 `***`；`multipart` → `invalid_asset`。
21. 3.3a 回归：只用静态 header 的项目行为与 Engine 0.3.0 一致（公开 `auth` 多出的键除外）。

## 7. Agent 分工（批准后写手册）

| Agent | 仓库 | 内容 |
|---|---|---|
| 0 | 只读 | 审计执行器 client 生命周期改造点、会话池接入点、所有输出路径、`request.type` 影响面、旧项目 `generic_flow` 需覆盖的配置样例；提出 fixtures |
| 1 | Engine | `LoginAuth`/`LoginStrategy`/`LoginContext` 协议、`generic_flow`（多步、四种提取、success、probe）、会话池、有效性判断、按需登录、`login_failed`、动态脱敏、`request.type`、公开 `logins` 与扩展 `auth`，发布 0.4.0 |
| 2 | Control 后端 | auth 配置 `login`/`connections` 段解析、插件发现与注册（entry point 或配置列表）、OpenAPI、`ExecutionAuth` 扩展、顺带修 `main.py:23` |
| 3 | Control 前端 | `login_failed` 醒目样式、登录过程面板、`kind`/Cookie 名/`login_ref` 展示、`skipped(login_failed)` 原因 |
| 4 | 集成 | 本地目标服务：JSON 登录、表单登录、CSRF 预取 + Cookie 登录 + probe、短 TTL、回显 token；测试用假插件包；真实矩阵覆盖 §6 第 11–21 条与 3.3a 回归；泄密搜索；干净安装 |
| 5（可选） | 插件示例 | 独立包 `testplat-login-vcode-redis`：按方案 C 实现旧项目 `vcode_redis`，作为特殊策略范例；需单独批准 |

## 8. 风险

| 风险 | 应对 |
|---|---|
| 会话池改造影响全部执行路径 | 只有 login 身份走会话 client；其余路径保持每次新建；Phase 3 / 3.3a 全量回归兜底 |
| 按需登录使登录失败发生在 Case 中途 | 归属到触发步骤，形状与系统错误归属一致；前面已执行步骤的结果保留 |
| 登录超时不再映射 5xx | 用户明确要求凸显登录失败；`error.login.reason` 保留 `timeout`/`transport` 供排查 |
| 插件可执行任意代码 | 插件注册只由 Control 启动配置决定，不可由资产或请求触发；插件拿不到资产、业务 client、其它身份状态 |
| token 提取后、加入替换集合前的窗口 | 不变量写死顺序；用回显 token 的目标服务测试 |
| `${login:名字}` 与 `${secret:}`、资产 `${var}` 三套占位符并存 | 作用域不同：资产只认 `${var}`；auth 配置只认 `${secret:}`/`${login:}`/`${login_host}`；Agent 0 核对解析器不会互相误吞 |

## 9. 明确不做（各自单独批准）

- 跨执行缓存 token/Cookie（内存或磁盘）
- 401 自动重登并重发
- 登录步骤级 `retry`（旧项目用于“操作频繁”）
- `multipart` 请求体
- 浏览器登录（3.5）
- 前端录入或管理秘密

## 10. 决策记录

| # | 问题 | v1 推荐 | v2 裁决 |
|---|---|---|---|
| D7 | 登录定义 | 复用 API 资产 | auth 配置内 `login.steps` flow（通用）+ 策略插件（特殊）；不复用 API 资产 |
| D8 | 登录时机 | 执行前统一登录 | 用到时按需登录，带有效性判断与失效重登 |
| D9 | 失败语义 | 业务失败、全部 skipped | 独立 `login_failed`，归属触发步骤，含超时/连接失败 |
| D10 | 提取方式 | 响应体或响应头二选一 | 四种来源、多命名值的 `extract` 规则；插件直接产出 |
| D11 | 请求体 | json + form | json / form / query / raw；multipart 后置 |
| D12 | Cookie 更新范围 | 全响应更新 | 保留 |
| D13 | 跨执行缓存 | 不做 | 不做；但有效性判断在本批建立 |
| D14 | 版本 | Engine 0.4.0 | 保留；契约变更范围更大 |
| D15 | 特殊策略扩展点 | — | 方案 C：Engine 定义协议，插件独立包/目录，Control 启动时注册（用户已同意） |

## 12. Agent 0 审计裁决（2026-10-13）

| # | 裁决 |
|---|---|
| Q1 | `headers` 与 `login` 仍互斥（避免两套语法）。`inject.headers` 的值可以是字面量、`${login:名字}`、`${secret:NAME}` 的任意组合；约束从“每个值至少一个 `${login:}`”放宽为“**每个 login 身份至少有一个 inject 值引用 `${login:}`，或声明 `session: cookie`**”。inject 中的字面量 header（如 `gsb_csrf` 的 `X-SOURCE`）与其它注入头一样按名遮蔽 |
| Q2 | `session: cookie` 时 `inject` 可省略；身份必须满足 Q1 的“至少一个 `${login:}` 或 `session: cookie`” |
| Q3 | 登录步骤与 probe 未写 `success` 时默认“状态码 < 400”；业务资产保持 2xx 默认不变。登录步骤一律 `follow_redirects=False`，302 必须可见 |
| Q4 | 显式写的 `Content-Type` 优先；`form` / `raw` 只在未显式声明时自动设置。`raw` 必须有 `content_type` 或显式 `Content-Type` 头，否则 schema 错误 |
| Q5 | 资产任何字段出现 `${secret:`、`${login:`、`${login_host}` → 加载即 `invalid_asset`。资产其它未解析 `${var}` 的既有行为（单 API 原样发出 / Case 报 variable_resolution）本批不改 |
| Q6 | 会话池经 `execute_api` 内部关键字参数与 `StepExecutionContext` 传递，不进任何公开 DTO |
| Q7 | 插件与 `generic_flow` 经 `ctx.record` 写入的登录请求/响应先缓存；`login()` 返回或抛出后，命名值与 jar Cookie 值加入执行级替换集合，再统一脱敏落定到 `logins`。登录**失败**时额外启用启发式：登录响应中长度 ≥ 20 的字符串叶子值一律 `***`（token 可能存在于配置错误的路径下，此时无法按值识别），短文本与错误消息保留 |
| Q8 | `query` 类型请求的公开快照：`url` 不含 query 参数；新增 `request.query` 对象单独展示并按名 + 按值脱敏。`url` 本身仍参与按值替换（M2） |
| H1 | 按值替换集合提升为**执行级**（挂在会话池上），每次生成快照用全部身份命名值、Cookie 值、登录参数秘密的并集 |
| H2 | 扩展内部端口 `HttpResponse`：新增 `cookies`（name/value/domain/path/expires 列表）与 `text`；`HttpxClient` 暴露 jar。Control 不受影响（内部端口） |
| H3 | 登录子流程内捕获 timeout/transport，转为 `login_failed(reason=timeout|transport)`，不进入 `_map_exception` |
| H4 | `skip_reason` 四处写死 `fail_fast` 的位置全部改为枚举 `fail_fast | login_failed`（Engine `case_runner`、`interface`；Control `case_schemas`；前端 `caseParse`） |
| H5 | Control `parse_execution_auth` 校验失败时不再静默返回 None：记一条 warning 日志（只含记录 id）并返回 None；Agent 2 同步扩展 `ExecutionAuth` |
| gsb_csrf | 经 Q1 放宽后可表达为第 2 档；五个旧 profile 全部可用配置表达，验证码登录 `df_admin_api` 归第 3 档（本批不做） |

## 13. Agent 1 交付裁决（2026-10-13，Engine `7a7ebb8`）

| # | 裁决 |
|---|---|
| A1 | 接受 `LoginContext` 增加 `identity`、`flow`、`timeout_seconds`，以及 `record(request, response, *, name=None)`。这是插件可用性所必需的，记入 0.4.x 稳定协议 |
| A2 | 接受插件 `login()` 整体受一份 `timeout_seconds` 限时。Control 文档写明：多步插件应自行在内部拆分超时，不要依赖 Engine 再给一层总预算 |
| A3 | 接受 `token_too_short` 只约束名为 `token` 的值；其它命名值长度不限，< 8 的不进入按值替换。Control 配置文档提示：短附属值（如 user_id）不要当秘密，需要脱敏的附属值必须 ≥ 8 |
| A4 | 接受重登丢弃旧 jar、换全新 client。Cookie 过期判定按“jar 中任意一个带 expires 的 Cookie 到期即失效”，记入契约 |
| A5 | 接受 `expires_from` 取不到数字时静默回退 ttl / 插件 `expires_at` / 本次执行内一直有效。不判失败，避免登录成功却因过期字段缺失变成 `login_failed` |
| A6 | 接受 `login_failed` 步骤的 `request`/`response` 为 null、`auth` 带 `login_ref`、`error` 无 `stage`。登录过程只在 `logins` 里看 |
| A7 | 接受 `logins` 只挂在 Case / 单 API 结果与 `partial_outcome` 上，不在每个步骤重复 |
| A8 | 接受 `logins` 与 `body_type` 始终输出（默认 `[]` / `"json"`）。这是 0.3.0 的可见差异，已写在 `COMPATIBILITY-0.4.md` |
| A9 | 接受测试 seam `client_factory`。注入共享 `http_client` 时 login 身份之间无 jar 隔离，**仅限测试**；生产路径 `create_engine()` 必须走 `client_factory` 每次新建。Control 不得注入共享 client |
| A10 | 接受 Q1/Q2 的校验分层：DTO 看到 SecretStr 模板先放行，`generic_flow.validate` 检查引用是否满足。Control 加载期仍按 §12 Q1 做身份级校验 |
| U1 | **接受当前脱敏范围，附硬约束**：登录响应里未提取的字段不做按值替换，只做按键脱敏；登录失败时另加 ≥ 20 字符叶子替换。配置必须把真正的秘密提取为命名值。文档写明：未提取字段即使碰巧等于 token，也会以原值出现在成功登录记录里。本批不改实现 |
| U2 | 本批不做插件单独总超时；见 A2 |
| U3 | Agent 2 必须配合：构造 `AuthTable.connections`；把公开 `auth` 扩成七键；处理 `logins`、`body_type`、`query` 三个新字段 |

## 14. Agent 2 交付裁决（2026-10-13，Control `5ae27d0`，主线程补丁未提交）

| # | 裁决 |
|---|---|
| B1 | 接受临时 `file://` 锁定 Engine `7a7ebb8`；合并前必须改为远程精确 rev（需用户授权推送 Engine） |
| B2 | 接受 `connections` 任一 `${secret:}` 不可用 → 整次加载 `invalid_environment` 422。代价：未用该连接的身份也被阻断；本批无插件包（E4），影响可控。插件落地时重新评估是否收窄到依赖该连接的身份 |
| B3 | 接受登录步骤 `path` 禁止 `${secret:}`（Engine `LoginStep.path` 为明文 str） |
| B4 | 接受按项目 `for_login_plugins` 切换并缓存引擎。主线程审查发现：白名单列出但未安装时异常在路由层未捕获 → **500**。已补 `_engine_for_auth` 捕获 `EngineError` → 按错误表 503、安全文案、不写记录，并加测试 `test_whitelisted_plugin_not_installed_is_503_without_record` |

## 15. Agent 4 集成审查（2026-10-13，Control `a909c59`，Engine `7a7ebb8`）

| # | 结论 |
|---|---|
| I1 | 通过。主线程复跑：Control ruff / 327 pytest / lock；真实矩阵（输出重定向到临时目录）3.3a 28/28、3.3b 25/25（L11–L21、E1/E1b、P1–P5）、两组浏览器截图、Phase 3 回归、泄密搜索 unexpected_hits=0（含 SQLite 文件字节与逐行） |
| I2 | Agent 3 遗留的历史列表 `login_failed` 缺口已由 `f89ea77` 补齐（列表项从 result_json 推导，OpenAPI 已重冻） |
| I3 | 非阻塞缺口：`auth_leak_search.py` 只搜静态假秘密，未包含目标服务动态签发的 `tok-*` / `sess-*`。主线程对日志、报告、DOM 证据做了正则补搜，命中 0；SQLite 因 trap 清理未补搜（其内容即公开结果，已由 L11/L11c 覆盖）。下批把动态 token 正则加入搜索目标 |
| I4 | `phase3_3-real-engine/` 旧截图与报告被重生成：差异为 auth 七键与折叠按钮文案「查看 header / Cookie 名」，属预期契约变化，接受 |
| B | BLOCKED：Engine 推送 + `v0.4.0` tag → Control 改远程精确锁定 → 远程锁干净安装复跑，需用户授权 |

## 11. 已批准的补充决策

| # | 问题 | 裁决 |
|---|---|---|
| E1 | 插件发现方式 | Python entry point（group `testplat_engine.login_strategies`）+ Control 配置白名单 `login_plugins: [vcode_redis]`，两者都满足才注册；只安装不列入白名单不生效 |
| E2 | 登录超时 / 连接失败 | 归为 `login_failed`（200，保存），`error.login.reason` 为 `timeout` / `transport`；不映射 504/502 |
| E3 | 过期安全余量 | 固定提前 30 秒，不开放配置 |
| E4 | `vcode_redis` 插件示例包 | 本批不做；通用层与协议稳定后单独立项 |
