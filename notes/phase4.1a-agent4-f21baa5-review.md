# Phase 4.1a Agent 4 审查：f21baa5

日期：2026-10-16。结论：**暂不放行，2 项 P1、1 项 P2 待修复。**

- PR：https://github.com/JiavinI/testplat-control/pull/44
- 固定 head：`f21baa56e84712e5f6b891c9c6576e15f4625f13`
- 已审查基线：Agent 3 `e7b417d3222c432643b83a3b456f64edb6b860f2`
- diff：`git diff e7b417d...f21baa5`，49 个文件。
- 审查快照：`/var/folders/54/cn3fz6zx2x3297j5r45ph8140000gn/T/testplat-agent4-review-hrznax2i`。
- 设计依据：`notes/phase4.1a-agent-prompts.md` §0.1、§4，`notes/phase4.1-design-draft.md` §8.1，技术债审计 D2。

## Standards

阻断发现 0 项。未改 Engine、Control `src/` 或前端 `src/`；旧浏览器脚本已注明历史用途，并从当前组合门禁退出，符合 D2 允许的处理方式。没有把重复的启动导入适配代码作为阻断问题。

## Spec

### P1：正常开发入口仍无法登录，截图专用代理不能代替 A1

定位：`scripts/phase4_1a_screenshots.py:132–133`、`frontend/scripts/vite.screenshot.mjs:26`；实际原因位于基线已有的 `frontend/vite.config.ts:24` 与 `src/testplat_control/services/platform_auth.py:49–51`。

设计稿 A1 要求“注册、登录、刷新页面、注销均可用”。默认 `npm run dev` 的代理使用 `changeOrigin: true`，把 Host 改成后端地址，浏览器 Origin 保持前端地址，后端因此拒绝登录/注册。新增截图入口另用 `changeOrigin: false` 的配置，截图可以通过，但日常启动入口仍失败。

本机启动该提交的真实 Control 与默认 Vite，复现：

```text
direct_register = 200
default_vite_login = 403
Playwright browser_login = 403
visible_error = Origin 与当前站点不一致
```

这是本轮集成发现的跨批次缺陷；PR 已如实记录偏差，不能因此认定整体登录闭环通过。应统一正常启动与验收使用的代理配置，保留 Origin 校验，再通过默认入口验证注册、登录、配置保存和注销。Agent 4 的禁止改业务源码要求不构成该缺陷已解决的依据，应交对应负责方修复。

证据目录：`/var/folders/54/cn3fz6zx2x3297j5r45ph8140000gn/T/agent4-origin-review-kkn7rvyi`。

### P1：真实资产脚本沿用调用者数据库，却覆盖主密钥

定位：`scripts/phase4_0_e2e_real_assets.py:95–106`，账号写入位于 `:141–163`。

脚本随机设置 `TESTPLAT_MASTER_KEY`，但未设置独立 `DATABASE_URL`；随后导入 Control，Settings 会使用调用者环境变量或 `.env` 的数据库。临时工作目录在导入之后才创建，不能隔离数据库。按照新集成门禁指南直接运行，可能在日常文件库注册固定测试用户、同步项目并创建环境/账号；新增密文使用进程内一次性密钥，原主密钥无法读取。

复现只使用新建的审查临时库，没有操作用户数据库：传入一个文件库和原 Fernet 密钥，仅把 `_template_hosts` 替换成 `127.0.0.1:1` 避免公网。脚本返回成功，传入的文件库却留下：

```text
platform_users: 1
projects: 1
project_environments: 2
test_accounts: 2
config_changes: 4
key_was_overwritten: True
original_key_decrypt: InvalidToken（两条账号密文）
```

应在任何 Control 导入前创建独立临时数据库并明确覆盖 `DATABASE_URL`，临时密钥仅用于该库。以调用者文件库内容不变作为回归断言；同时检查其他独立 TestClient 门禁是否有相同隔离问题。

复现库：`/var/folders/54/cn3fz6zx2x3297j5r45ph8140000gn/T/agent4-external-db-proof-v32711xo/configured.db`；资产编排报告：`/tmp/agent4-spec-external-local-report2.json`。

### P2：LEAK 未扫描实际签发的业务 token 和重登录会话

定位：`scripts/phase4_1a_matrix.py:1130–1140`；签发位置 `:187–188`，重新登录位置 `:603–607`。

Agent 4 交付第 5 项要求“响应、日志、SQLite 中无明文密码/token”，A12 也要求日志不含 token 原值。当前 banned 仅包含几组密码、主密钥和首次平台 cookie，没有业务登录签发的 `token_for(PASS_A/PASS_B/NEW_PASS)`，也未加入注销后重登录的新会话 token。因此它们即使泄漏到三类载体也可能通过。

运行时注入验证未改源码：在假服务处理 `/site/login` 后，向已被矩阵监听的 `testplat_engine` logger 写入实际派发 token。注入 6 次后，矩阵仍返回 0，报告 A1–A13、REAL、LEAK 共 **15/15 PASS**。

应收集所有实际派发的业务 token 与所有新会话 cookie，扫描响应、日志和 SQLite；报告输出同样脱敏。增加故意注入 token 泄漏时门禁必须失败的回归。

注入实验报告：`/tmp/agent4-token-mutation/phase4_1a-matrix-report.json`。

## 本机验证

在固定快照安装锁定依赖，所有输出使用审查临时目录；没有切换、重置或修改用户 Control/Assets 工作区。

| 验证 | 本机结果 |
|---|---|
| `uv sync --extra dev --locked` | 通过，Engine 0.5.0 / 5231fb4 |
| Ruff check / format | 通过，122 files formatted |
| `uv run pytest -q -rs` | **358 passed** |
| 前端 lint / vitest / build | 通过，**155 passed**，build 成功 |
| 原始 `phase4_1a_matrix.py` | **15/15 PASS**，包含子进程重启读取凭据 |
| `run_real_engine_matrix.sh`，本机临时 Redis/端口/输出 | 退出 0；auth **28/28**、login **25/25**、vcode **12/12**、cache **15/15**；泄密扫描 unexpected_hits=0 / dynamic_targets=68 |
| Phase 4.0 矩阵，截图跳过 | **9 PASS / 0 FAIL / 1 SKIP** |
| Phase 4.1a 截图脚本 | 390 / 1440 共 **12 张**重跑成功；使用的是专用代理和 Fake harness |
| 默认 Vite 登录，HTTP + 真实浏览器 | **403，可复现** |
| 注入业务 token 到日志 | LEAK **错误通过** |
| 沿用调用者文件库实验 | **文件库被写入，原主密钥无法解密新增账号** |

常规矩阵和截图输出：`/tmp/testplat-agent4-review-results/`。

旧 HTTP 矩阵证据：`/var/folders/54/cn3fz6zx2x3297j5r45ph8140000gn/T/agent4-legacy-review-zau7qq14/`。

真实 GSB 业务成功不在本次自动门禁结论内。HTTP 200 也可能封装执行失败；提交中的 `metadata_only` 记录不能作为真实业务成功证据。本轮不新增真实账号连通性要求。

未推送、未合并，未对外发布审查评论。Control 原有未跟踪 `PHASE2_3_REVIEW_REPORT.md` 保留。

计数：Standards 0 项；Spec 3 项，最高 P1。
