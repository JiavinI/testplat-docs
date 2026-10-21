# Phase 3.3d 设计稿：跨执行登录缓存 + 401 重登重发

日期：2026-10-13 · 状态：v1，用户已批准 4 个方向点（见 §1）
基线：Engine `v0.4.0` = `f1a0c875`；Control `main` = `2dad704`（#39）。
前置：`phase3.3b-login-auth-design-draft.md`（D8/D13/E3/Q7/H1 继续有效，本稿只写变化）。

## 1. 已批准方向

| # | 裁决 |
|---|---|
| R1 | 过期时间取值顺序沿用 0.4.0（`login_session.py:355`）：登录响应 `validity.expires_from` → 插件 `LoginResult.expires_at` → 登录成功时刻 + `validity.ttl_seconds`；提前 30 秒判失效（E3）。**本批不改这套规则**，只让它跨执行生效。 |
| R2 | 三者都没有的身份**不跨执行缓存**，每次执行照旧重新登录；不编默认有效期。 |
| R3 | 缓存在 Engine 进程内存，不落盘；token 不离开 Engine。服务重启即清空。 |
| R4 | 缓存键 = Control 传入的作用域（项目 + 环境）+ 身份名 + 登录配置指纹（含秘密的 HMAC 摘要）；改密码 / 改登录配置自动失效。 |
| R5 | `session: cookie` 身份本批**不跨执行缓存**（仍执行内复用）。 |
| R6 | 401 重登重发：只看业务响应 HTTP 状态码 401；默认开启，身份级可关；所有方法都重发；每个请求最多重发一次。body 里的业务码判过期留到下一批。 |
| R7 | Engine `0.5.0`；Control 约束 `>=0.5.0,<0.6`。 |

## 2. Engine：缓存（`LoginCache`）

### 2.1 位置与生命周期

- 新模块 `engine/login_cache.py`。`create_engine()` 构造的 Engine 实例持有**一个** `LoginCache`，传给每次执行新建的 `SessionPool`。
- Control 的 `PackageEngineAdapter` 按插件白名单缓存 Engine 实例，所以白名单不同的项目各自一份缓存；加上作用域隔离，互不影响。
- `create_engine(..., login_cache_max_entries: int = 256)`：`0` = 关闭跨执行缓存（行为等同 0.4.0）。负数 → `ValueError`。
- `LoginCache` 不可 pickle/copy；`repr` 只含条目数。不提供任何列出键或值的公开方法。

### 2.2 作用域

- `AuthTable` 新增可选字段 `cache_scope: str | None = None`。`None` → 该表的所有身份不跨执行缓存（向后兼容：0.4.0 调用方、测试替身不受影响）。
- 校验：非空字符串，≤ 200 字符，只允许 `[A-Za-z0-9._:-]`；它不是秘密，但也不进公开结果与 `to_dict()`。
- Control 传 `f"{project.id}:{project.env}"`。

### 2.3 键与指纹

键 = `(cache_scope, identity, fingerprint)`。

- `fingerprint = HMAC-SHA256(进程级随机密钥, 规范化登录材料)`，十六进制。密钥在 `LoginCache` 构造时 `secrets.token_bytes(32)` 生成，只在内存，不可读出。
- 规范化材料（JSON，`sort_keys`，分隔符固定）包含：`strategy`、`tier`、**解析后的登录 host_url**、`flow` 结构（步骤、方法、路径、请求体模板、提取规则、成功条件）、`params`、`inject_headers`、`validity`、`probe`、`session_cookie`、`reauth_on_401`，以及**本身份可能用到的连接**（插件档位：`AuthTable.connections` 全部；其它档位：不含）。
- 其中 `SecretStr` 在 HMAC 输入里以 reveal 后的值参与（这样改密码才会失效）。`reveal()` 只允许出现在 `login_cache.py` 的一个函数里，并加入 Engine 现有的 AST 白名单测试。
- 指纹与键不进日志、不进公开结果、不进异常消息；日志只记 `identity` 与 `cache=hit|miss|store|invalidate|evict`。

### 2.4 写入条件（全部满足才写）

1. `cache_scope` 非空，且 `login_cache_max_entries > 0`；
2. 身份 `session_cookie=False`（R5）；
3. 登录成功（含 probe 成功）；
4. 按 R1 算出了 `expires_at`，且 `expires_at - now > 30`（否则写了也立刻失效）。

写入内容：命名值 `values`（dict 副本）、`expires_at`、单调递增的 `generation`。**不缓存** HTTP client、Cookie jar、登录过程记录。

### 2.5 读取（`SessionPool.ensure`）

