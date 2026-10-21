# Phase 4.1a Agent 3 复审

日期：2026-10-16。结论：**暂不通过：前次四项主路径已修复，补查发现账号冲突刷新的一项 P2。**

- PR：<https://github.com/JiavinI/testplat-control/pull/43>
- 审查提交：`fdeb798a05a0133c4a57d651239365624f508936`
- 增量：`git diff ab05851...fdeb798`，单个修复提交，只涉及 frontend。
- 对照：[前次审查](phase4.1a-agent3-ab05851-review.md)、[Agent 3 任务](phase4.1a-agent-prompts.md)、[设计稿](phase4.1-design-draft.md) §5.1/§5.2/§6/§8.1。

## Standards

### S1 · P2 · 冲突刷新先更新 revision，再读凭据，允许提交半更新的草稿

位置：`frontend/src/features/environments/AccountSection.tsx:294–305`；保存按钮 `523–531` 只绑定 saving，未在 refreshing 时禁用；save 入口亦无刷新完成检查。

触发：打开账号并查看凭据，修改密码保存得到 409，点击刷新。getAccount 成功后立刻 setDraft，写入最新 revision；第二个凭据请求尚未返回，旧 revealed / credentialDraft 仍保留，保存按钮可用。

独立 React 复现：初始用户名 old-user、密码 old-password；将密码改为 my-new-password，首次保存返回 409。刷新账号详情返回 revision 2，凭据请求延迟。此时点保存，实际第二次 updateAccount 带 `revision: 2` 和 `credentials: { username: 'old-user', password: 'my-new-password' }`。服务器最新凭据原应为 server-new-user / server-new-password。旧草稿取得新版本号后可通过后端乐观锁，覆盖其他人的配置。

若第二个请求失败，同样保留新 revision 与旧凭据，不能只在网络等待期间禁用保存而忽略失败后的状态。该问题违反设计稿 §5.2 的“避免两人静默覆盖”，归 Standards 的状态一致性问题，不在 Spec 重复计数。

修复要求：需要刷新凭据时，先完成账号与凭据读取、校验仍属当前编辑会话，再一起提交到组件状态。刷新期间禁止保存；失败时保留旧版本或明确使草稿不可提交，不能留下“新 revision + 旧凭据”。补充延迟响应与第二请求失败的回归，并保留已有取消/切换保护。

复现探针在独立临时副本 `agent3-standards-4ai4ui44/frontend/src/features/environments/ProjectConfigPanel.test.tsx:580`。定向测试 1 passed 表示成功证明上述不安全提交存在，不是安全断言已通过。

前次两项 P1 的主路径已修复：编辑目标包含代次、项目、环境和账号；关闭/换账号使旧响应失效；配置组件按 projectId、账号子组件按 projectId+env 重挂载，缓存项目切换回归有效。

## Spec

前次两项主路径已修复，独立阻断发现 0 项。

- 当前环境 case_params 只影响必填校验，不复制进表单或请求；空值省略，显式输入仍优先；默认值修改及切换环境有回归。
- 查看凭据、账号冲突刷新、环境冲突刷新通过统一 guard 处理 401，清会话和业务缓存；新增测试使用真实 RequireAuth 与登录页。
- 非阻断观察：取消账号 A 的凭据请求并打开 B 后，A 迟到的 401 会先被编辑目标校验丢弃，同一登录会话仍保留。已独立复现。若后续统一处理，要同时防止旧请求的 401 清除后来新登录的会话。

## 本机验证

固定提交独立快照，按 pnpm-lock.yaml 安装依赖：

- `npm run lint`：通过。
- `npm test -- --run`：24 个文件、151 项通过（较前次 +9）。
- `npm run build`：通过；现有大 chunk 提示不阻断。
- `git diff --check 8b66ec5...fdeb798`：通过。
- Standards / Spec 双轴独立复核；另用隔离副本验证上述遗漏，不计入交付测试数。

未修改交付源码、未推送、未发布 GitHub 评论或合并。真实 GSB 连通、浏览器端到端截图仍待后续集成。

汇总：Standards 1 项、最高 P2；Spec 0 项独立阻断，另有 1 项非阻断观察。
