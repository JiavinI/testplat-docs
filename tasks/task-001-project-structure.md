# Task 001: 设计 testplat 项目目录结构

## 任务目标
设计 testplat 项目的完整目录结构，包括：
1. testplat-engine (Python 执行引擎)
2. testplat-control (FastAPI 后端 + React 前端)

## 背景信息
- 项目当前名称：testplat1 (后续会改名为 testplat)
- 内部统一使用 testplat 标识
- 参考项目：testplat-new (已存在)
- Git 策略：**分开的 Git Repo**（两个独立仓库）
- 团队规模：3-10 人
- 需要考虑：后续拓展、维护空间、清晰的职责分工

## 需求文档参考
- `/Users/jiavin/Developer/products/testplat1/docs/requirements/` (如果存在)
- grill 会话记录：`/Users/jiavin/Developer/products/testplat1/notes/2026-10-10-testplat-dev-workflow-grill.md`

## 具体任务

### 1. 设计目录结构
创建以下文档：
- `docs/architecture/directory-structure.md` - 完整的目录结构设计文档

文档应包括：
- 完整的目录树 (包含所有层级)
- 每个目录的职责说明
- 文件命名规范
- 模块划分原则

### 2. testplat-engine 结构
设计要点：
- 基于 Python + 六边形架构
- 包含：domain/、engine/、ports/、plugins/、adapters/
- 可独立发布为 Python 包
- 清晰的测试结构

### 3. testplat-control 结构
设计要点：
- 后端：FastAPI
- 前端：React
- 包含：API 路由、业务逻辑、数据模型、前端组件
- 清晰的前后端分离

### 4. 依赖关系
说明：
- control 如何依赖 engine
- 版本管理策略
- 本地开发环境配置

### 5. 文档组织
设计 docs/ 目录结构：
- requirements/ - 需求文档
- architecture/ - 架构设计
- guides/ - 开发指南
- api/ - API 文档

## 输出要求
1. 创建 `docs/architecture/directory-structure.md` 文档
2. 文档必须包含：
   - 完整的目录树（使用 ASCII 树形图）
   - 每个目录/文件的用途说明
   - 技术栈选择理由
   - 后续拓展建议

## 验收标准
- [ ] 文档结构清晰，易于理解
- [ ] 目录划分符合最佳实践
- [ ] 考虑了团队协作和维护性
- [ ] 包含具体的示例文件路径
- [ ] 说明了 Git 仓库的组织方式

## 注意事项
- 当前在 testplat1 目录下工作
- 内部代码使用 testplat 命名
- 后续会将 testplat1 改名为 testplat
- 两个独立 Git 仓库：testplat-engine、testplat-control
