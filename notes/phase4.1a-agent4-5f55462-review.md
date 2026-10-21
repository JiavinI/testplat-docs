# Phase 4.1a Agent 4 复审：5f55462

日期：2026-10-16。结论：**通过代码审查；上一轮剩余 P2 已关闭，当前没有阻断发现。** 未合并 PR。

- PR：https://github.com/JiavinI/testplat-control/pull/44
- 固定 head：`5f55462c104906ea576e89b7a5814c513c4c8b7b`
- 本轮基线：`6b42bb377232fd51ff7888d7685635e1971e16fe`
- diff：`git diff 6b42bb3...5f55462`，4 文件，1 个修复提交。
- 固定快照：`/var/folders/54/cn3fz6zx2x3297j5r45ph8140000gn/T/testplat-agent4-5f55-review-4izmxzch`。
- 前次报告：`notes/phase4.1a-agent4-6b42bb3-review.md`。

## Standards

0 项。未修改 Engine、Control 业务源码或前端源码。子进程日志在内存中收集，结果只包含是否通过、载体类别和会话数量；异常原文不写入结果，监听完成恢复 logger 状态。新增注入仅针对隔离测试会话且默认关闭。未发现需要报告的代码异味。

## Spec

0 项。Agent 4 第 5 项及设计稿 A12 对重启子进程的泄密检查已补齐：

- 子进程在真实注册／登录前安装 Control、Engine 日志监听；`Trace` 捕获注册、登录和凭据响应正文。
- `establish()` 返回的实际新会话被登记，连同密码与主密钥，对子进程日志、响应正文和 SQLite 扫描。
- 子进程只输出脱敏 JSON；父进程检查 stdout、stderr 中已知秘密、退出码、扫描结果和会话数量。
- 新增独立子进程注入回归，要求注入实际会话到 Engine 日志后 A12 失败、整体非零退出。

父进程 LEAK 的 sessions=2 与 A12 的 child_sessions=1 分属两个扫描进程，第三枚会话现在由子进程直接检查，不再是漏扫。

## 本机验证

| 验证 | 结果 |
|---|---|
| 锁定依赖安装 | 通过；Engine 0.5.0 / 5231fb4 |
| Ruff check / format | 通过，124 files formatted |
| 全量 pytest，显式提供已批准 Assets checkout | **365 passed，无 skipped** |
| 干净 4.1a 矩阵 | **15/15 PASS**；A12 明确 child_sessions=1，子进程三类载体 clean |
| 新增父进程／子进程注入完整矩阵回归 | 均通过（包含在 365 项中）：分别只失败 LEAK / A12 |
| 沿用上一轮方式的独立注入实验 | **成功拦截**：子进程退出 1，载体 log，整体退出 1，仅 A12 失败 |
| 独立实验公开输出脱敏检查 | 通过；会话未传播到 stderr，报告／输出不含测试密码或业务 token |
| 完整 PR diff whitespace 检查 | 通过 |

独立实验没有调用新增的 `PHASE4_1A_INJECT_CHILD_LEAK` 钩子，也未改快照代码：临时 `sitecustomize` 在重启子进程包装真实 `establish()`，取得实际 cookie 后写入 Engine warning；父进程观察器只记录返回码和脱敏 JSON，不修改返回值。上一轮同类实验错误通过，本轮正确失败。

证据均在 `/tmp/testplat-agent4-5f55-results/`：

- `pytest.log`：365 passed。
- `clean/phase4_1a-matrix-report.json`：15/15 PASS。
- `independent-proof.json`：独立注入的无秘密结论。
- `independent-child-result.json`：子进程返回 `{ok:false, carrier:log, sessions:1}`、退出 1。
- `independent-inject/phase4_1a-matrix-report.json`：仅 A12 失败。
- `independent-hook/sitecustomize.py`：临时注入与观察脚本。

本轮仅更改脚本、测试、说明和矩阵报告，未重复运行无变更的前端与 Redis 门禁。前端在 6b42bb3 本机 lint / 155 tests / build、默认浏览器登录闭环均通过；四组 Redis/HTTP 矩阵在 f21baa5 本机通过 auth 28/28、login 25/25、vcode 12/12、cache 15/15。本轮全量 pytest 也再次覆盖调用者数据库保护回归。

真实 GSB 业务成功仍不作为自动门禁结论。此处通过仅表示当前交付代码审查和上述验证通过，不代表已完成真实账号或内网业务验收。

未推送、未合并、未对外发表评论；用户工作区及原有未跟踪报告保持原样。

计数：Standards 0 项；Spec 0 项，无阻断。
