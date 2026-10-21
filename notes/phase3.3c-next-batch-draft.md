# Phase 3.3c 设计稿 v2：验证码 Redis 登录插件（已批准方向，2026-10-13）

基线：Engine `v0.4.0`（`f1a0c875`）**不改、不发版**；Control main `789d6b6`。

## 1. 范围

| 做 | 不做（后续批次） |
|---|---|
| `vcode_redis` 插件（统一插件包 `testplat-login-plugins`） | 鉴权过期自动重登 + API 重发（随跨执行缓存一起做，见 §7） |
| Control：连接秘密只阻断引用它的身份（重审 §14 B2） | 跨执行缓存、multipart、登录步骤 retry |
| 集成：本地 Redis + createVCode 目标端点 + 泄密搜索加动态值（§15 I3） | 插件发布到 PyPI、Redis 写操作、Cluster / Sentinel |
| 删除已合入的 3.3b 分支（附在执行 Agent 任务里，§8） | Engine / 前端 / OpenAPI 改动 |

## 2. 关键约束（来自 v0.4.0 代码，主线程已核对）

1. `LoginAuth.params` 是**扁平** `Mapping[str, str | SecretStr]`，不能嵌套；插件参数用点号键表达结构（§3）。
2. 插件失败归类只有三种：内置 `TimeoutError` / httpx 超时 → `timeout`；内置 `ConnectionError` / httpx 传输异常 → `transport`；其它任何异常 → `strategy_error`（公开面只记异常**类型名**）。`LoginStepError` 是 Engine 内部类，插件**不得**导入。
   - 因此 redis-py 的异常（不是内置异常的子类）必须由插件转换成内置 `ConnectionError` / `TimeoutError`。
   - 业务失败（验证码为空、登录非 2xx 等）用插件自定义异常，类型名要能说明原因（§4.3）。
3. `ctx.record(request, response, name=...)` 记录的内容**由插件构造**，Engine 只再做按键 / 按值脱敏。插件可以、也必须在记录前自己遮蔽验证码、uuid、密码。这样短验证码（< 8 位）也不会以原值出现。
4. `LoginResult.values` 全部视为秘密（≥ 8 才进按值替换集合）；`token` 必须 ≥ 8，否则 `token_too_short`。inject 引用的名字没产出 → `extract_missing`。
5. 插件 `validate(auth)` 拿不到 `connections`；连接是否存在由 Control 加载期检查（§5）。
6. `SecretStr.reveal()` 的 AST 白名单只约束 Engine 仓；插件包要有自己的同类约束（§4.4）。
7. 整个 `login()` 受一份 `timeout_seconds` 限时（§13 A2）。

## 3. auth 配置（等价旧项目 `df_admin_api`）

```yaml
login_plugins: [vcode_redis]
connections:
  redis_main:
    driver: redis
    host: 127.0.0.1
    port: 6379
    database: 0
    password: ${secret:TESTPLAT_REDIS_MAIN_PASSWORD}   # 可省略（无密码 Redis）
identities:
  df_admin:
    login:
      strategy: vcode_redis
      host: daifa_admin
      params:
        redis_connection: redis_main
        redis_key_prefix: "gsb:vcode:prefix:"
        create_vcode.path: /createVCode
        create_vcode.method: POST            # 可选，默认 POST
        create_vcode.body_type: json         # json | form | query，默认 json（空对象）
        uuid_path: $.data.uuid               # 可选，默认 $.data.uuid
        login.path: /login
        login.method: POST                   # 可选，默认 POST
        login.body_type: form                # json | form，默认 form
        field.username: admin_user           # field.* 原样进登录请求体
        field.password: ${secret:TESTPLAT_DF_ADMIN_PASSWORD}
        field.source: daifa
        field.userType: "1"
        code_field: code                     # 可选，默认 code
        user_key_field: userKey              # 可选，默认 userKey
        extract.token: $.data.token          # 必填
        extract.user_id: $.data.userId       # 可选，extract.* 产出同名命名值
      validity: { ttl_seconds: 72000 }
      inject:
        headers:
          X-TOKEN: ${login:token}
          X-USER-ID: ${login:user_id}
```

- 旧项目的 `password_account_extra_field` 不再需要：`field.password` 直接写对应的 `${secret:}`。
- `connections` 的值可以是 str / int / float / bool / SecretStr（Engine `_copy_connections` 与 Control `_parse_connections` 均已支持）；插件对 port / database 同时接受 int 和数字字符串。
- `user_id` 通常 < 8 位，按 §13 A3 不进按值替换；它本来就不是秘密。