顺序：
1. 本次执行池内已有且有效 → 复用（0.4.0 行为，`session_source="execution"`）。
2. 池内没有 → 查缓存。命中且未过期（`now < expires_at - 30`）→
   - **先**把 `static_login_secrets(material, connections)` 与缓存的全部命名值加入执行级替换集合（H1，缺这一步会泄露）；
   - 用 `_check_values` 再核一次（防御：inject 引用必须仍被覆盖），不过 → 当作未命中并删除该条目；
   - 用 client factory 新建本次执行专用 client（池负责关闭），建 `SessionEntry`，`login_ref=None`，记 `session_source="cache"`；
   - **不**追加 `logins` 记录。
3. 未命中 / 过期 → 正常登录（`session_source="login"`），成功后按 §2.4 写入。过期条目在读到时删除。

### 2.6 并发

- 同一键的“查缓存 → 登录 → 写缓存”用**每键一把 `asyncio.Lock`**（single-flight）：两个并发执行冷启动时只登录一次，第二个等锁后直接命中。
- 锁表随条目淘汰一起清理；锁内只做本键的登录，不跨键持锁。
- 某些系统“新登录会踢掉旧 token”，single-flight 同时减少了这类互踢；剩余情况由 §3 的 401 重发兜底。

### 2.7 淘汰与作废

- 容量满 → 淘汰最久未使用（LRU）。
- 登录失败 → 删除该键条目。
- 401 作废（§3）：只在条目的 `generation` 等于本次请求所用的那一代时删除，避免把别的执行刚写入的新 token 删掉。

## 3. Engine：401 重登重发

### 3.1 配置

- `LoginAuth` 新增 `reauth_on_401: bool = True`（构造校验为布尔）。静态 header 身份不涉及。
- Cookie 身份同样适用（重登发生在执行内，与 R5 不冲突）。

### 3.2 流程（`_execute_api`，在业务响应返回后、断言之前）

条件：本步使用 login 身份，`reauth_on_401=True`，响应 `status_code == 401`，本请求尚未重发过。满足时：

1. 记日志 `reauth identity=… trigger_status=401`（不记 URL 之外的任何东西，也不记 URL）。
2. 作废：从本次执行池删除该身份条目（关闭其专用 client）；若条目来自缓存或已写入缓存，按 generation 作废缓存条目。
3. 强制重新登录（走正常 `_login`，追加一条 `logins` 记录，`login_ref` 指向它）。失败 → 返回 `login_failed` 业务结果（与 D9 相同：200 + 保存），`auth.reauthenticated=true`。
4. 用新命名值重新渲染 inject、重建 `send_headers`；请求体、方法、URL 与第一次完全相同；刷新替换集合快照后**重发一次**。
5. 公开 `request` / `response` 只是**第二次**的；第一次 401 响应不进公开结果（只在日志记状态码）。
6. 第二次仍是 401 或其它结果 → 按正常断言流程判定，**不再重发**。

### 3.3 Case 路径

- 重登后池内条目已更新，后续步骤直接用新会话。
- 每个步骤最多一次重发；整次执行不设总次数上限（每步至多多一次登录）。

### 3.4 超时

- 每次请求各用一次 `timeout_seconds`；重发会让单步耗时最多变为“2 次请求 + 1 次登录”。文档注明，不另设预算。

## 4. 公开契约变化（Engine 0.5.0）

### 4.1 `auth` 由 7 键变 9 键

新增：

| 键 | 取值 | 说明 |
|---|---|---|
| `session_source` | `"login"` / `"execution"` / `"cache"` / `null` | 本步使用的会话来源：本步刚登录 / 本次执行前面步骤登录的 / 跨执行缓存；静态 header 身份为 `null` |
| `reauthenticated` | `bool` | 本步是否触发了 401 重登重发；静态 header 身份恒为 `false` |

- `session_source="cache"` 时 `login_ref=null`（本次执行没有登录记录）。
- 重发后 `session_source="login"`，`login_ref` 指向重登那条记录。
- `_public_auth` 校验同步更新：恰好 9 键；静态身份 `session_source is None and reauthenticated is False`。
- 不公开 `expires_at`、缓存键、指纹、generation（C2）。

### 4.2 其它

- `logins`：缓存命中不追加；401 重登追加一条，结构与 0.4.0 相同。
- `login_failed` 的 reason 枚举不变。
- 版本 `0.5.0`；CHANGELOG 写明 `auth` 新增两键为**破坏性**（严格键校验的调用方需升级）。

## 5. Control

1. 依赖：`testplat-engine>=0.5.0,<0.6`。开发期临时 `file://` 锁 Agent 1 已审查 commit，合并前改远程精确锁定（同 3.3b 流程）。
2. `auth_loader`：
   - `AuthTable(cache_scope=f"{project.id}:{project.env}")`。`load_auth` 目前只收 `config_root, env`，新增关键字参数 `cache_scope: str | None = None`，由 `execution_service._load_auth` 传入。
   - login 身份新增可选键 `reauth_on_401: bool`（加入 `_LOGIN_KEYS`），非布尔 → 422 schema 错误。
