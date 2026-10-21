# Phase 4.1a Agent 1 修复复审

日期：2026-10-15。结论：**暂不通过，修复以下残留及回归后再放行 Agent 2/3。**

- PR：<https://github.com/JiavinI/testplat-control/pull/42>
- 本轮比较：`fd131dd7d6ec89726878712956ed98021cb6bdf3` → `3a0557c2a3a365202f6d339948862727e4d764de`。
- [首次审查](phase4.1a-agent1-review.md)；[设计依据](phase4.1-design-draft.md)。
- 固定最新提交独立快照；Standards / Spec 双轴复审；未修改应用或测试源码，未合并、未发布 GitHub 评论。

## 验证结果与已修复项

- Ruff lint / format 均通过。
- 全量 `uv run pytest -q -rs`：**347 passed、1 failed**，耗时 62.17 秒。失败为下述 S2。
- R1 持久化：实际 engine 已读取 DATABASE_URL。仅在独立验证时覆盖测试子进程 cwd 为现存目录，不改源码；两个独立进程成功保存并重新读取用户、环境、账号，凭据解密成功。
- S1 明文回显：请求校验错误已去掉 input / ctx，缺字段与错误类型请求不再回显凭据。
- R2 部分修复：环境 PUT、账号 PUT 已使用条件 UPDATE；真实并发测试验证一方成功、一方 409，失败操作不多写版本和修改记录。但账号创建仍遗漏原子版本更新。
- R3 部分修复：环境参数可补齐单接口必填项，请求可覆盖环境默认；新路径存在下述版本一致性和未知字段回归。

## Standards

### S2 · P2 · 新增持久化测试写死 /workspace，导致全量测试失败

位置：`tests/test_phase4_1a_review_fixes.py:51`。

`subprocess.run(..., cwd="/workspace")` 依赖实现者机器目录。当前 macOS 无此目录，跨进程测试在子进程启动前即抛 `FileNotFoundError`，无法证明持久化，且使全量测试失败。与可在仓库运行既定验收的要求不符。

要求：省略 cwd 或从实际仓库路径 / 临时目录计算；不得靠创建全局 /workspace 目录规避。保持两个独立进程的验证方式。

## Spec

### R2a · P2 · 新建账号仍会丢失环境 revision 增量

位置：`src/testplat_control/services/environment_store.py:402`，旧 helper 在 `347–351`；新 SQL helper 只用于账号更新。

使用真实 SQLite 文件库，两个线程创建不同账号，在提交前同步：两个账号均保存成功，但环境 revision 从 1 只到 2，两条账号创建修改记录的 revision 均为 2，正确结果应到 3。旧 helper 仍是 Python 读后加一，与 A11 的配置版本语义及本轮文档“账号保存成功时原子递增环境 revision”不符。

要求：账号创建也使用事务内原子递增；验证两个不同账号并发创建，以及创建账号与环境更新并发的版本和修改记录，不得重用或倒退环境 revision。

### R3a · P2 · 两次合并环境默认值使实际参数与记录版本不一致

位置：`src/testplat_control/api/run.py:420–431`；随后 `services/execution_service.py:266–272` 再次 resolve，`services/run_config.py:126–127` 将第一次合并结果当请求覆盖。

第一次读取环境默认参数并合并为 effective 后，环境被另一个会话更新；执行服务再解析配置，却将旧 effective 当成显式请求参数覆盖新默认值。独立文件库复现得到：记录 config_revision=3、snapshot.case_params.keyword=new-default，实际 resolved.params.keyword=old-default。用户未显式覆盖该字段。这违反 A11 的一致版本和 §5.1 的配置冻结要求。

要求：仅解析一次运行配置；保留原始请求覆盖语义，在同一份冻结结果上合并、校验、执行与记录，避免将旧环境默认值升级为请求覆盖。增加在两次读取之间修改配置的验证。

### R3b · P2 · 为一个接口配置默认参数，会使其他接口无法运行

位置：`src/testplat_control/api/run.py:420–425`。

环境 case_params 是整个项目环境共享的参数默认值，本轮将其全量合并后交给拒绝未知键的校验器。HTTP 复现：环境配置 username/email 后，api.user.create 运行 200；同环境无输入参数的 api.common.health_check 运行 422，提示 unknown param username/email。

本轮还删除原测试 extra=keep 断言，并在健康检查前清空 case_params，绕过了此回归。A5 要求环境默认配置支持重跑，不应迫使用户切换接口前反复清空环境参数。

要求：区分请求显式未知键和共享环境中其他接口的默认键；按接口声明选取适用的默认值，保留必填、类型及显式未知参数校验。增加同一环境连续运行不同输入接口的验收，不通过修改环境规避。

## 下一轮验收

补齐上述路径后重新运行全量检查，并针对账号创建并发、参数冻结和跨接口默认值复验。当前原始四项并非全部关闭，不建议据此放行后续依赖工作。

汇总：Standards 1 项（最高 P2）；Spec 3 项（最高 P2）。
