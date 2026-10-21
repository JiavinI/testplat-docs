# Phase 4.1a 技术债与旧接口盘点

日期：2026-10-15。范围：Control PR #42 最新提交 `3a0557c2a3a365202f6d339948862727e4d764de`，及其现存前端、脚本、测试和文档调用方。未将未实施的 Agent 2/3/4 工作当作 Agent 1 缺陷；不是对整个 Engine 或全部旧业务资产的完整审计。

结论：存在未收尾的技术债与过时接口说明。建议在当前修复中增加有限的收尾批次，并给后续 Agent 明确迁移任务和退出条件。业务源码未修改。

## 先处理的功能缺陷

[最新复审记录](phase4.1a-agent1-rereview.md) 的四项仍有效：账号创建丢环境 revision、参数和记录版本不一致、共享环境默认参数阻断其他接口、跨进程测试硬编码工作目录。它们是当前实现缺陷，应在放行前修复，不能只登记为以后处理的技术债。

## 已确认的技术债

### D1 · P2 · 网络执行仍占用数据库事务

证据：`services/execution_service.py:266–292` 解析运行配置后直接 `await engine.execute`；`database/session.py:60–65` 仅在请求结束时关闭 Session。独立 HTTP 验证在 FakeEngine.execute 入口观察 `Session.in_transaction() == True`，运行返回 200。

影响：网络等待期间保留数据库事务 / 连接；后续并发、连接池容量和写记录行为更难控制。此处仅确认事务仍活跃，未声称已复现 SQLite 全库锁死。设计稿 §5.1 明确要求结束配置读取事务后再发网络。

处理：Agent 1 在冻结配置后结束读取事务，捕获执行记录需要的项目标识等值，避免 ORM 过期后再次隐式读库；结果写入单独事务。配置读取还需保证一致视图，不能只调用 commit 就声称已解决全部一致性问题。

退出条件：API / Case 在 Engine 入口均不持有配置读取事务；运行中编辑配置不改变本次材料；执行结果仍正常落库。

### D2 · P2 · 旧验收脚本没有迁移平台登录

证据：`scripts/phase4_0_matrix.py:69` 创建 TestClient 后直接同步 / 查询，无注册登录；`scripts/phase4_0_e2e_real_assets.py:98` 同样如此。auth / login / vcode / cache 矩阵的 httpx.Client 也未接入平台登录。

实跑命令：`PHASE4_MATRIX_OUT=<临时目录> PHASE4_SKIP_SCREENSHOTS=1 uv run python scripts/phase4_0_matrix.py`，结果 **2 passed、7 failed、1 skipped**；六项直接提示“未登录”，另一个条目因为响应已变成鉴权错误而缺少 `ok` 字段。

`tests/conftest.py` 默认注入平台用户只在 pytest 中生效，不能证明独立脚本仍可用。README 仍把旧矩阵作为可运行入口介绍。

处理：Agent 4 迁移仍作为门禁使用的脚本，通过正常注册 / 登录取得 cookie；并发新建客户端时显式继承会话。Agent 3 的浏览器流程接登录、401 引导。若某脚本只保存为历史证据，应从当前运行指南移除并标记适用版本。

退出条件：明确当前门禁清单，每个入口要么可运行，要么标明历史用途；不能只检查脚本文本已使用新 URL。该项应在 4.1a 整体交付前关闭。

### D3 · P2 · 当前文档混杂旧契约，且本轮文档自相矛盾

证据：

- `README.md:5–29` 写 Engine `>=0.2.0,<0.3`、锁定 `4715c61`；实际 pyproject 是 `>=0.5.0,<0.6`、锁定 `5231fb4`。
- `README.md:118–121` 仍给出 `/projects/{id}/cases/...` 和旧记录 URL；当前执行 API 是 `/repos/{repo}/projects/{project}/...`，旧执行路径已经移除。
- `docs/phase4_1a/control-backend.md:68` 仍称环境默认值不能补必填；同文档 `:92` 又称可以补齐。
- `api/case_schemas.py:211` 仍声明旧通用记录 URL 可用；该文本会进入最新 OpenAPI 的 schema 描述。前端 `features/executions/types.ts:168` 也残留旧 URL 注释。

处理：Agent 1 更新当前 README、schema 描述和 4.1a 文档，并重新生成 OpenAPI；Agent 3 更新相关前端契约注释。历史版本文档 / 基线快照保留原样，明确其版本。

退出条件：当前文档只有一套版本、路径和参数语义；生成文档与运行时一致，且文案也正确。目前保存的 4.1a OpenAPI 与运行时精确相同（25 个 paths），说明没有结构漂移，但相同并不能排除源代码中的过时描述。

### D4 · P3 · 已脱离运行路径的旧辅助函数仍由测试维持

