# Phase 4.1 设计稿：Web 配置与 GSB 冒烟闭环

日期：2026-10-15。状态：**v1 设计稿；产品方向已确认，以下技术方案尚未实现**。

需求基线：[phase4-roadmap.md](phase4-roadmap.md)。术语：[CONTEXT.md](../CONTEXT.md)。4.0 的 Git 同步与 commit 快照继续使用。本轮先写设计和验收条件，不修改运行代码、不删除数据库。

## 1. 目标与交付顺序

测试人员能在 Web 中维护某项目的环境、地址和账号，选环境运行已有 GSB 冒烟，查看每步结果；密码、地址或商品关键字改了，保存后直接重跑。

| 小批次 | 可验收结果 | Engine 影响 |
|---|---|---|
| 4.1a | 注册 / 登录 → 项目环境和账号管理 → 用数据库配置运行现有 GSB API / smoke → 修改后重跑 → 记录操作人及配置版本 | 复用已有 environment / AuthTable；不增加步骤 DSL |
| 4.1b | case 步骤引用账号名、单接口选择账号；同一接口先后用两个账号运行且登录态不串用 | 增量契约，先冻结 schema / DTO 和兼容条件，再实施 |

4.1a 先沿用当前 GSB 资产的 `auth: gsb`，在 Web 建立账号名为 `gsb` 的账号；它引用 Git 的 GSB 登录流程。账号管理可保存其他账号，但“同一接口按步骤切换账号”在 4.1b 验收，不能把只建了多条账号记录称为已经支持切换。

平台登录只用于识别操作人；所有登录用户具有同等操作能力。分享链接、结果脱敏全局开关、排队、Web 编辑 case、flow / 数据驱动 / UI 不混入 4.1a。现有执行结果脱敏保持当前协议，测试账号编辑页提供查看凭据能力。

## 2. 旧项目参考与新项目现状

参考根目录：`/Users/jiavin/Developer/products/testplat-new`，只读。

| 参考资产 | 已核实的业务含义 | 当前采用方式 |
|---|---|---|
| `assets/api/gsb/b2/account/balance.yaml` | POST `/api/account/balance`，无业务入参，成功条件为 `$.errorCode == "0"` | 沿用新资产 `api.gsb.account.balance` |
| `assets/api/gsb/b2/order/search_product.yaml` | GET 搜商品，业务参数 product_keyword 映射为 keywords | 沿用新资产 `api.gsb.order.search_product`；遵循新 Engine query 格式 |
| `assets/scenarios/go2/baseline/readonly_smoke.case.yaml` | 同一账号下查余额、搜商品，原场景另有业务结果断言 | 当前 smoke 保持两步；不能宣称已经有旧版全部断言 |
| `assets/scenarios/shared/take_return/common/take_assigned.flow.yaml` | 以订单号查询代发拿货行，回 GSB 查订单，再切管理账号完成指派；有数据提取、轮询与关联校验 | 用于核对项目、多系统、账号切换、数据依赖的模型；后续逐段实现 |
| `assets/scenarios/e3e3/order/full_pay/resolve_item.flow.yaml` | 根据完整货号和颜色 / 尺码选择 SKU，不能把任意第一项当成业务目标 | 将来做下单场景时作为选择与断言依据，当前不扩写完整下单链路 |

旧项目 DSL 的“简单 case 直接调用 api，重复时才抽 flow”“谁操作显式，登录机制自动完成”值得保留。旧版 `as/default_as/use/with/->` 和身份坐标不作为新 schema；用户已确定按项目、环境、账号名查找。

本轮核对的新代码：