## 4. 插件包 `testplat-login-plugins`

### 4.1 形态

- 位置：Control 仓 `plugins/testplat_login_plugins/`，独立 `pyproject.toml`，**不加入** Control 的依赖与 `uv.lock`。
- entry point：`[project.entry-points."testplat_engine.login_strategies"] vcode_redis = "testplat_login_plugins.vcode_redis:VcodeRedisStrategy"`。以后的登录方式在同一包加模块、加一行 entry point。
- 依赖：`testplat-engine>=0.4.0,<0.5`；extras `redis = ["redis==<精确版本>"]`（官方 redis-py，用 `redis.asyncio`）。核心包无运行时依赖；`vcode_redis` 模块在 import redis 失败时，`validate()` 抛出“需安装 testplat-login-plugins[redis]”（消息不含配置值）。
- 测试：插件包自带 `tests/`，用 fake redis 客户端（注入点见 §4.2）+ httpx `MockTransport`；另有一组只在本机有 `redis-server` 时运行的测试（缺失则 skip 并在输出中注明）。

### 4.2 行为

`validate(auth)`（预检，不连 Redis、不发请求）：
- 必填：`redis_connection`、`redis_key_prefix`、`create_vcode.path`、`login.path`、`extract.token`；
- 枚举：`create_vcode.body_type ∈ {json, form, query}`、`login.body_type ∈ {json, form}`；
- 未知参数键（不在已知键、且不以 `field.` / `extract.` 开头）→ `ValueError`；
- 所有错误消息只含参数键名，不含值。

`login(ctx)`：
1. 计算剩余预算：起点记 `time.monotonic()`，每一步的 HTTP / Redis 超时 = `ctx.timeout_seconds` 减已用时间（< 0.1 秒直接抛 `TimeoutError`）。
2. createVCode：`ctx.http.request(method, host_url + path, ...)`；状态码 ≥ 400 → `VcodeCreateFailed`；按 `uuid_path`（简化 JSONPath，只支持 `$.a.b`）取 uuid，缺失 → `VcodeUuidMissing`。
3. Redis：用 `connections[redis_connection]` 构造 `redis.asyncio.Redis`（host / port / db / password；`socket_timeout` 与 `socket_connect_timeout` 取剩余预算），只调用一次 `GET {prefix}{uuid}`，`finally` 关闭连接。值为空 → `VcodeCodeMissing`。redis-py 异常转换：`redis.TimeoutError` → 内置 `TimeoutError`，`redis.ConnectionError` / `redis.AuthenticationError` → 内置 `ConnectionError`，转换时不带原消息（`raise ConnectionError("redis") from None`）。
4. 登录：请求体 = 全部 `field.*` + `{code_field: 验证码, user_key_field: uuid}`；状态码 ≥ 400 → `VcodeLoginFailed`。
5. 提取：每个 `extract.<名>` 按路径取值转字符串，`token` 缺失 → `VcodeTokenMissing`；其它缺失不报错（由 Engine 的 inject 检查决定是否 `extract_missing`）。
6. 返回 `LoginResult(values={...extract 结果, "vcode_uuid": uuid})`。uuid 作为命名值返回，让 Engine 把它加入执行级替换集合；`expires_at=None`（有效期走配置 `validity`）。

自定义异常全部继承 `Exception`（不继承内置 `ConnectionError` / `TimeoutError`），公开面只出现类型名，例如 `strategy_error`（`VcodeCodeMissing`）。

### 4.3 记录与脱敏（插件自己负责的部分）

- 每步调用 `ctx.record(request, response, name=...)`，name 依次为 `create_vcode`、`redis_get`、`login`。
- 插件构造的记录里：验证码、uuid、`field.password` 的值一律写成 `"***"`；Redis 一步只记 `{"connection": 名, "key_prefix": 前缀, "found": true/false}`，不记 key 全文与值。
- 请求 / 响应形状与 Engine 的公开快照一致（method / url / headers / body / body_type；status / headers / body），Engine 会再做一遍按键 / 按值脱敏。
- 日志：只记策略名、身份名、步骤名、异常类型名；不记 Redis key、uuid、host。

### 4.4 插件包的安全约束测试

- AST 测试：`reveal()` 只允许出现在一个模块（如 `_secrets.py`），其它模块出现即失败。
- 只读测试：fake redis 客户端记录调用的方法名，断言只出现 `get`（与 `aclose`）；本地 `redis-server` 测试用 `MONITOR` 或 `INFO commandstats` 断言没有写命令。
- 记录测试：createVCode 响应、登录请求体里的验证码（4–6 位）与 uuid 在 `logins` 中都是 `***`。

