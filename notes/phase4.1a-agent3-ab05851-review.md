# Phase 4.1a Agent 3 前端审查

日期：2026-10-16。结论：**暂不通过：Standards 2 项 P1，Spec 2 项 P2。**

- PR：<https://github.com/JiavinI/testplat-control/pull/43>
- 基点：`8b66ec5b108cad1becc3f3c21941ec24a2305581`（已通过的 Agent 1 后端）
- 审查提交：`ab0585145513b54a41f88e7ad09256c1ac7a56bf`
- 比较：`git diff 8b66ec5...ab05851`；提交为 `214aa06`、`ab05851`，38 个文件，均在 frontend 内。
- 对照：[任务](phase4.1a-agent-prompts.md) Agent 3 / §0.1、[设计稿](phase4.1-design-draft.md) §5.1 / §6 / §8.1、该提交实际后端接口及既有前端模式。
- Standards 与 Spec 两轴独立审查；重叠问题只计一次。

## Standards

### S1 · P1 · 迟到的凭据响应污染其他账号表单

位置：`frontend/src/features/environments/AccountSection.tsx:223–225`，以及 `refresh()` 的异步写状态。

触发：查看账号 A 的凭据，请求尚未返回时取消，再打开账号 B。A 响应返回后，无编辑会话代次或账号校验，直接覆盖当前 `revealed` / `credentialDraft`。B 表单显示 A 的凭据。

独立 React 复现不只验证显示：随后修改 B 的密码并保存，实际调用为 `updateAccount('main/gsb', 'test', 'other', { credentials: { username: 'account-A-user', password: 'changed-by-B-editor' }, ... })`。A 的用户名进入 B 的更新体。仅原样保存时可能被 sameCredentials 判定为未修改，不能把该场景夸大为必然提交凭据；修改任一凭据字段会提交整表。

这是异步结果与编辑目标失去关联的正确性问题，不是纯风格异味。为编辑会话标识 project/env/account/generation，关闭或切换后废弃旧响应；同样保护 refresh 和其他异步回写。补充延迟响应、取消后重开与跨账号保存的回归。

### S2 · P1 · 缓存项目间切换保留旧草稿，保存写入新项目

位置：`frontend/src/features/environments/AccountSection.tsx:135–137`（AccountTable 仅以 env 为 key）、`frontend/src/features/environments/EnvironmentSection.tsx:62–70`（草稿未按 projectId 重置）；上层 `ProjectConfigPanel` 亦未按项目重挂。

触发条件：目标项目已缓存，路由在两个项目之间切换，配置组件不因首次 loading 卸载；两项目具有同名环境和账号。项目变了，但本地 draft、revealed 与 credentialDraft 仍属于旧项目，mutation 则已经绑定新 projectId。冷加载目标项目会卸载子树，因此不以冷加载结果否定此问题。

独立复现：在项目 A 查看 gsb 凭据，切到缓存项目 `main/second`，表单仍保留 `project-A-user`。修改密码保存后，实际 `updateAccount('main/second', 'test', 'gsb', ...)` 携带旧项目的 username。两边 revision 相同时，后端版本校验不能识别这种错目标请求。

配置编辑器应在项目切换时重置或以 projectId 为 key 重挂；账号编辑器再包含 env，配合 S1 的异步失效保护。环境草稿也必须同样处理，避免只修账号。回归需覆盖目标项目命中缓存的导航。

## Spec

### F1 · P2 · 单 API 必填校验挡住环境默认参数

位置：`frontend/src/features/executions/ApiExecutionPanel.tsx:22,66`、`ParamForm.tsx` 的无条件 required 规则；新工作台接入环境列表后仍只将 env 名称传给 API 执行面板。

设计稿 §5.1.3 要求请求参数优先于环境 case_params，A5 要求 Web 修改商品关键字后重跑生效。后端已在冻结配置合并后验证必填字段，但前端尚未适配。

独立复现：缓存中 `pre.case_params.product_keyword = 'web-default'`，商品搜索参数声明 required；运行表单留空点执行，出现“请填写该参数”，`runAsset` 从未调用。用户被迫显式输入，无法使用环境已配置的该默认值。Case 的 params={} 路径不受此缺口影响。

应允许使用环境默认值满足必填项，空白时省略请求覆盖，并保留显式输入优先；不要将旧的环境默认值复制成每次都提交的请求参数，否则后续改默认值仍不会生效。补环境切换、默认值更新和显式覆盖的测试。

### F2 · P2 · 查看凭据与冲突刷新绕过 401 登录引导

位置：`frontend/src/features/environments/AccountSection.tsx:223–227,239–249`、`EnvironmentSection.tsx:138–143`。

Agent 3 任务第 1、7 项要求未登录及 401 引导登录。这些直接 API 调用不经过 QueryCache / MutationCache，全局 `dropSessionOnUnauthorized` 收不到错误；本地 catch 仅显示提示。

独立复现使用真实 app QueryClient：预置当前用户 alice，凭据接口返回 401，`authKeys.me` 仍为 alice，编辑器继续显示，RequireAuth 不会切换登录页。账号/环境刷新存在相同绕过路径。

所有直接请求也应统一处理认证失效、清会话及业务状态，同时继续保证凭据不进入 Query 缓存。补凭据查看和冲突刷新收到 401 的回归。

## 验证与已满足部分

- 固定提交提取独立快照，通过 `pnpm install --frozen-lockfile` 安装锁定依赖。
- `npm run lint`：通过。
- 原交付全部测试：24 个文件、142 项通过；审查临时测试从该次运行排除，未混入交付测试计数。
- `npm run build`：通过；有大 chunk 提示，非本轮阻断。
- `git diff --check 8b66ec5...ab05851`：通过。
- 独立复现：跨账号迟到响应、缓存项目切换、凭据 401、环境默认参数被拦截均成立；仅使用合成凭据和模拟网络。
- 注册/登录/注销、顶栏操作人、环境新增/编辑/停用、按需凭据查看、revision/409 提示、启用 DB 环境下拉、结果与历史版本展示均已实现。现有测试通过不能覆盖上述遗漏。

独立复现保存在临时目录；未修改交付源码，未推送、发布 GitHub 评论或合并。未执行真实 GSB 连通与浏览器截图验收，仍属后续集成范围。

汇总：Standards 2 项、最高 P1；Spec 2 项、最高 P2。
