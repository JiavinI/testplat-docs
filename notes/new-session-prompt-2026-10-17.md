> 2026-10-18：用户已确认两级选择并实施。当前状态以 `notes/handoff-2026-10-18.md` 和 `tasks/current.md` 为准；下文为历史快照。

请继续 TestPlat 项目，工作区 `/Users/jiavin/Developer/products/testplat1`。

按 AGENTS.md 读取 CONTEXT.md、tasks/current.md 和最新交接。当前权威需求是 `docs/requirements/platforms-and-projects.md`，必须完整读取；旧配置盘点见 `notes/platform-config-inventory-2026-10-17.md`。

关键更正：GO2、E3E3 本身都是被测网站平台，且各有自己的 GSB，分别沿用所属网站的身份。代发后台、拿货端、GO2 管理后台各自独立入口和登录，并能管理对应业务项目订单。GO2 网站与 GO2 管理后台不同。不要实现“只有四个平台”“一个公共 GSB 下切换项目”或直接扁平新增两个 Project 的旧方案。

先核对三仓分支、diff、依赖及现有数据模型，再按权威需求评估最小实现。界面层级、表结构和增量迁移尚未定稿；原 main/gsb、账号、历史、预览库和主密钥必须保留。指定账号及用户名见需求文档；密码分别从旧 config/secrets 对应账号读取，不输出秘密，不用当前 GSB 密码覆盖。当前尚未进行配置导入。

验收聚焦 Host 和账号正确新增、加密保存、安全读回一致，不能把配置保存写成登录或订单流程已验收。测试只用隔离库。Page 执行、复杂编排、测试数据、套件、Assets 编辑保存不扩展；不远程推送、合并或发布。

此前本地依赖交付已完成，详见最新交接，不要重做。搜索商品与完整场景仍未真实验收。完成本轮后更新当前任务和交接。
