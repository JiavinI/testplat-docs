# GSB 单接口实际执行记录（2026-10-16）

已从浏览器完成「完整临时请求体 → 自动登录 → 余额查询 → 执行记录」闭环。

- 项目／环境：`main/gsb` / `test`。
- 接口：`api.gsb.account.balance`，`POST /api/account/balance`。
- 认证完整引用：`gsb.b2.primary`；分组 `gsb / b2`，组内引用名 `primary`。`gsb.b1.primary` 可以独立存在。
- 登录流程：Assets 的 `_profiles.yaml` / `gsb_login`。凭据来自 Web，保存或读取账号不会立即登录；运行时按需登录并注入 `X-TOKEN`、`X-USER-ID`。
- 补齐 Web 环境的 `unified_login`，地址沿用旧项目 test 配置。原账号 `primary` 迁入完整引用，保留原凭据，随后按用户提供的新用户名更新；本文不记录凭据。
- 旧账号曾被被测系统拒绝，返回「您的账号已注销」。更换用户名后登录成功。

实际结果：

| 验证 | 执行记录 | 结果 |
|---|---|---|
| 真实登录和余额查询 | `57b125f6-197d-43a1-982c-41f41a3990d8` | HTTP 200，`errorCode: "0"`，登录 1 次 |
| 浏览器编辑完整请求体 `{}` 后运行 | `f9f7b6fc-7abc-4961-b618-e258914dcbbd` | HTTP 200，业务成功；账号分组展示正确；无页面脚本错误 |
| 后续运行复用登录缓存 | `623ea3f2-36a5-4b59-8f49-aaba45e182af` | HTTP 200，业务成功；`session_source: cache`，登录 0 次 |

临时请求内容通过 `request_override.body` 整体替换，支持 JSON、Query、Form、Raw；任意字段可增删，实际内容记录在执行历史中。覆盖不写回 YAML，可切回模板运行。已覆盖空对象、数组、数字、布尔、null、嵌套数据、字面量占位符、原始文本及参数编码。

验证：Control 全量 379 项，Engine 全量 429 项；相关前端 45 项通过；前端 lint/build、Python ruff、Assets 校验通过。显式 JSON null 的发送修正后另复跑真实 Engine 假服务集成 19 项，通过。

Assets 本地分支 `codex/gsb-b2-auth`，最新提交 `3765345`，预览同步此分支。Control 与 Engine 修改保留在本地工作区，远程未推送。

当前预览用本地 Engine editable 源码。远程锁定的 Engine v0.5.0 尚无 `request_override`，正式发布前需要提交 Engine 并更新 Control 的依赖锁定。Apifox 风格的完整布局调整仍待下一轮实现。