3. 不改数据库表；执行记录里的 `auth` JSON 自然带上两个新键。旧记录缺这两个键，读取侧按 `null` / `false` 兼容。
4. OpenAPI：`auth` schema 加两个字段，重新生成 `docs/phase3_3/openapi-*.json` 与 diff。
5. 前端（小改）：
   - `types.ts` 加两字段（可选，兼容旧记录）。
   - `AuthSummary`：`session_source="cache"` 显示“复用登录（未重新登录）”；`reauthenticated=true` 显示“鉴权失效，已重新登录并重发”。
   - `LoginProcessPanel` / 执行历史：`login_ref=null` 且 `session_source="cache"` 时不显示“查看登录过程”，改为一行说明；不得报错。
6. 文档：`docs/phase3_3/control-auth-backend.md` 增加“跨执行缓存与 401 重发”一节：条件（§2.4）、不缓存的情况（R2/R5）、如何关闭重发、服务重启清空、单步最长耗时（§3.4）。

## 6. 安全验收矩阵（续 3.3c 第 22–30 条）

31. 缓存命中：同一项目同一身份连续两次执行，目标服务只收到 1 次登录；第二次 `session_source="cache"`、`login_ref=null`、`logins=[]`，inject 生效。
32. **缓存命中路径的按值替换**：第二次执行里业务响应回显 token → `***`（H1，最关键的一条）。
33. 过期：`ttl_seconds` 设为 32，等 3 秒后第二次执行重新登录；`expires_from` 同样验一次。
34. R2：无 ttl、无 expires_from → 每次执行都登录。
35. R5：`session: cookie` 身份 → 每次执行都登录。
36. 指纹：修改登录密码对应的环境变量后再执行 → 重新登录（不得复用旧 token）；修改登录 path 或 host 同样重新登录。
37. 作用域隔离：两个项目同身份名同配置 → 各登录一次，互不命中。
38. 401 重发成功：目标服务吊销当前 token → 业务首次 401 → 重登 → 重发 200；`reauthenticated=true`，`logins` 多一条，公开 `response` 是 200 那次；目标服务收到的两次业务请求方法、路径、body 完全相同（POST 验一次）。
39. 401 两次：重发后仍 401 → 断言失败；目标服务恰好收到 2 次业务请求、1 次重登。
40. `reauth_on_401: false` → 不重发，401 直接走断言。
41. 重登失败 → `login_failed`（200 + 保存），`reauthenticated=true`。
42. 缓存作废：38 之后再执行一次 → 命中的是重登后的新 token（目标服务不再收到登录）。
43. 并发：两个执行同时冷启动 → 目标服务只收到 1 次登录（single-flight）。
44. 泄密搜索：所有签发过的 token（含被吊销的旧 token）都在动态目标里，意外命中为 0；后端日志里缓存相关行只含 `cache=hit|miss|store|invalidate|evict` 与身份名。
45. 3.3a / 3.3b / 3.3c 矩阵与 Phase 3 回归不变。

## 7. 主线程裁决（本稿内定，不再询问）

| # | 裁决 | 理由 |
|---|---|---|
| C1 | 作用域由 Control 显式传 `cache_scope`，`None` 不缓存 | Engine 不认识“项目”；默认安全 |
| C2 | 不公开 `expires_at` | 契约少长一个字段；需要时下一批再加 |
| C3 | 第一次 401 响应不进公开结果，只记日志状态码 | 公开面只描述最终那次请求，和旧项目一致；避免重复脱敏面 |
| C4 | 指纹用进程级随机密钥 HMAC，而不是普通哈希 | 普通哈希可被离线比对猜密码；进程重启缓存本来就清空 |
| C5 | 缓存不存 client / Cookie jar / 登录记录 | 只存命名值，泄露面最小，也与 R5 一致 |
| C6 | `login_cache_max_entries` 默认 256，`0` 关闭 | 内存有界；给测试和排障一个开关 |
| C7 | single-flight 用每键 `asyncio.Lock` | Control 是单进程 asyncio；多进程部署时各进程各一份缓存，文档注明 |

## 8. 不在本批

- body 业务码判过期（`code=401` 但 HTTP 200）。
- Cookie 身份跨执行缓存。
- 缓存落盘、多进程共享、手动“清除登录缓存”按钮。
- 多请求体格式（multipart）。

## 9. 3.3c 遗留小修（附在集成 Agent）

1. L24 写死 uuid 的断言改为用本次签发的 uuid。
2. L23 `"tok-" not in blob or "***" in blob` 改为对每个动态 token 精确断言不出现。
3. 泄密搜索短验证码：文档描述改为“引号包裹匹配”，并补 `=值` / `:值` 形态（`code=1234`）的匹配。
