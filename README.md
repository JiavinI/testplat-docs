# TestPlat 产品文档

本仓库存放 **TestPlat** 产品设计与任务文档（ADR、需求、任务、交接笔记、AI 入口等），供多方阅读与协作，无需把设计材料复制进各代码仓。

## 代码仓库（本仓不含代码）

- [JiavinI/testplat-control](https://github.com/JiavinI/testplat-control) — 控制面
- [JiavinI/testplat-engine](https://github.com/JiavinI/testplat-engine) — 执行引擎
- [JiavinI/testplat-assets](https://github.com/JiavinI/testplat-assets) — 资产与用例

本地工作副本仍在 `products/testplat1/`；本仓是可远程阅读的文档权威副本。

## 请勿放入本仓

- 密钥、凭证、`.env`、数据库 dump
- 三仓代码目录或 scenario / worktree 工作副本

## 目录概览

| 路径 | 说明 |
|------|------|
| `CONTEXT.md` / `AGENTS.md` | AI / Agent 入口与上下文 |
| `docs/` | 架构、决策（ADR）、需求、评审等 |
| `notes/` | 交接、设计草稿、会话笔记 |
| `tasks/` | 当前任务与路线图 |
