> 历史提示词，已由 [2026-10-17 版本](new-session-prompt-2026-10-17.md) 替代。下一切片以 tasks/current.md 和被测平台权威需求为准，不直接按下文开始搜索接口验收。

请继续我的 TestPlat 项目，工作区是 `/Users/jiavin/Developer/products/testplat1`。

先阅读当前任务和交接文档：
`/Users/jiavin/Developer/products/testplat1/tasks/current.md`
`/Users/jiavin/Developer/products/testplat1/notes/handoff-2026-10-16.md`
并按 `/Users/jiavin/Developer/products/testplat1/AGENTS.md` 的顺序读取 `CONTEXT.md` 和相关决策记录。

先核对 Control、Engine、Assets 三仓的实际 diff、分支和依赖，不要覆盖已有未提交工作或重做已完成内容。真实 GSB 余额接口已经跑通，auth 是 `gsb.b2.primary`，登录流程在 `_profiles.yaml`，账号与环境在 Web；无用的 `auth/test.yaml` 已删除。四个一级菜单「接口管理、Page 管理、场景测试、配置」已拆开，各功能内选择项目。

接口工作区本轮已经完成：独立接口树、接口信息、右上方环境、完整 URL、完整请求临时编辑、响应页签和历史折叠。下一步先做真实 `api.gsb.order.search_product` 验收，再做 `case.gsb.smoke` 的余额 → 搜商品两步场景。先阅读 `testplat-control/docs/api-workspace-2026-10-16.md` 和 `testplat-control/docs/local-engine-delivery-2026-10-16.md`，复述当前验收条件和依赖缺口，再推进；关键需求不清楚时暂停问我，不要扩展成全部模块重写。

同时检查交接中指出的发布缺口：当前运行用了本地 Engine 扩展，Control 仍锁旧远程版本。先整理出可审查、可复现的方案；不要未经授权远程推送、合并、发布。搜索商品接口和完整场景还未完成真实验收，不要当作已经通过。

不要改动或重建现有预览数据库、覆盖账号凭据、输出密码 / Cookie / 主密钥。测试使用隔离库。Page 执行、复杂场景编排、测试数据、套件和 Assets 编辑保存暂时不要扩展，按交接规划逐项推进。完成后更新 `tasks/current.md` 和新的交接记录。