证据：`services/execution_service.py:184–203` 的 `_load_auth` 在运行源码中无调用方，只有 `tests/test_auth_cache_scope_and_compat.py:14–31` 直接测试它。实际执行已使用 `run_config.resolve_run_config`，历史加载也在 resolver 的 `_legacy` 分支完成。

`database/session.py:19` 仍定义固定内存 `DATABASE_URL` 常量，无读取方；实际 engine 已使用 settings。初始化另有 `init_database`、`init_engine_schema` 和模块导入建表几套入口，旧 `create_session` / `get_engine` 主要供数据库测试使用，配置方式分散。

处理：Agent 1 删除无运行调用方的 `_load_auth` 与无用常量，把 cache_scope 验证移到实际 resolver 路径；数据库初始化入口按现有调用方逐步统一。只用于测试的数据库工具可明确放到测试支持模块，避免扩大公开内部 API。

退出条件：不存在只为保住旧实现而存在的测试；文件库和内存库通过统一引擎配置策略创建。此项不单独阻断功能验收，可与当前修复一起小范围清理。

### D5 · P3 · 扁平项目详情兼容接口缺少退出约定

证据：`api/projects.py:38–47` 保留 `GET /projects/{project_id:path}`；同时已有 `GET /repos/{repo}/projects/{project}`。当前前端及矩阵使用后者；已找到旧路径调用为 `tests/test_api.py:47` 的兼容测试。运行时 OpenAPI 仍将两者都列为普通接口，未标 deprecated。仓外消费者未调查，不能据此认定可立即删除。

处理：明确是否继续支持该只读别名；若仅用于过渡，标 deprecated、记录替代路径及删除版本，核实消费者后移除。`GET /projects` 是仍在使用的项目列表，不在删除范围内。

退出条件：每个兼容接口有明确用途和保留期限；新增调用只使用规范路径。

### D6 · P2 · Git / 数据库双配置来源没有退出条件

证据：`services/project_access.py:42–75` 在无 Web 环境时返回 Git envs；`services/run_config.py:94–95,163–168` 回退 Git 环境文件和读取进程秘密的历史 load_auth。这是已接受的本批兼容，不是新发现的鉴权绕过；正常路由会检查该项目是否存在 Web 环境。

影响：同一个入口的配置来源随是否创建 Web 环境改变，维护两套路径，尚无 Web 环境的项目也不会进入设计中的“去配置环境”流程。

处理：保留纯数据 auth 构造与必要的历史格式测试；在本批 Web 资产与新矩阵就绪后明确兼容策略，例如将 Git 回退限制在显式兼容模式。不要直接删除整个 auth_loader，Web profiles 仍复用其中的校验和 SecretStr 构造。退出时间需要在集成清单写明。

退出条件：正常 Web 使用数据库配置；兼容模式可识别、有独立测试、有终止条件，不依赖隐式猜测。

### D7 · P2 · 持久化默认值及团队使用前的数据升级策略未完成

当前设置默认内存库，显式配置 DATABASE_URL 后持久化已验证。开发可接受，但运行指南需给出实际文件库配置；否则按默认启动仍会丢数据。现阶段允许显式删开发库重建、不引入迁移工具是已确认决定，不能描述成程序自动丢弃数据。

处理：在部署 / 团队使用前确定持久化默认或启动检查，并落实数据库和主密钥备份、表结构升级策略。当前不要求提前引入复杂迁移系统。

## 已清理及应保留的内容

| 对象 | 当前结论 |
|---|---|
| 项目 POST / PUT / DELETE | 已移除；项目来自 Git 清单，有否定性测试 |
| 旧 `/projects/{id}/.../run` 等执行路径 | 已移除；当前前端及矩阵已改两段式 URL，文档未全部更新 |
| Control 的旧 parse_case_yaml | 已删除；Case 语义归 Engine |
| 历史七键 auth 记录解析 | 有明确历史数据用途，保留 |
| 历史阶段 OpenAPI / 响应快照 | 是版本基线，保留并标明历史版本 |
| auth_loader 纯数据构造及 SecretStr 校验 | Web profiles 仍使用，保留 |
| Assets `_profiles.yaml` 实物、登录与配置页面 | 分别是 Agent 2/3 的既定交付，不是遗漏清理 |
| 4.1b 账号选择、connections 扩展 | 后续功能范围，不应为清债提前混入当前交付 |

## 建议执行顺序

1. Agent 1 关闭最新复审缺陷，同时完成 D1 / D3；顺带处理范围小的 D4，并给 D5 / D6 写明保留策略。
2. Agent 2/3 基于稳定契约交付资产与前端；不从过时 README 推断接口。
3. Agent 4 完成 D2 的脚本迁移与真实登录矩阵，核实 D6 兼容退出条件；团队使用前完成 D7。

技术债应有位置、影响、负责人、退出条件和验证结果。当前主要问题是多轮变更只追加新路径，调用方、旧文档和部分旧实现没有同步收尾。