- Control `services/env_loader.py` 仍从 Git 环境文件读配置，`services/project_access.py:require_env` 仍根据同步生成的 `Project.envs` 验证。
- Control `services/auth_loader.py` 从 Git auth 文件和传入的环境变量映射构造 AuthTable；应复用其严格校验和 SecretStr 包装，改变配置来源。
- Engine `domain/case.py` 的 schema 仍为 0.2，严格拒绝 `account` 等未知步骤字段。
- Engine 已有 `output` alias 和 `${alias.body.field}` 基础传值；参考 `notes/phase3-case-contract-v0.2.md`。这不是需要从零开发的能力。
- Engine `engine/login_cache.py` 的 key 包含 cache_scope、identity、材料指纹；指纹涵盖登录地址、流程及秘密等。应验证更新后的材料自然不命中旧缓存，不额外建立一套 Control token 缓存。

## 3. 业务模型与数据归属

```text
资产仓库
└── 业务项目（例如 main/gsb；也可为跨系统业务项目）
    ├── Git：api、case、登录流程
    └── Web：项目环境 test / pre
        ├── 服务地址：gsb、unified_login，后续可有 daifa_admin 等
        ├── 配置：超时、业务参数默认值
        └── 测试账号：gsb、merchant_a、dispatcher …
            ├── 展示名 / 业务角色
            ├── 所用 Git 登录流程
            └── 用户名、密码等凭据
```

账号唯一键是 `(project_id, env, account_name)`，其中 project_id 包含仓库键。不增加全局被测系统表、全局角色表或跨项目账号池。业务角色先是账号的描述字段，同一角色可有多个账号。

账号名与实际登录用户名分开：改密码或登录用户名不改 case 引用。首版账号名沿用 Engine identity 规则 `^[a-z][a-z0-9_]*$`，`none` 保留；中文用于展示名和角色描述。环境名沿用现有路径安全规则，保存后标识不可重命名；先支持新增、编辑、停用，避免重命名破坏引用。标识命名约束是本稿的技术方案。

### 3.1 数据表建议

| 实体 | 主要字段与约束 |
|---|---|
| PlatformUser | id、唯一 username、password_hash、created_at；无权限角色字段 |
| PlatformSession | session token 的哈希、user_id、expires_at；注销后失效 |
| ProjectEnvironment | 唯一 `(project_id, env)`；enabled、api_hosts、timeout_seconds、case_params、revision、updated_by、updated_at |
| TestAccount | 唯一 `(project_id, env, account_name)`；display_name、business_role、login_profile、encrypted_credentials、enabled、revision、updated_by、updated_at |
| ConfigChange | 操作人、项目 / 环境、对象名、动作、字段名、时间与 revision；凭据变更只记“已修改” |
| ExecutionRecord 增量 | 复用 operator，增加 config_revision、account_revisions 和非秘密配置快照；保留 env、repo、commit_sha |

账号和环境修改在同一事务中递增环境的 revision；执行记录用该版本和实际账号版本追溯。Project 的 envs / default_env 如保留为响应字段，应由 Web 环境查询派生，不再由每次 Git 同步覆盖。

## 4. Git 登录流程与数据库凭据的结合

### 4.1 与环境解耦的流程文件（建议格式）

新增项目内 `config/auth/_profiles.yaml`，沿用现有 `identities.<name>.login` 内容形状；其中名字作为可复用登录流程名，不再等于某个具体账号。其模板不含环境地址或真实凭据。

```yaml
identities:
  gsb_login:
    login:
      host: unified_login
      steps:
        - name: login
          method: GET
          path: /site/login
          body:
            type: query
            fields:
              username: "${secret:username}"
              password: "${secret:password}"
              source: go2
          extract:
            - {source: body_json, path: "$.data.hash", as: token}
            - {source: body_json, path: "$.data.userId", as: user_id}
      validity:
        ttl_seconds: 14400
      inject:
        headers:
          X-TOKEN: "${login:token}"
          X-USER-ID: "${login:user_id}"
```

`source: go2` 沿用现有 GSB 登录请求，不从示例项目键 gsb 自动推导；将来不同业务项目有不同 source 时，在相应 Git 登录流程定义。`host: unified_login` 的实际 URL 来自所选项目环境。

