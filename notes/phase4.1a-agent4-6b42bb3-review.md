# Phase 4.1a Agent 4 复审：6b42bb3

日期：2026-10-16。结论：**两项 P1 已关闭，原泄密门禁 P2 尚有重启子进程盲区，暂不整体放行。**

- PR：https://github.com/JiavinI/testplat-control/pull/44
- head：`6b42bb377232fd51ff7888d7685635e1971e16fe`
- 修复基线：`f21baa56e84712e5f6b891c9c6576e15f4625f13`
- 本轮 diff：`git diff f21baa5...6b42bb3`，27 个文件。
- 固定快照：`/var/folders/54/cn3fz6zx2x3297j5r45ph8140000gn/T/testplat-agent4-6b42-review-f6nggor2`。
- 上轮报告：`notes/phase4.1a-agent4-f21baa5-review.md`。

## Standards

0 项。未发现 §0.1 硬性规范违反或需要单列的代码异味。未改 Engine、Control 业务 src 或前端 src。临时主密钥与独立临时数据库配对；假秘密注入钩子默认关闭，报告脱敏。

## Spec

### 已关闭：正常开发代理登录失败（P1）

默认 `frontend/vite.config.ts` 已改为 `changeOrigin: false`，后端 Origin 校验保持原样。删除截图专用 `vite.screenshot.mjs`，两份截图启动脚本直接使用默认配置。

在固定快照启动真实 Control 与默认 `npm run dev`，复跑新增 Playwright 流程：

```text
direct_register: 200
default_vite_login: 200
browser_register: 200
browser_login: 200
browser_config_save: 201
browser_logout: 204
mismatched_origin: 403
ok: true
```

本机报告：`/tmp/testplat-agent4-6b42-results/vite-login-check.json`。

### 已关闭：真实资产脚本污染调用者数据库（P1）

`scripts/phase4_0_e2e_real_assets.py` 在导入 Control 前建立临时 SQLite 并设置仅用于该库的主密钥。`phase4_0_matrix.py` 也在导入前隔离数据库。

新增回归先建立含用户、环境和加密账号的调用者文件库，再以子进程运行两份独立脚本，比较前后表行数、账号密文，并用原主密钥解密。两份回归均本机通过，资产路径显式指向已批准的 f226b4a，未因缺少资产跳过。

### 部分关闭：父进程业务 token / 新会话扫描已补齐（P2）

假服务派发业务 token 时登记，注册及注销后登录也登记会话。`scan_secret_carriers` 扫描父进程收集的响应正文、日志和 SQLite。新增单元覆盖日志、SQLite、普通响应、凭据响应的 token 检测；完整矩阵注入实验验证默认干净运行通过，打开注入钩子时仅 LEAK 失败。

### 仍需修复的 P2：A12 重启子进程未纳入泄密扫描

定位：`scripts/phase4_1a_matrix.py:548–552`、`:1010–1015`、`:1193–1197`。

设计稿 A12 要求“日志不含密码 / token 原值”，Agent 4 交付第 5 项要求“响应、日志、SQLite 中无明文密码/token”。

`restart_main()` 使用 `establish()` 重新登录，实际生成第三枚平台会话，但没有登记或扫描它。父进程 A12 对子进程 stdout 只解析末行 JSON，忽略 stderr；后面的 LEAK 只扫描父进程的日志与流量。即使重启后的 Control 发生会话日志泄漏，矩阵仍可显示 A12 / LEAK 都通过。单纯在父进程扩大 banned 集合不足以覆盖这个边界。

本轮用临时 `sitecustomize.py` 包装真实 `establish()`，仅在 `PHASE4_1A_CHILD=restart` 时将刚取得的真实 cookie 写入 `testplat_engine.warning`。未改快照代码，也没有伪造矩阵结果。父进程观察钩子只检查捕获 stderr 中的值是否与子进程实际签发 cookie 的 SHA-256 一致，不更改 subprocess 返回值。结果：

```text
child_returncode: 0
actual_child_session_present_in_captured_engine_stderr: true
leak_occurrences: 1
matrix_exit: 0
passed: 15
failed: 0
A12: PASS
LEAK: PASS（tokens=3 sessions=2）
```

无秘密证据：`/tmp/agent4-6b-child-leak-proof/evidence.json`。

错误通过的矩阵报告：`/tmp/agent4-6b-child-leak-proof/out/phase4_1a-matrix-report.json`。

注入/观察钩子：`/tmp/agent4-6b-child-leak-proof/hook/sitecustomize.py`；仅使用临时测试库和假账号，没有接触用户数据库。

建议在子进程内部登记实际会话并扫描其日志、响应和 SQLite，将脱敏后的检测结果交父进程断言；或建立等价的跨进程采集机制。同时检查子进程退出状态。补充重启子进程发生真实会话泄漏时矩阵必须失败的回归，确保检测报告不回显秘密。

## 本机验证

| 验证 | 结果 |
|---|---|
| `uv sync --extra dev --locked` | 通过，Engine 0.5.0 / 5231fb4 |
| Ruff check / format | 通过，124 files formatted |
| pytest（显式提供已批准 Assets checkout） | **363 passed，无 skipped** |
| 前端 lint / vitest / build | 通过；**155 passed** |
| 默认 Vite 真实浏览器注册/登录/保存/注销 | 通过；错误 Origin 保持 403 |
| 干净 4.1a 矩阵 | **15/15 PASS**，LEAK tokens=3 sessions=2 |
| 父进程注入泄漏完整矩阵回归 | 通过：仅 LEAK 失败，进程退出非 0，报告不包含注入 token |
| 两份独立脚本调用者数据库保护回归 | 通过：行数、密文不变，原密钥仍能解密 |
| 重启子进程日志注入 | **错误通过：15/15 PASS、退出 0** |
| 完整 PR diff whitespace 检查 | 通过 |

前端测试日志：`/tmp/testplat-agent4-6b42-frontend-tests.log`。本轮常规输出：`/tmp/testplat-agent4-6b42-results/`。

旧四组 HTTP 矩阵未在本轮重复运行：相关脚本相对 f21baa5 未变，上轮已本机通过 auth 28/28、login 25/25、vcode 12/12、cache 15/15。本轮未重拍全部截图，已核查它们共用默认配置并实跑默认浏览器门禁。

真实 GSB 业务成功仍不作为自动门禁；未推送、未合并、未对外发表评论。用户工作区及原有未跟踪报告保持原样。

计数：Standards 0 项；Spec 仍有 1 项 P2。
