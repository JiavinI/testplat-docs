# 场景步骤编辑本地验收与自审

日期：2026-10-20。本轮为实现自审与测试验证，不冒充独立会话复审；未授权本次新功能远程交付。

基线：Engine main 65f227b、Control main b547c0b、Assets main aa6a561。Assets 未改；Engine/Control 为本地未提交候选，用户报告排除并保留。

## 需求验收

| 用户反馈 | 实现与证据 |
| --- | --- |
| 子步骤折叠 | 分层目录有独立展开按钮；折叠包含选中项的父级后回到父级；组件及真实浏览器验证 |
| 操作难理解 | 数量累加、列表整理、金额计算、筛选、检查均有业务说明；常见字段显示中文含义，原字段仍可查看 |
| 子步骤参数与外层同步 | 同一工作副本，完整覆盖链生效；重复调用、嵌套对象、map 第二项及组合数组追溯有回归；浏览器双向编辑通过 |
| 登录身份切换 | 当前环境启用账号、匿名或恢复定义；调用路径隔离、map 全部迭代生效，预检与真实假请求/历史一致；不支持的旧服务禁用切换 |
| 提取及类似操作冗余 | 连续消费同一业务调用的提取/计算/校验收进该步骤；流程开始准备归入执行前；跨步骤汇总/校验保留独立，原执行编号与顺序不变 |
| 来源/选择器/映射重复 | 主详情只保留来源、读取字段、筛选条件及输入值；删除对处理配置的重复通用映射展示 |
| 断言归属 | 本步骤结果检查收在结果处理与校验，前置检查收在所属流程；比较不同步骤结果的校验独立保留 |
| 引用来源定位 | 当前定义显示来源编号/名称并跳转展开相关目录；历史来源仅按当次 trace 编号和命名 |

## 验证

- Engine 全量 588 通过，ruff check/format 与 diff 检查通过。新增身份路径、重复调用、map、DTO 冻结与非法路径/身份零请求回归。
- Control/Assets 使用显式相邻 Engine 全量 527（432+95）通过；追加最终场景/Case API/OpenAPI 契约 51 通过。没有升级正式远程锁。
- 前端全量 219 通过；最后的显示文案/分组与路径处理继续用 5 项模型回归及 lint/build 检查。构建仅有原 bundle 大小提示，后端保留既有 13 项警告。
- 身份请求序列化新增测试先红后绿：实际请求封装原来丢弃 step_identities，现与 UI/服务端/Engine 贯通。不能仅依赖 mock 的组件提交断言。
- 隔离真实 Control + 本地 Engine + 内存库 + MockTransport 浏览器三分支通过：成功8、缺SKU2、金额不一致8次假请求；成功分支验证第二项编辑、外层恢复、备用身份实际用于两个遍历项、来源跳转。嵌套结果/历史、390px、页面异常均通过。
- 常用 5173/8000 预览实际浏览器验证折叠、map第二项、参数双向同步、身份控件可用及来源跳转；页面/API错误0、窄屏溢出0、业务请求0、共享写入0。

日志：/tmp/testplat-scenario-ux-engine.log、testplat-scenario-ux-backend.log、testplat-scenario-ux-frontend-complete.log、testplat-scenario-ux-contract-final.log、testplat-scenario-ux-browser-final.log；临时文件可能被系统清理。维护的复现入口在 Control scripts/verify_pending_workspace.py 与 frontend/scripts/e2e-pending-workspace.mjs。

## 自审与边界

检查了完整参数覆盖链、旧0.2兼容、不支持新能力的远程锁定版本、重复流程/动态 map、未知引用显示、失败停止与秘密脱敏。身份覆盖不修改 AuthTable 的凭据，不重写资产身份，不修改业务参数规则。界面只投影已由 Engine 描述校验的定义，不执行 DSL 计算或改变执行节点顺序。

原预览库升级后除 repos/projects 同步字段外其余表逐行一致；无新增表，SQLite integrity_check 为 ok、foreign_key_check 为空。env.sh/repos.yaml 字节不变，用户报告 SHA256 不变。启动脚本已显式使用本地 Engine 候选，不把它描述为正式远程依赖。新接口的远程交付仍需后续独立审查、Engine版本/提交交付及Control精确锁更新。