Web 的 gsb 账号选择 `gsb_login`，凭据字典填写 username / password。Control 对每个账号单独把模板中的 `${secret:字段名}` 从该账号凭据字典解析，再构造以**账号名**为 identity 的 AuthTable。多个账号可引用同一流程；禁止把各账号凭据混成一个全局字典或临时写入 os.environ。

此文件名和解析入口是本稿建议，现有 auth_loader 尚不能直接加载它；实施时提取纯数据构造入口并复用现有校验。不能把以上示例直接当作当前运行配置。

4.1a 的真实资产同步调整：从现有 `config/auth/test.yaml` 提取 GSB 流程到 `_profiles.yaml`、凭据引用改成字段名，真实凭据由 Web 录入；API 的 `auth: gsb` 保留。旧 auth_loader 对历史格式的直接调用与测试夹具保留兼容，平台管理的账号则不回退读取服务器 TESTPLAT_GSB_*。

### 4.2 环境文件与导入

- 运行时环境列表和取值以数据库为准；新建 pre 可复用 gsb_login，无须在 Git 新建 pre.yaml。
- Git `config/envs/*.yaml` 可继续作为示例 / 初始化模板。同步资产不会覆盖 Web 的环境、账号或启停状态。
- 首次同步是否自动导入尚未单独确认；4.1a 以 Web 手动创建环境完成验收，自动导入不阻塞开发。若之后采用模板导入，应显式记录初始化状态，并保证重复同步幂等、删除 / 停用后不偷偷重建。
- 不从旧项目 ignored secrets 或进程环境变量自动搜集账号密码入库。

## 5. 配置解析 Module

Control 提供一个运行配置解析 Module。Interface 接收项目、环境和已经固定的资产 commit，返回 Engine environment、AuthTable、有效超时 / 参数及可持久化的非秘密配置摘要。调用者不需要知道账号加密和模板文件布局。

Implementation 内部读取环境与账号、解密凭据、解析 Git 流程并复用现有 auth_loader 构造逻辑；Engine 继续负责登录、缓存、HTTP、case 预检与执行。Engine 不读取数据库、平台会话或 Git 远程，Control 不解析 / 重写 case 语义。

### 5.1 生效顺序

1. 校验平台会话、项目状态、所选数据库环境及 enabled 状态；固定本次 commit。
2. 在一致的数据库读取视图中取得环境、账号和版本，构造本次运行材料；事务结束后才调用网络。
3. 参数采用浅层顶级键覆盖：运行请求 params > 环境 case_params > case 资产 params。环境参数必须是非秘密业务默认值；缺参规则继续由 Engine 预检裁决。
4. HTTP 步骤超时采用：请求显式 timeout_seconds > 环境 timeout_seconds > 平台默认 30 秒。HTTP schema 需区分“省略”与“显式 30”，避免旧请求模型默认值吞掉环境配置。仍是每步超时，不声称是整个 case 总预算。
5. api_hosts 来自数据库环境，不提供请求体任意覆盖 host 的额外通道。
6. 通过 EnginePort 执行；保留既有业务失败、preflight 错误和 execution 系统错误的 HTTP / 记录语义。
7. 记录 commit、env、操作人、config_revision、实际账号名 / 版本和有效非秘密配置；历史详情不再读取当前环境来补全。

4.1a 对当前使用的账号检查模板与凭据。未使用的、未配置完整的账号不应阻止 GSB 冒烟；复用 UnavailableAuth 思路，仅当被引用时拒绝。4.1b 的显式账号预检由 Engine 决定，不在 Control 另写 case 解析器。

### 5.2 修改与缓存

- 一次执行全程使用启动时冻结的配置；编辑影响之后开始的运行。跨步骤不重新读数据库。
- cache_scope 沿用当前实现的 `repo:project:env` 编码；Engine 禁止 scope 含 `/`，不能直接照早期文档传 `repo/project:env`。身份为账号名，材料指纹进一步区分凭据 / 登录地址 / 流程变化。
- 改用户名、密码、流程或登录地址后，新执行必须按新材料登录；可以让旧缓存自然过期，但不能因身份名相同而命中旧材料。
- 账号停用后，新执行预检拒绝；不能因旧 token 仍缓存而绕过。已开始的执行不强制中断。
- 配置更新采用 revision 条件更新，旧版本提交返回 409，提示刷新后再保存，避免两人静默覆盖。

