# 场景步骤编辑独立复审

2026-10-21。固定本地候选：Engine main 65f227b、Control main b547c0b、Assets main aa6a561；HEAD 均等于给定基线，无新增提交。Engine 5 个跟踪文件修改，Control 16 个跟踪文件修改及新增 ScenarioStepInspector、scenarioWorkbench、模型测试和使用文档，全部纳入复审。用户 PHASE2_3_REVIEW_REPORT.md 排除并保留。

## 原候选结论

Standards：无文档硬性违规。Spec：两项 P2，修复前暂不放行。

1. scenarioWorkbench.ts:39–59 只解析整字段引用。合法插值 sku-${params.code} 穿过 flow 后，子步骤显示模板原文，丢失可编辑根来源；sku-${response.body.code} 同样丢失运行来源链接。Inspector:80–84 将其作为已知字面值显示。Engine 实际解析 sku-A。违反 ADR-009 沿调用绑定追溯与运行期来源只读要求。
2. scenarioWorkbench.ts:101–105 归组无条件允许 inputs，不检查业务来源。外层 first API 后调用 flow(previous=${first.body})；内部 next API 后断言 actual=${next.body}、expected=${inputs.previous}。校验错误归入 next，尽管 previous 来自外层 first。违反跨业务校验独立展示要求。

两反例均以候选 TypeScript ES2022 内存转译直接调用复现，Engine resolve_step_params 无网络验证插值值。精确引用、组合对象/数组、重复 flow、map 第二项定位及覆盖优先级未发现新增缺陷。

## 验证与边界

- Engine 场景 tests/engine/test_scenario.py：102 passed。
- Control 显式本地 Engine 的 tests/integration/test_scenario_flow.py：7 passed，1 条既有 Starlette 弃用警告。
- 前端 workbench、CaseExecutionPanel、scenario、api：42 passed，4 files。
- 审查子代理另跑身份定向 Engine6、Control2，包含于上述覆盖，不累加。
- Engine/Control diff --check 通过。身份路径在预检/HTTP/执行/历史一致，0.2 非空覆盖拒绝，旧 Engine 能力缺失禁用，原脱敏链路保留。
- 上轮588/527/219及浏览器8/2/8为假服务证据，本轮未机械重跑全量。

主预览8000监听PID23161（父23143），5173 PID35872。原库、凭据和进程未操作。用户报告SHA256仍为630eb25e97c60f6a49ae7afc4fad8a7c8bcc799cd11062cd97e89379fe436638。无版本/远程锁、提交/推送/合并/发布或真实业务操作。

## 后续反馈

用户追加步骤变量缓存、默认值/本次值及默认配置保存同步、紧凑横向布局要求。转入同一工作台本地修订切片，两项复审问题同时修复；以上结论是修订前原候选的独立复审结果。
