# Phase 4.1a Agent 1 独立审查

审查日期：2026-10-15。结论：**暂不通过，建议 Agent 1 修复后再放行 Agent 2/3**。

- PR：<https://github.com/JiavinI/testplat-control/pull/42>（draft）
- 基点：`bb74cb2a8e90d481173dd7275510c22ca48741ae`
- 审查提交：`fd131dd7d6ec89726878712956ed98021cb6bdf3`
- 需求依据：[Phase 4.1 设计稿](phase4.1-design-draft.md)，重点为 §5.1、§7、A5、A11、A12。
- 方法：固定提交的独立快照；Standards / Spec 两轴审查；额外运行真实进程重启、文件数据库并发保存与 HTTP 请求复现。
- 范围：仅审查，未修改应用代码，未合并或向 GitHub 发布评论。

## 验证结果

`uv run ruff check .` 与 `uv run ruff format --check .` 通过；`uv run pytest -q -rs` 为 **343 passed**（12 条弃用警告）。现有测试通过，但未覆盖下列实际失败条件。

## Standards

### S1 · P2 · 请求校验错误回显明文凭据

位置：`src/testplat_control/api/environments.py:87–93`、`src/testplat_control/api/platform_session.py:25–27`。

新请求模型包含 `credentials` / `password`，应用仍使用 FastAPI 默认校验错误响应。缺少同级必填字段时，`detail[].input` 会包含整个提交对象。

独立 HTTP 复现：平台注册提交合成密码而省略 `username`，422 响应包含该明文密码；创建测试账号提交有效 `credentials` 而省略 `account_name`，422 响应包含完整凭据。这违反密码字段“不回显”声明及设计稿 §7 的凭据响应边界。

修复要求：对请求校验错误统一安全序列化，移除输入值及敏感上下文；增加缺字段、错误类型等失败请求的凭据不回显验证。未发现可单独成立的命名或模块设计阻断项。

## Spec

### R1 · P1 · 重启丢失环境、账号和平台用户

位置：`src/testplat_control/database/session.py:11–12,31–40`；新增配置 `src/testplat_control/config.py:21` 未接入实际 engine。

数据库仍硬编码为 `sqlite:///:memory:`。两个独立 Python 进程使用相同文件 `DATABASE_URL`：首进程创建用户、环境、账号后计数为 `1/1/1`，第二进程计数为 `0/0/0`，目标数据库文件未创建。此结果直接违反 A12。

`tests/test_phase4_1a_backend.py` 的“进程内重启”仅在同一进程再次读取，不能证明重启持久化。

修复要求：将配置的持久化数据库连接接入实际 engine / SessionLocal；内存库仅用于相应测试；保留既定表结构兼容检查与显式重建策略；用两个独立进程验证保存后仍能读取并解密账号。

### R2 · P1 · revision 无法阻止真实并发覆盖

位置：`src/testplat_control/services/environment_store.py:249–255`；账号更新同样位于 `386–400`，环境版本递增也需检查。

实现先在 Python 比较 revision，再执行只按主键匹配的 UPDATE。使用真实 SQLite 文件数据库和两个线程，让双方读到 revision 1 后再提交：两次保存均成功并返回 revision 2，后者覆盖前者，没有 409。违反 A11。

这不只是“未使用 SELECT FOR UPDATE”：可以不使用该锁，但必须保证检查与更新的原子性。

修复要求：采用包含预期 revision 的条件 UPDATE 并检查受影响行数，或其他有效的数据库并发控制；冲突返回 409，并回滚相关账号、环境版本及修改记录。验证真实并发请求，不能仅顺序提交两个旧 revision。

### R3 · P2 · 单接口运行在合并环境参数前拒绝缺参

位置：`src/testplat_control/api/run.py:381–382`。

路由先校验 `body.params`，之后才进入合并环境 `case_params` 的配置解析。因此环境已经配置的必填参数仍会被拒绝。

独立 HTTP 复现：为环境写入 API 所需的 `username` 和 `email` 默认值后，单接口运行提交 `params: {}`，仍返回 422，提示缺少这两个字段；未进入执行。与 §5.1 的参数优先级及 A5 不符。

修复要求：先生成有效运行参数，再按现有 API 参数规则校验；验证“环境补齐必填项”“请求覆盖环境默认值”“合并后仍缺参则拒绝”。此前报告将其列为接受偏差，但该偏差使环境参数能力在单接口执行路径不完整，建议修复。

## 放行条件与范围

上述四项修复后，以新提交复审并补充针对性验证，再决定是否放行 Agent 2/3。特别是持久化和 revision 是已写明的验收要求，不宜用当前 343 项通过代替。

未将前端、Assets `_profiles.yaml` 实物、4.1b、真实 GSB 连通列为本次缺陷；历史无 Web 环境时的兼容路径、暂不支持 connections、未带 Origin 请求放行均按本批已声明范围处理。对所有外部凭据统一最短八位的限制亦存在于既有 Engine 边界，本次未将其另列为独立新增缺陷。

汇总：Standards 1 项（最高 P2）；Spec 3 项（最高 P1）。