## 6. 最小 Web 页面与 HTTP 接口建议

页面：注册 / 登录；项目详情增加环境管理和测试账号管理入口。环境表单支持新增、编辑、停用，填写逻辑服务名和地址、超时、参数默认值；账号表单支持账号名、展示名、业务角色、Git 登录流程及凭据字段。密码可点击显示 / 编辑，不要求超管权限。首版不用 Monaco、配置继承树、拖拽编辑器或单独测试登录按钮。

运行页环境下拉来自该项目启用的数据库环境。没有环境时提示去配置；账号缺失时指出账号名与所在环境。4.1a 继续通过现有资产 auth 选择账号，4.1b 再增加单接口账号下拉。

以下是设计接口，尚未存在；项目路由前缀为 `/repos/{repo}/projects/{project}`：

| 接口 | 行为 |
|---|---|
| `POST /auth/register`、`POST /auth/login`、`POST /auth/logout`、`GET /auth/me` | 注册、登录、注销、当前操作人 |
| `GET/POST .../environments` | 项目环境列表 / 新增 |
| `GET/PUT .../environments/{env}` | 查看 / 更新环境及 enabled；PUT 携带 revision |
| `GET .../login-profiles` | 当前资产 commit 下可选择的流程及所需凭据字段名；不返回凭据 |
| `GET/POST .../environments/{env}/accounts` | 账号列表 / 新增；列表不批量返回密码 |
| `GET/PUT .../environments/{env}/accounts/{name}` | 查看 / 编辑账号；PUT 携带 revision |
| `GET .../environments/{env}/accounts/{name}/credentials` | 当前登录用户显式查看该账号的凭据，响应禁止缓存 |
| 现有 api / case run、执行记录接口 | 使用数据库配置，补操作人及版本摘要 |

会话建议采用同源 HttpOnly、SameSite=Lax cookie，随机 token 仅存其哈希，有效期 7 天；部署 HTTPS 时开启 Secure。变更接口校验同源 Origin，不在 URL 中携带会话。没有会话返回 401，不为 health / 注册 / 登录等入口设置自我依赖。会话是平台身份，不进入 Engine 的 auth 表。

平台注册密码使用成熟密码哈希库保存；被测系统凭据需要解密发给登录流程，因此使用现成认证加密库加密保存，主密钥由 TESTPLAT_MASTER_KEY 提供。密钥缺失 / 错误应明确报配置错误，不自动生成新密钥使旧数据失效。详情与凭据接口、错误日志、修改记录和执行配置摘要分别建模，避免序列化时把完整凭据字典带到不需要的地方。

## 7. 4.1b 账号选择契约方向

产品语义已确认；下列 YAML 为**后续契约提案**，当前 schema 0.2 严格拒绝 account，发布前必须明确新 schema 版本、请求 DTO 与兼容规则。

```yaml
steps:
  - step_id: merchant_balance
    type: api
    api_id: api.gsb.account.balance
    account: merchant_a
  - step_id: another_balance
    type: api
    api_id: api.gsb.account.balance
    account: merchant_b
```

- 账号解析固定为当前项目 + 所选环境 + 步骤账号名，不允许在 case 中写用户名、密码或跨项目账号路径。
- 显式 step account 优先于 API 的非匿名 auth，再按旧有默认身份规则处理未显式选择账号的资产；API 的 `auth: none` 必须保持匿名，显式账号与其冲突时拒绝。
- 预检和实际执行都使用同一个账号选择规则；未知 / 停用账号、缺失凭据在发业务请求前发现。
- Engine 必须取得步骤选择的身份并隔离会话，不能只在 Control 改 AuthTable.default_identity，因为当前 API 已显式固定 auth。
- 单接口选择账号应经明确的请求字段传入 Engine；不修改只读 API 文件、不靠把 token 塞进 params。
- 用两个独立账号调用同一个 API 验证身份选择，再做 GSB / 代发跨系统步骤；不把账号选择与完整订单 / 支付 / 退款链路捆在一次交付里。

