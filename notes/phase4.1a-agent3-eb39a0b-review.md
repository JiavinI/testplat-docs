# Phase 4.1a Agent 3 再复审

日期：2026-10-16。结论：**暂不通过，同一项刷新一致性 P2 仍有在途凭据请求的遗漏分支。**

- PR：<https://github.com/JiavinI/testplat-control/pull/43>
- 审查提交：`eb39a0b8ac1facbca6726f28a0ed9ad4a41e092e`
- 固定增量：`git diff fdeb798...eb39a0b`，仅账号组件及其测试两文件。
- 对照：[前次复审](phase4.1a-agent3-fdeb798-review.md)、[设计稿](phase4.1-design-draft.md) §5.2 的并发覆盖保护、Agent 3 任务。

## Standards

### S1 · P2 · 刷新未失效在途的查看凭据请求，仍可提交新 revision 与旧密码

位置：`frontend/src/features/environments/AccountSection.tsx:297–320`；关联 `reveal()` 的 `268–272`。

本次正确修复了前次的常规触发：已展示凭据时，两个读取都完成才写组件状态；刷新期间按钮禁用，save 入口也用 ref 防止提交；凭据读取失败保留旧版本。新增两项回归有效。

但 refreshTicket 只用于刷新函数，查看凭据仍只校验编辑目标，没有刷新代次。刷新开始时以 revealed 是否已落状态决定是否读取凭据，未考虑同一编辑会话中尚未返回的 reveal 请求。

独立组件复现（仅合成凭据，模拟有先后顺序的网络响应）：

1. 打开 revision 1 账号，点查看凭据，旧凭据响应挂起。
2. 修改展示名保存，返回 409，服务器账号已更新到 revision 2。
3. 点刷新，此时 revealed 仍为 null，所以 revealedNow=false。
4. getAccount 返回 revision 2。刷新跳过凭据读取，写入新版本并恢复保存。
5. 原查看凭据请求迟到，编辑目标仍相同，旧密码被写入组件。
6. 修改用户名后保存，实际第二次 updateAccount 携带 revision 2 和 old-password。旧密码由此取得新版本号，可覆盖另一客户端的新密码。

独立探针在 `/var/folders/54/cn3fz6zx2x3297j5r45ph8140000gn/T/agent3-race-repro-6tk06qoh/frontend/src/features/environments/ProjectConfigPanel.test.tsx:676`。执行 `npm test -- --run src/features/environments/ProjectConfigPanel.test.tsx -t 'independent pending reveal'`，安全断言在 706 行失败：第二次更新体确实匹配 revision 2 与 old-password。

修复要求：刷新开始时协调或废弃在途 reveal，使其旧结果不能进入刷新后的草稿；reveal 与 refresh 应共享足以识别凭据读取版本的失效机制，并处理刷新期间新发起的 reveal。可采用请求串行化或统一读取代次等方式，不强制具体实现。保留已有关闭、换账号、换项目的保护；补上述先查看挂起、再冲突刷新、再迟到回写的回归。

这是前次 S1 的未覆盖分支，不另起无关技术债。

## Spec

0 项独立阻断；上述一致性问题只计入 Standards。前次跨账号、缓存项目切换、环境默认参数、直接请求 401 的主路径修复仍在，无新增范围扩张。此前已记录的迟到 401 非阻断观察未变。

## 本机验证

固定提交独立快照，pnpm 按 lockfile 安装：

- `npm run lint`：通过。
- `npm test -- --run`：24 个文件、153 项通过（+2）。
- `npm run build`：通过，现有大 chunk 提示不阻断。
- `git diff --check 8b66ec5...eb39a0b`：通过。
- Standards / Spec 双轴独立审读；遗漏分支在另一个临时副本复现，未混入 153 项交付测试。

未修改交付源码、未推送、未发布 GitHub 评论或合并。

汇总：Standards 1 项、最高 P2；Spec 0 项独立阻断。
