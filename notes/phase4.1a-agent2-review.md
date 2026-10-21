# Phase 4.1a Agent 2 资产审查

日期：2026-10-16。结论：**暂不通过，需修复新增校验脚本的一项 P2 错误输出问题。登录流程及资产行为验收通过。**

- PR：<https://github.com/JiavinI/testplat-assets/pull/1>
- 基点：`80ad73bc14a6adb6cda2a96a33e6235fa9892f40`
- 审查提交：`e9d1f366a6aa4ba9651381629666eee405543998`
- 对照：[Agent 2 任务](phase4.1a-agent-prompts.md)、[设计稿](phase4.1-design-draft.md) §4.1/§4.2。
- 依赖：Control `8b66ec5`、Engine v0.5.0 / `5231fb4` 的已安装包。
- 固定提交独立快照，双轴审查。未改业务源码、未合并、未向 GitHub 发布评论。

## Standards

### S1 · P2 · 格式错误的登录 YAML 会经异常输出泄露密码原文

位置：`scripts/validate.py:236–237`；环境 auth 的同类分支在 `291–292`；最终打印位于 `482–483`。

新增 `_validate_profiles` 将 `yaml.YAMLError` 的 `str(exc)` 加入错误列表，主函数逐条输出。PyYAML 异常包含文件片段，不能当成安全消息。行首匹配的秘密扫描不足以保证片段没有密码。

独立复现：在临时 `_profiles.yaml` 的 login 中写入合成错误行 `body: {password: Example123, x: [}`，校验返回错误列表包含合成密码 `Example123`。该列表会由主函数打印到终端 / CI 输出。没有使用实际账号、没有修改被审文件。

违反 Agent 提示词 §0.1 的“秘密不进日志”“禁止把异常原文外显”约束。即使校验的目的就是阻止错误或秘密文件进入提交，失败路径也不能把输入原文复制到日志。

修复要求：所有新增 auth/profile 校验异常采用安全格式，只保留固定说明、异常类型和可选行列；不要将 YAML 原文或异常原文写进汇总列表 / `_fail` 输出。相关构造异常输出也应复用安全格式。补充格式错误且含合成敏感值的验证，检查完整 stdout / stderr 不含该值。

未发现其它独立可操作的模块设计或命名阻断项。辅助 lint 按 Control 同等 E/F/I/N/W/UP、100 列规则通过；Assets 没有自有 Ruff 配置，未把继承机器级配置的结果作为本仓门禁。100 列格式检查有一处可压成单行的差异，属非阻断样式建议。

## Spec

通过，0 项功能缺陷。

- `_profiles.yaml` 的流程名为 `gsb_login`；两个秘密引用变为 username/password。登录 method、path、source、extract、inject、TTL 与旧 GSB 流程一致。
- API 仍为 `auth: gsb`，smoke 未改；余额文件只更新注释。
- `auth/test.yaml` 为可被历史 load_auth 接受的空 identities；不再引用服务器 TESTPLAT_GSB_*。无 Web 环境时不再依靠服务器凭据登录的行为已在 PR 明确说明，符合此批迁移方向。
- 环境模板地址不变；文档明确 Web 管理地址和凭据、同步不覆盖。
- 校验脚本支持新 profiles 与空兼容层，两种安装组合均可运行。

## 独立验收

1. 使用真实 Engine 包 + 已通过 Control，运行 `scripts/validate.py`：ALL PASSED，流程成功构造 LoginAuth。
2. 隔离 Control 导入，仅用 Engine：ALL PASSED，脚本明确提示跳过 Control schema。
3. 将实际资产快照加入临时 Git 用例仓库，正常平台注册获取 cookie；创建 Git 中不存在的 Web 环境 pre、账号 gsb、流程 gsb_login，凭据仅使用合成值。
4. 真实 PackageEngineAdapter 请求仅连接本地 ThreadingHTTPServer：余额单 API、商品搜索单 API、case.gsb.smoke 均为 success。
5. 捕获 5 个请求，其中登录 1 次；确认 query 中 username/password 来自 Web、source 为 go2、X-TOKEN/X-USER-ID 注入、搜索 keywords 映射与 case 参数覆盖正确；执行响应不含合成密码或 token。
6. `git diff --check 80ad73b...e9d1f3`：通过。

以上只证明本地编排与实际资产对接，不代表真实 GSB 已连通。真实环境与整体矩阵仍归 Agent 4。

汇总：Standards 1 项，最高 P2；Spec 0 项。