## 5. Control 改动（重审 §14 B2）

- `_parse_connections`：某个连接的秘密不可用 → 不再整次 422；该连接不进入 `AuthTable.connections`，并记下“不可用连接名”集合。
- 身份判定：login 身份的 `params` 中任一**字符串值**等于不可用连接名 → 该身份变为 `UnavailableAuth`（原因只含连接名）。其它身份照常。
- 引用了**不存在**的连接（拼写错误）不在 Control 判断（Control 不理解插件参数）；由插件 `login()` 在找不到连接时抛 `VcodeConnectionMissing`（`strategy_error`）。在 Control 文档写明。
- 日志与 422 detail 不出现连接字段的值；沿用 3.3a 变量不可用的文案格式。
- 不改 OpenAPI、不改前端。

## 6. 安全验收矩阵（新增，在 §6 第 11–21 条之上）

22. `vcode_redis` 成功：`logins` 依次有 `create_vcode` / `redis_get` / `login` 三步；inject 的 `X-TOKEN` / `X-USER-ID` 生效。
23. 验证码、uuid、登录密码、token、Redis 密码不出现在任何公开面（API 响应、SQLite 原始行、后端日志、DOM）。
24. 业务响应回显 token 或 uuid → `***`。
25. 失败映射：Redis 不可达 → `transport`；Redis 超时（或预算耗尽）→ `timeout`；key 不存在 → `strategy_error`（`VcodeCodeMissing`）；createVCode 非 2xx → `strategy_error`（`VcodeCreateFailed`）；登录非 2xx → `strategy_error`（`VcodeLoginFailed`）；缺 token → `strategy_error`（`VcodeTokenMissing`）。全部 200 + 保存，后续 `skipped(login_failed)`。
26. 只读：整个矩阵期间 Redis 上没有任何写命令。
27. 连接秘密缺失：引用该连接的身份 → 身份不可用（沿用 3.3a 的 422 语义）；同项目其它身份正常执行。
28. 白名单未列 `vcode_redis` 但已安装 → 不注册，使用它的身份 → 422 且无出站请求（E1 回归）。
29. 泄密搜索加入动态值：目标服务签发的 `tok-*` / `sess-*`、本次生成的 uuid 与验证码（§15 I3 收尾）。
30. 3.3a / 3.3b 矩阵与 Phase 3 回归不变。

## 7. 推迟项：鉴权过期自动重登（留档）

- 旧项目做法（`testplat-new/src/testplat/engine/executor.py` 约 1312 行）：业务请求**第一次**返回 401 → 作废该身份缓存的 token → 重新登录 → 同一请求重发一次；第二次仍 401 → 按断言失败。没有方法区分、没有次数上限之外的规则。
- 旧项目需要它，是因为 token 跨执行缓存到文件（默认 12 小时），拿到的经常是旧 token。新平台每次执行都重新登录，被动过期只在长执行或服务端踢下线时出现。
- 结论：与“跨执行缓存”同一批做，届时按旧项目规则实现（401 一次、重登、重发一次），再决定是否需要更细的过期判定。

## 8. 删除已合入分支（附在 Agent 1 任务里执行）

```bash
cd /Users/jiavin/Developer/products/testplat1/testplat-engine
git branch -d codex/phase3-3b-engine-login codex/phase3-3b-integration-engine
cd ../testplat-control
git branch -d codex/phase3-3b-control-login-backend codex/phase3-3b-control-login-frontend codex/phase3-3b-integration-control
```

远程分支（Engine `origin/codex/phase3-3b-engine-login`、Control `origin/codex/phase3-3b-integration-control`）删除属对外操作：**用户已同意删除**，由 Agent 1 执行 `git push origin --delete <分支>` 后 `git fetch --prune`。

## 9. 决策记录

| # | 裁决 |
|---|---|
| C1 | 插件放 Control 仓 `plugins/`，统一包 `testplat-login-plugins`，按依赖分 extras；不按插件分仓 |
| C2 | 本批不改 Engine：uuid 经 `LoginResult.values` 入替换集合；验证码等短值由插件在 `ctx.record` 前自行遮蔽 |
| C3 | 连接秘密不可用 → 只阻断 params 中引用该连接名的身份 |
| C4 | 鉴权过期自动重登、API 重发推迟到跨执行缓存批次（§7） |
| C5 | 分工：主线程做设计 / 审查 / 只读验证；执行 Agent 写代码与提交；推送、删远程分支、PR、tag、合并需用户授权 |
