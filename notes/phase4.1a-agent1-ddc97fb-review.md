# Phase 4.1a Agent 1 第三轮审查

日期：2026-10-15。结论：**再打回：仍有一项 P2 配置读取一致性问题。**

- PR：<https://github.com/JiavinI/testplat-control/pull/42>
- 本轮增量：`3a0557c2a3a365202f6d339948862727e4d764de` → `ddc97fb62e7cbd58a32fb514ad2f27196e715b46`。
- 依据：[设计稿](phase4.1-design-draft.md) §5.1 第 2 条、A11；[上轮复审](phase4.1a-agent1-rereview.md)；[技术债清单](phase4.1a-technical-debt-audit.md)。
- 方法：固定提交独立快照，Standards / Spec 双轴复查，本机全量检查，额外真实文件 SQLite 并发验证。
- 仅审查和记录；未改应用源码、未合并、未发布 GitHub 评论。

## 本机检查

- `uv sync --extra dev --locked`：通过。
- `uv run ruff check .`：通过。
- `uv run ruff format --check .`：117 个文件通过。
- `uv run pytest -q -rs`：**354 passed**，13 条警告，61.78 秒。
- 运行时 OpenAPI 与 `docs/phase4_1a/openapi-phase4_1a.json` 精确一致；扁平项目详情已标 deprecated。

## Standards

无独立新增缺陷。本轮 D3 文档、D4 死代码和引擎配置收尾符合要求；D5–D7 已说明旧接口、兼容路径、文件库配置和退出条件。D2 旧矩阵登录迁移明确留给 Agent 4，仍须在整体交付前完成。

## Spec

### R4 · P2 · 一个 Session 事务不等于 SQLite 一致读取视图

位置：`src/testplat_control/services/run_config.py:112–134`；引擎配置在 `src/testplat_control/database/session.py:38–45`。

`resolve_run_config` 先查询环境，再查询账号。当前 Python sqlite3 默认 legacy transaction control 下，SELECT 不会真正发起数据库事务。因此即使 `Session.in_transaction()` 为 true，这两次 SELECT 仍可能读到不同提交时点的数据。仅调用一次 resolver 和在执行前 rollback，并不能保证设计稿所要求的一致视图。

真实文件 SQLite 复现，使用真实 store 方法：

1. 创建环境和一个账号，数据库状态为环境 revision 2、账号 revision 1。
2. reader 进入 resolver，读完环境后，在账号 SELECT 前暂停。
3. writer 独立会话调用 `update_account(expected_revision=1)` 并提交，原子更新为环境 revision 3、账号 revision 2。
4. reader 继续查询账号，最终冻结结果为环境 revision 2、账号 revision 2。该组合从未在数据库提交状态中存在。

本机输出：

```text
SQLAlchemy transaction: True SQLite real transaction: False
Frozen config revision: 2 Frozen account revisions: {'gsb': 2}
Actual environment revision after account update: 3
```

复现同步点是 reader 的 `do_orm_execute` 事件，在 `FROM test_accounts` SELECT 前让独立 writer 完成提交；没有修改 resolver 源码。两名审查 Agent 与主线程均独立确认此问题。

影响：执行可能组合旧环境版本与新账号材料，记录无法对应一次一致的配置状态；违反设计稿 §5.1 的一致读取视图与 A11。

修复要求：为实际数据库建立一致读取事务，或实现可验证的版本检查 / 重读机制。需覆盖环境被路由提前读入 ORM identity map 的路径；不能仅用 Session 标志证明隔离。若改变 SQLite 事务模式，须一并验证现有写入并发和锁冲突行为。

验收：在环境 SELECT 与账号 SELECT 之间提交账号修改，运行只能使用修改前或修改后的完整配置，或受控重试；不得返回旧环境与新账号的混合版本。当前在 Engine 已进入后再修改环境的测试只验证冻结后不再变化，未覆盖冻结读取过程。

## 已关闭的上轮项目

- 子进程 cwd 不再写死；跨进程保存和解密测试本机通过。
- 账号创建与更新均使用 SQL 原子递增；双账号创建及创建与环境更新混并发测试通过。
- 路由传原始请求 params，只调用一次 resolver；校验使用冻结参数。
- 单接口按声明字段选取共享环境默认值，跨接口运行正常，显式未知参数仍拒绝；旧测试的清空环境绕过已移除。
- 网络执行前释放读取事务；API / Case 入口不持事务断言通过。未发现释放后再访问过期项目 ORM 的执行路径。

配置读取的一致性是最后仍未闭合的要求。建议只修该问题并补充对应查询间并发验收，随后复核放行。Agent 2/3 继续等待；合并未获授权。

汇总：Standards 0 项；Spec 1 项，最高 P2。