## 8. 验收矩阵

### 8.1 4.1a 必须通过

| # | 触发与预期 |
|---|---|
| A1 | 注册、登录、刷新页面、注销均可用；未登录请求配置 / 执行返回 401；普通注册用户可以维护项目配置 |
| A2 | 项目 P、Q 都建 test 环境及同名 gsb 账号；各自运行使用各自的地址和凭据，操作人不充当测试账号 |
| A3 | 在 Web 新建 Git 中不存在的 pre 环境；填地址、账号后，可运行现有 GSB API / smoke |
| A4 | GSB 登录通过 unified_login，业务请求通过 gsb；余额 POST 和商品搜索 GET 的参数 / success 与参考接口一致 |
| A5 | 在 Web 改商品关键字、超时、host 后重跑，实际请求采用新配置；显式请求参数可覆盖对应默认值 |
| A6 | 同一账号改密码后重跑，登录请求采用新密码；不命中旧材料的缓存。重复使用未变配置时仍可复用缓存 |
| A7 | 在第一步暂停的测试夹具中编辑环境和账号；正在运行的后续步骤仍使用旧快照，新运行使用新配置 |
| A8 | Git 更新模板环境后点同步，Web 已维护的数据、启停状态不被覆盖；代码 / 登录流程 commit 可以正常更新 |
| A9 | 未引用账号未配置完整不阻断 GSB；引用账号缺失 / 停用则预检失败，不能回退到服务器账号 |
| A10 | 历史记录保留原 commit、env、操作人和配置版本；后续编辑不改历史；配置摘要和修改记录无凭据原值 |
| A11 | 两客户端从同一 revision 保存不同修改，后保存的旧版本返回 409；新执行取得一个一致版本的配置 |
| A12 | 保存后重启 Control，环境 / 账号仍可用；账号凭据为密文，平台用户密码为哈希；日志不含密码 / token 原值 |
| A13 | 现有 Engine 严格 schema、基础跨步骤引用、业务失败 / fail-fast、登录缓存和错误响应契约回归通过 |

自动化验收使用本地可识别账号 / host / 入参的假服务，证明配置实际进入请求。真实 GSB 连通性作为单独人工 / 内网验收；本机无法访问目标时只能证明编排走通，不能记录为真实业务通过。

### 8.2 4.1b 增量验收

同一接口的两个步骤分别选 merchant_a / merchant_b；假服务断言两次身份不同，切回 merchant_a 不串用 merchant_b 的会话。相同账号名在不同项目 / 环境隔离。旧 schema 无 account 的资产结果不变；新字段在旧 Engine 上明确拒绝，不静默忽略。显式匿名接口与账号冲突、账号缺失和停用均按契约拒绝。

## 9. 实施顺序与开发数据

1. 先完成 Control 的平台注册 / 会话、表结构、环境 / 账号持久化和运行配置解析；用本地假服务验证来源和缓存。
2. 接入前端的环境 / 账号表单、环境选择、配置错误提示、操作人 / 版本展示。
3. 在资产仓库提取 GSB 登录流程，填写 Web 配置并完成 4.1a 矩阵；旧项目始终只读。
4. 单独冻结 4.1b Engine 契约，再接步骤账号字段和单接口账号选择；之后才进入 4.2 跨系统样例 / 排队。

当前开发库可由执行人显式删除重建，本稿不要求引入 Alembic；启动检查不兼容表结构时给出说明，不能自动清空。正式团队使用前另定备份与升级策略。提交、推送、PR、合并和部署仍按实际用户授权执行；本稿不代替这些操作的授权。
