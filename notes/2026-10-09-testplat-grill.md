# 通用测试平台 testplat: Grill / Discovery Notes
Date: 2026-10-09 · Goal: 深度挖掘 testplat 需求,确保设计完整且可执行

## Summary / key decisions

**Grill 完成度**: 12/12 问题已确认 ✅

### 核心决策汇总表

| # | 问题 | MVP 方案 | Phase 2+ |
|---|------|---------|---------|
| Q0 | 补充需求 | ✅ 脱敏开关 + API+UI混合 (P0) | - |
| Q1 | 目标用户 | 3-10 人团队 | - |
| Q2 | 权限控制 | 简单方案 (只记录,不限制) | 细粒度权限控制 |
| Q3 | 部署方式 | 本地 + Docker | 云部署 |
| Q4 | 浏览器管理 | 独立会话/复用 (可配置) | - |
| Q5 | 脱敏规则 | 全局+项目级补充 | 字段级配置 |
| Q6 | Page Object | 参考设计+改进 (YAML继承) | - |
| Q7 | 步骤扩展 | API + UI 内置 | db/redis/custom 扩展 |
| Q8 | 并发执行 | 串行执行 (全局锁) | 任务队列 |
| Q9 | 失败重试 | 无自动重试 | Phase 2 支持 |
| Q10 | 报告管理 | 保存+查询,无自动清理 | 趋势分析 + 自动清理 |
| Q11 | 资产管理 | 本地编辑+上传 | Web 在线编辑 |
| Q12 | 环境隔离 | 严格隔离 (project+env二元组) | - |

### 设计约束 (强制执行)

1. **禁止打补丁**: 所有扩展必须通过插件机制
2. **隔离优先**: project+env+user 三维隔离
3. **配置驱动**: 业务逻辑通过配置文件控制,不硬编码
4. **渐进式**: MVP 功能最小化,后续版本按需扩展

### 优先级调整

**P0 (MVP 必须)**:
- 项目管理
- 配置管理 (含脱敏规则)
- 鉴权架构 (3 种内置策略)
- 单接口运行
- **API + UI 混合场景** ← 从 P2 提升
- 结果展示 (含脱敏开关)
- 执行历史查询

**P1 (下一版本)**:
- 任务队列 (支持多人并发)
- 失败重试机制
- 自动清理策略
- 趋势分析页面

**P2 (按需)**:
- Web 在线编辑
- 数据库/Redis 步骤
- 完全并发执行
- 权限细分

## Summary / key decisions

**核心约束确认**:
1. ✅ 脱敏必须是开关控制,不是全局强制
2. ✅ API + UI 混合测试是 P0 功能 (已在 testplat-new 验证)
3. ✅ 禁止打补丁 - 必须考虑扩展性和可维护性
4. ✅ 步骤类型必须是插件化的 (api/ui/db/redis/custom)

**优先级调整**:
- API + UI 混合场景从 P2 提升到 P0
- 脱敏规则配置作为配置管理模块的一部分

## Q&A log

### Q0 — 补充需求确认 (用户主动提出)

**背景**: 用户在 grill 开始前补充了两个关键需求

**Captured**:
1. **脱敏控制**:
   - 测试报告中的请求/响应需要支持脱敏开关
   - 不是全局脱敏,而是用户可选的显示/隐藏开关
   - 脱敏规则可配置 (password/token/phone 等字段)
   - 不同项目可能有不同的脱敏规则

2. **API + UI 混合测试**:
   - testplat-new 已经支持在同一个 case 中混合 API 和 UI 步骤
   - 参考 case: `/Users/jiavin/Developer/products/testplat-new/assets/scenarios/go2/order/full_pay/items_search_visible.case.yaml`
   - 典型场景: API 创建订单 → UI 搜索订单 → UI 验证可见性 → UI 清理数据
   - 必须保留这个能力,且优先级为 P0

3. **设计约束 (强制)**:
   - ❌ 禁止为实现功能而打补丁
   - ✅ 必须考虑扩展性和可维护性
   - ✅ 步骤执行器必须是策略模式 + 注册表机制
   - ✅ 支持未来扩展新的步骤类型 (db/redis/custom)

**Flags**: 
- 需要确认 Playwright 浏览器驱动的部署方式 (本地安装 vs Docker)
- 需要确认 Page Object 的管理方式 (YAML vs 代码)

**Decision**:
- 补充需求文档已创建: `docs/requirements/补充需求.md`
- API + UI 混合测试优先级提升到 P0
- 架构设计必须遵循插件化原则

### Q1 — 目标用户与团队规模

**Asked**: 平台的主要使用者是谁?团队规模多大?

**Captured**:
- **团队规模**: 3-10 人
- **主要使用者** (推测确认中):
  1. 用户本人 - 测试负责人/自动化测试工程师
  2. 团队成员 - 其他测试人员
  3. 开发人员 - 偶尔调试接口或运行回归测试
  
- **核心使用场景** (推测确认中):
  1. 日常接口调试 - 开发新功能时快速验证接口
  2. 回归测试 - 发版前批量运行核心场景
  3. 问题定位 - 测试失败时查看详细的请求/响应日志

**Implications**:
- 3-10 人规模意味着:
  - 需要基本的权限管理 (不同用户可能操作不同项目)
  - 需要操作审计 (谁修改了配置/运行了测试)
  - 但不需要复杂的 RBAC (角色权限控制)
  
- 多用户协作意味着:
  - 配置变更需要 git 历史追溯
  - 执行记录需要显示操作人
  - 可能需要并发执行隔离 (避免多人同时运行冲突)

### Q2 — 多用户协作与权限控制

**Asked**: 3-10 人团队的权限控制策略?

**Captured**:
- **✅ 采用简单方案 (MVP)**:
  1. 所有用户看到所有项目和执行记录
  2. 只记录"谁执行的",不限制"谁能执行"
  3. 配置修改通过 git commit 追溯操作人
  4. 暂不实现复杂的角色权限控制

**Decision**:
- MVP 不做复杂权限控制
- 执行记录表增加 `executed_by` 字段记录操作人
- 配置变更的 git commit 自动填写 author
- 后续可迭代加强 (项目级权限隔离/审批流程)

**Implications**:
- 数据模型简化:
  ```python
  class ExecutionRecord:
      executed_by: str  # 用户名或邮箱
      executed_at: datetime
  ```
- 前端不需要实现复杂的权限判断逻辑
- 用户登录仅用于身份标识,不做权限校验

### Q3 — 部署方式与环境

**Asked**: 平台的部署方式?有无特殊要求?

**Captured**:
- **✅ 采用混合方案**:
  1. **MVP 阶段**: 本地部署
     - 在公司内网服务器上直接运行 (uvicorn)
     - 团队通过浏览器访问 (如 http://testplat.internal:8000)
     - SQLite 数据库 + 本地文件存储
     - 数据不出内网,便于调试
  
  2. **同时提供**: docker-compose 选项
     - 提供 docker-compose.yml 和 Dockerfile
     - 便于换公司后快速搭建
     - 包含所有依赖 (Python/Node/Playwright)

**Decision**:
- 项目结构包含:
  ```
  testplat-control/
  ├── Dockerfile
  ├── docker-compose.yml
  ├── requirements.txt
  └── README.md (部署说明)
  ```

- 部署文档需包含两种方式:
  1. 快速启动: `uvicorn main:app --host 0.0.0.0 --port 8000`
  2. Docker 启动: `docker-compose up -d`

**Implications**:
- 数据持久化使用 SQLite (单机够用)
- 不需要考虑分布式/负载均衡
- Playwright 浏览器驱动需要在 Docker 镜像中预装
- 配置文件路径需要支持环境变量注入

### Q4 — API + UI 混合执行的浏览器管理

**Asked**: 浏览器生命周期怎么管理?headless 模式如何配置?

**Captured**:
- **✅ 浏览器管理支持两种模式 (可配置切换)**:
  
  1. **模式 1: 独立会话** (默认)
     - case 开始时启动浏览器
     - case 结束时关闭浏览器
     - 优点: 隔离性好,不会互相干扰
     - 缺点: 启动/关闭开销大 (每次 3-5 秒)
     - 适用场景: 并发执行、生产环境
  
  2. **模式 2: 复用模式**
     - 遇到第一个 UI 步骤时启动浏览器
     - case 结束后保持浏览器打开 (仅清理 cookies)
     - 下次运行直接复用
     - 优点: 性能好,调试方便
     - 缺点: 需要处理状态清理
     - 适用场景: 本地调试、单次运行

- **✅ Headless 模式配置化**:
  - **默认**: headless=true (服务器部署必须用无头模式)
  - **可配置**: 通过配置文件或运行时参数覆盖
  - **智能判断**: 服务器环境自动强制 headless (检测 DISPLAY 环境变量)

**用户关键洞察**:
> "部署到服务器中的话有头也不能使用"
- 说明用户清楚服务器环境无图形界面
- 需要在架构设计中考虑环境自适应

**Decision**:
- 配置结构:
  ```yaml
  # config/envs/test.yaml
  ui:
    browser_mode: isolated  # isolated | reuse
    headless: true          # true | false
    browser_type: chromium  # chromium | firefox | webkit
    timeout: 30000          # 默认超时 (ms)
  ```

- 运行时参数覆盖:
  ```bash
  # Web 界面运行时可选
  testplat run case.xxx --ui-headed  # 覆盖为有头模式
  
  # CLI 运行
  uv run testplat run case.xxx --env test --ui-headed
  ```

- 环境自适应逻辑:
  ```python
  def should_use_headless(config: dict, env: dict) -> bool:
      # 显式配置优先
      if "--ui-headed" in sys.argv:
          return False
      
      # 检测服务器环境 (无 DISPLAY)
      if not env.get("DISPLAY"):
          return True
      
      # 使用配置文件值
      return config.get("ui", {}).get("headless", True)
  ```

**Implications**:
- BrowserProvider 接口需要支持 mode 参数
- 浏览器复用需要实现状态清理逻辑:
  - 清理 cookies
  - 清理 localStorage
  - 关闭额外打开的 tab
- 前端 UI 需要提供 "有头模式" 的勾选框 (仅本地运行时显示)

### Q5 — 脱敏规则的配置粒度

**Asked**: 脱敏规则在哪个层级配置?粒度如何?

**Captured**:
- **✅ 采用全局 + 项目级补充规则**:
  
  1. **全局规则** (config/envs/_global.yaml):
     ```yaml
     redaction:
       enabled: true  # 全局开关
       fields:
         - password
         - token
         - phone
         - payment_password
         - access_token
         - refresh_token
       patterns:
         phone: "(\d{3})\d{4}(\d{4})"  # 保留前3后4
         token: "show_first_last:4:4"  # 显示前4后4
         password: "***"               # 完全隐藏
     ```
  
  2. **项目级补充** (config/envs/test/project-a.yaml):
     ```yaml
     redaction:
       fields:
         - custom_secret_field
         - internal_api_key
       # 可以覆盖全局 pattern
       patterns:
         phone: "(\d{3})\d{5}(\d{3})"  # 项目特殊要求
     ```

**Decision**:
- 规则合并逻辑: 全局 fields + 项目 fields (去重)
- 前端展示逻辑:
  ```typescript
  // 结果展示组件
  <ResponseViewer 
    data={response}
    showSensitive={userToggle}  // 用户控制的开关
    redactionRules={mergedRules}  // 后端返回的合并规则
  />
  ```

- 脱敏处理时机:
  - **存储时**: 原始数据完整保存 (便于调试)
  - **展示时**: 根据用户开关动态脱敏
  
- 后端 API 返回:
  ```json
  {
    "result": {
      "request": {"password": "123456"},
      "response": {"token": "abc...xyz"}
    },
    "redaction_rules": {
      "fields": ["password", "token"],
      "patterns": {...}
    }
  }
  ```

**Implications**:
- 执行记录表不需要存两份数据 (原始+脱敏)
- 前端实现脱敏逻辑 (根据 JSONPath 匹配字段名)
- 需要支持嵌套字段脱敏 (如 `user.phone`, `data.token`)
- 配置校验: pattern 必须是合法的正则表达式

### Q6 — Page Object 与测试数据的管理

**Asked**: Page Object 和测试数据文件如何组织和管理?

**Captured**:

**1. Page Object 组织**:
- **参考实现**: `/Users/jiavin/Developer/products/testplat-new/assets/pages/go2/home.page.yaml`
- **目录结构**: `assets/pages/{project}/{page_name}.page.yaml`
- **Page 定义结构**:
  ```yaml
  desc: go2 门户首页
  web_host: portal  # 引用配置中的 web_hosts.portal
  path: /
  elements:
    order_system_entry: 'role=link[name="订单管理系统"]'
  ```

- **元素选择器**: 使用 Playwright 的 role-based selector
  - 格式: `role=link[name="订单管理系统"]`
  - 优点: 更接近用户行为,可读性好,维护性强
  - 也支持 CSS selector: `.user-dropdown`

**2. 测试数据管理**:
- **参考实现**: `/Users/jiavin/Developer/products/testplat-new/assets/scenarios/go2/items.data.yaml`
- **目录结构**: `assets/scenarios/{project}/*.data.yaml` (与 case 就近放置)
- **数据结构设计**:
  ```yaml
  base:
    common:  # 公共配置
      order:
        receiver_text: "呵呵，18899992222，四川省..."
        express_code: "YD"
      payment:
        payment_wait_max_attempts: 240
    
    profiles:  # 数据 profile (不同变体)
      single_sku:
        items:
          - product_keyword: "小红帽"
            quantity: 1
      multi_sku:
        items:
          - product_keyword: "小红帽"
            quantity: 1
          - product_keyword: "玛莎妃"
            quantity: 1
  ```

- **数据引用方式**:
  ```yaml
  # case 文件中
  data:
    use: data.go2.items
  steps:
    - name: 创建订单
      with:
        items: "${params.profiles.multi_sku.items}"
        receiver: "${params.common.order.receiver_text}"
  ```

**用户关键洞察**:
> "可以参考不一定是复制过来，需要根据平台来设计"
- 说明不能照搬,要根据通用平台的需求重新设计
- 需要考虑跨公司复用时的通用性

**Decision - Page Object 设计改进**:

1. **支持多种选择器策略**:
   ```yaml
   # assets/pages/common/login.page.yaml
   elements:
     username_input:
       selector: "#username"
       type: css  # css | xpath | text | role
     
     login_button:
       selector: 'role=button[name="登录"]'
       type: role  # Playwright role selector (推荐)
     
     error_message:
       selector: "//div[@class='error']"
       type: xpath
   ```

2. **支持元素继承和复用**:
   ```yaml
   # assets/pages/_base/common_layout.page.yaml
   desc: 公共布局元素
   elements:
     user_menu: ".user-dropdown"
     logout_button: 'role=button[name="退出"]'
   
   # assets/pages/project-a/dashboard.page.yaml
   extends: page._base.common_layout  # 继承公共元素
   elements:
     dashboard_title: "h1.title"
   ```

3. **支持动态选择器**:
   ```yaml
   elements:
     order_row_by_no:
       selector: "tr[data-order='{{order_no}}']"
       type: template  # 支持变量替换
   ```

**Decision - 测试数据设计改进**:

1. **数据层级清晰化**:
   ```yaml
   # assets/data/common/base.data.yaml (跨项目公共数据)
   common:
     timeout: 30
     retry: 3
   
   # assets/data/project-a/orders.data.yaml (项目专属)
   base:
     common:
       receiver:
         name: "测试用户"
         phone: "13800138000"
   
   profiles:
     simple_order: {...}
     complex_order: {...}
   ```

2. **支持数据组合**:
   ```yaml
   # case 中可以组合多个 data 源
   data:
     use: 
       - data.common.base
       - data.project_a.orders
     override:
       receiver.name: "自定义名称"  # 运行时覆盖
   ```

3. **支持环境变量和敏感数据分离**:
   ```yaml
   # data 中不直接写死敏感信息
   profiles:
     payment:
       amount: 100
       # 密码从 secrets 读取,不写在 data 文件
       # password: "${secrets.accounts.buyer.password}"
   ```

**Implications**:
- Page Object 需要 YAML 解析器支持继承 (extends 字段)
- 元素选择器需要运行时模板渲染 (支持 `{{variable}}`)
- 测试数据需要支持多文件合并和覆盖机制
- 需要文档说明 Page Object 和 Data 的最佳实践

### Q7 — 步骤类型的扩展机制

**Asked**: 除了 API 和 UI,还会有哪些步骤类型?扩展机制如何设计?

**Captured**:
- **✅ MVP 阶段内置步骤类型**:
  1. `api` - API 调用步骤
  2. `ui.*` - UI 操作步骤族:
     - `ui.open` - 打开页面
     - `ui.click` - 点击元素
     - `ui.fill` - 填充输入框
     - `ui.read` - 读取元素属性/文本
     - `ui.screenshot` - 截图
     - `ui.dismiss` - 关闭弹窗
     - `ui.navigate` - 页面导航
     - `ui.wait` - 等待元素状态

- **✅ 未来扩展步骤类型** (保留扩展点):
  1. `db.*` - 数据库步骤
     - `db.query` - 查询数据
     - `db.assert` - 断言数据库状态
     - `db.execute` - 执行 SQL (插入测试数据)
  
  2. `redis.*` - Redis 步骤
     - `redis.get` - 获取缓存值
     - `redis.set` - 设置缓存
     - `redis.delete` - 清理缓存
  
  3. `wait.*` - 延迟步骤
     - `wait.seconds` - 固定时间等待
     - `wait.until` - 条件等待 (已在 case 级别支持)
  
  4. `custom` - 自定义步骤
     - 执行 Python 脚本
     - 调用外部工具/命令

**Decision - 插件化架构设计**:

```python
# testplat_engine/executor/registry.py

from abc import ABC, abstractmethod
from typing import Dict, Any

class StepExecutor(ABC):
    """步骤执行器基类"""
    
    @abstractmethod
    def execute(self, step: Step, context: Context) -> StepResult:
        """执行步骤
        
        Args:
            step: 步骤定义 (包含 kind, name, with 等)
            context: 执行上下文 (变量、鉴权、配置等)
        
        Returns:
            StepResult: 执行结果
        """
        pass
    
    def validate(self, step: Step) -> List[str]:
        """校验步骤定义 (可选)
        
        Returns:
            错误列表,空列表表示通过
        """
        return []

class StepExecutorRegistry:
    """步骤执行器注册表"""
    
    _executors: Dict[str, StepExecutor] = {}
    
    @classmethod
    def register(cls, kind: str, executor: StepExecutor):
        """注册步骤执行器"""
        cls._executors[kind] = executor
    
    @classmethod
    def get(cls, kind: str) -> StepExecutor:
        """获取步骤执行器"""
        # 支持 ui.click 这样的子类型
        # 先尝试精确匹配,再尝试前缀匹配
        if kind in cls._executors:
            return cls._executors[kind]
        
        # ui.click -> ui
        prefix = kind.split('.')[0]
        if prefix in cls._executors:
            return cls._executors[prefix]
        
        available = ', '.join(sorted(cls._executors.keys()))
        raise ValueError(
            f"Unknown step kind: {kind}\n"
            f"Available: {available}\n"
            f"To add custom step type, register with "
            f"StepExecutorRegistry.register()"
        )
    
    @classmethod
    def list_kinds(cls) -> List[str]:
        """列出所有已注册的步骤类型"""
        return sorted(cls._executors.keys())

# 内置执行器
class ApiStepExecutor(StepExecutor):
    """API 步骤执行器"""
    
    def execute(self, step: Step, context: Context) -> StepResult:
        # 1. 加载 API 契约
        # 2. 执行鉴权
        # 3. 渲染变量
        # 4. 发送请求
        # 5. 断言
        pass

class UiStepExecutor(StepExecutor):
    """UI 步骤执行器 (处理所有 ui.* 步骤)"""
    
    def execute(self, step: Step, context: Context) -> StepResult:
        # 根据 step.kind 分发到具体操作
        # ui.click -> _handle_click()
        # ui.fill -> _handle_fill()
        action = step.kind.split('.', 1)[1]  # 'ui.click' -> 'click'
        handler = getattr(self, f'_handle_{action}', None)
        if not handler:
            raise ValueError(f"Unsupported UI action: {action}")
        return handler(step, context)

# 初始化默认执行器
def init_default_executors():
    """注册 MVP 阶段的内置执行器"""
    StepExecutorRegistry.register('api', ApiStepExecutor())
    StepExecutorRegistry.register('ui', UiStepExecutor())

# 用户自定义扩展示例
class DbStepExecutor(StepExecutor):
    """数据库步骤执行器 (用户自定义)"""
    
    def execute(self, step: Step, context: Context) -> StepResult:
        # 从 context.connections 获取数据库连接
        # 执行 SQL 查询
        pass

# 注册自定义执行器
StepExecutorRegistry.register('db', DbStepExecutor())
```

**Decision - 步骤定义统一结构**:

```yaml
# 所有步骤类型遵循统一结构
steps:
  - name: 步骤名称
    kind: api  # 或 ui.click, db.query 等
    use: api.user.login  # 可选: 引用资产
    with: {...}  # 可选: 参数
    ->: result_name  # 可选: 保存结果到变量
    as: admin  # 可选: 指定身份
    assert: [...]  # 可选: 步骤级断言
```

**Implications**:
- 核心执行循环只需要:
  ```python
  for step in case.steps:
      executor = StepExecutorRegistry.get(step.kind)
      result = executor.execute(step, context)
      if step.output_name:
          context.set(step.output_name, result)
  ```

- 扩展新步骤类型无需修改核心代码:
  1. 实现 `StepExecutor` 接口
  2. 调用 `StepExecutorRegistry.register()`
  3. 在 case 中使用新的 kind

- 文档需要说明:
  - 内置步骤类型的使用方法
  - 自定义步骤执行器的开发指南
  - 扩展点注册机制

### Q8 — 并发执行与资源隔离

**Asked**: 多用户并发执行时如何避免冲突?采用什么方案?

**Captured**:
- **✅ 采用渐进式方案**:
  
  **Phase 1 - MVP: 串行执行**
  - 同一时间只允许一个 case 运行
  - 用户提交运行请求时:
    - 如果无任务运行 → 立即执行
    - 如果有任务运行 → 显示提示 "有任务正在运行,请稍候"
  - 执行记录按时间排序,不会冲突
  - 优点: 实现简单,无冲突风险
  - 缺点: 并发能力弱 (3-10 人团队可接受)

  **Phase 2 - 任务队列** (下一版本)
  - 多个用户的运行请求进入队列
  - 后台 worker 按顺序执行 (单 worker)
  - 前端显示队列位置: "您的任务排在第 3 位"
  - 优点: 不阻塞用户操作,体验更好
  - 实现: 使用简单的内存队列或 Redis Queue

  **Phase 3 - 完全并发** (如有实际需求再扩展)
  - 每个执行任务独立隔离
  - 测试数据用随机 ID/UUID 避免冲突
  - 浏览器实例池管理 (多 worker)
  - 需要大量设计工作,按需实现

**Decision - MVP 实现方案**:

```python
# testplat-control/services/execution_lock.py

import time
from typing import Optional
from dataclasses import dataclass

@dataclass
class ExecutionLock:
    """执行锁 (简单的全局锁)"""
    
    _locked: bool = False
    _current_task_id: Optional[str] = None
    _started_at: Optional[float] = None
    _executed_by: Optional[str] = None
    
    def acquire(self, task_id: str, user: str) -> bool:
        """尝试获取锁
        
        Returns:
            True if acquired, False if already locked
        """
        if self._locked:
            return False
        
        self._locked = True
        self._current_task_id = task_id
        self._started_at = time.time()
        self._executed_by = user
        return True
    
    def release(self):
        """释放锁"""
        self._locked = False
        self._current_task_id = None
        self._started_at = None
        self._executed_by = None
    
    def get_status(self) -> dict:
        """获取锁状态"""
        return {
            "locked": self._locked,
            "current_task_id": self._current_task_id,
            "running_duration": time.time() - self._started_at if self._started_at else 0,
            "executed_by": self._executed_by,
        }

# 全局单例
execution_lock = ExecutionLock()

# API 使用
@router.post("/execute/case")
async def execute_case(request: ExecuteCaseRequest, current_user: str):
    # 尝试获取锁
    task_id = generate_task_id()
    if not execution_lock.acquire(task_id, current_user):
        # 锁已被占用
        status = execution_lock.get_status()
        raise HTTPException(
            status_code=409,
            detail={
                "message": "有任务正在运行,请稍候",
                "current_task": status["current_task_id"],
                "executed_by": status["executed_by"],
                "running_duration": status["running_duration"],
            }
        )
    
    try:
        # 执行测试
        result = await execution_service.run_case(...)
        return result
    finally:
        # 无论成功失败,都释放锁
        execution_lock.release()
```

**Decision - Phase 2 升级路径** (保留设计):

```python
# 未来使用任务队列
from queue import Queue
from threading import Thread

class ExecutionQueue:
    """执行队列 (Phase 2)"""
    
    def __init__(self):
        self.queue = Queue()
        self.worker = Thread(target=self._process_queue, daemon=True)
        self.worker.start()
    
    def submit(self, task: ExecutionTask) -> str:
        """提交任务到队列"""
        task_id = generate_task_id()
        self.queue.put((task_id, task))
        return task_id
    
    def get_position(self, task_id: str) -> int:
        """获取任务在队列中的位置"""
        # 返回排队位置
        pass
    
    def _process_queue(self):
        """后台 worker 处理队列"""
        while True:
            task_id, task = self.queue.get()
            try:
                # 执行任务
                result = execute_task(task)
                # 通知前端 (WebSocket)
                notify_task_complete(task_id, result)
            finally:
                self.queue.task_done()
```

**Implications**:
- MVP 阶段:
  - 前端需要处理 409 错误并显示友好提示
  - 可以轮询检查锁状态,显示"还需等待 XX 秒"
  
- Phase 2 升级时:
  - 只需替换 `ExecutionLock` 为 `ExecutionQueue`
  - API 接口保持兼容
  - 增加 WebSocket 推送任务完成通知

- 测试数据冲突:
  - MVP 阶段串行执行,无冲突
  - Phase 3 如需并发,建议在 case 中使用:
    ```yaml
    data:
      override:
        order_prefix: "${uuid()}"  # 动态生成唯一标识
    ```

**Flags**:
- Phase 2 实现时需要考虑: 任务队列持久化 (服务重启不丢失队列)
- Phase 3 完全并发需要考虑: 浏览器实例池大小、测试数据命名冲突

### Q9 — 失败重试与错误恢复

**Asked**: 失败重试策略是什么?自动重试还是手动重新运行?

**Captured**:
- **✅ MVP 阶段不实现自动重试**:
  
  1. **Case 级别**:
     - 失败后不自动重试
     - 用户手动点击"重新运行"按钮
     - 保留 `retry_safe: false` 标记 (文档说明,但暂不影响执行逻辑)
  
  2. **步骤级别**:
     - 不自动重试失败的步骤
     - **保留 `wait_until` 机制** (步骤内的轮询等待):
       ```yaml
       - name: 等待订单状态
         use: api.order.query
         ->: order_status
         wait_until:
           until: "${steps.order_status.status} == 'SUCCESS'"
           interval_s: 2
           max: 30  # 最多轮询 30 次
       ```
     - 这不是"重试",而是"等待条件满足"
  
  3. **Teardown 保证**:
     - ✅ 无论 case 成功或失败,teardown 步骤必须执行
     - 使用 try-finally 机制确保清理逻辑运行
     - 示例: 测试完成后取消订单、清理测试数据

**Decision - 执行流程设计**:

```python
# testplat_engine/executor/case_executor.py

def execute_case(case: Case, context: Context) -> CaseResult:
    """执行 case"""
    
    steps_to_run = [*case.setup, *case.steps]
    results = []
    failed = False
    
    try:
        # 执行 setup + steps
        for step in steps_to_run:
            try:
                result = execute_step(step, context)
                results.append(result)
                
                if not result.success:
                    failed = True
                    break  # 任一步骤失败,立即停止
            
            except Exception as e:
                # 步骤执行异常
                results.append(StepResult(
                    name=step.name,
                    status=StepStatus.FAILED,
                    error=str(e),
                ))
                failed = True
                break
    
    finally:
        # ✅ 无论成功失败,都执行 teardown
        for step in case.teardown:
            try:
                result = execute_step(step, context)
                results.append(result)
            except Exception as e:
                # Teardown 失败也要记录,但不影响 case 整体状态
                results.append(StepResult(
                    name=step.name,
                    status=StepStatus.FAILED,
                    error=f"Teardown failed: {e}",
                ))
    
    return CaseResult(
        status=StepStatus.FAILED if failed else StepStatus.SUCCESS,
        steps=results,
        duration=...,
    )
```

**Decision - wait_until 实现** (保留):

```python
def execute_step_with_wait(step: Step, context: Context) -> StepResult:
    """执行步骤,支持 wait_until 轮询"""
    
    if not step.wait_until:
        # 没有 wait_until,直接执行
        return execute_step_once(step, context)
    
    # 轮询逻辑
    interval = step.wait_until.interval_s
    max_attempts = step.wait_until.max
    condition = step.wait_until.until
    
    for attempt in range(1, max_attempts + 1):
        result = execute_step_once(step, context)
        
        # 检查条件是否满足
        if evaluate_condition(condition, result, context):
            return result  # 条件满足,返回结果
        
        if attempt < max_attempts:
            time.sleep(interval)
    
    # 超过最大次数,条件仍未满足
    return StepResult(
        name=step.name,
        status=StepStatus.FAILED,
        error=f"Timeout: condition '{condition}' not met after {max_attempts} attempts",
    )
```

**Decision - 前端 UI 设计**:

```
┌─────────────────────────────────────────────┐
│  执行结果: ❌ 失败                           │
├─────────────────────────────────────────────┤
│  步骤:                                       │
│    ✓ 1. 创建订单        200ms              │
│    ✓ 2. 搜索订单        150ms              │
│    ❌ 3. 验证订单可见    5000ms  [展开]     │
│       错误: 元素未找到: .order-row         │
│    ⚠️  4. 取消订单 (cleanup)  300ms        │
│                                             │
│  [重新运行]  [查看日志]  [下载报告]         │
└─────────────────────────────────────────────┘
```

**Implications**:
- 用户体验:
  - 失败后可以立即点击"重新运行"
  - 不需要等待自动重试
  - 清晰看到哪一步失败
  
- Teardown 清理:
  - 即使 case 失败,也能清理测试数据
  - 避免垃圾数据累积
  
- wait_until 与重试的区别:
  - wait_until: 步骤内部的"等待条件满足" (如等待订单状态变更)
  - 重试: case 级别的"重新执行整个 case" (暂不实现)

**Future - Phase 2 自动重试** (保留设计):

```yaml
# case 配置 (未来可能支持)
retry_policy:
  enabled: true
  max_attempts: 3
  retry_on: [network_error, timeout]  # 只重试特定错误
  backoff: exponential  # 重试间隔: 1s, 2s, 4s

# 或全局配置
# config/envs/test.yaml
execution:
  retry:
    enabled: false  # MVP 默认关闭
    max_attempts: 2
```

### Q10 — 测试报告的保存与查询

**Asked**: 测试报告如何保存?查询需求是什么?有无清理策略?

**Captured**:
- **✅ MVP 保存策略**:
  
  1. **保存内容**:
     - 执行记录元数据 (SQLite 表)
     - 详细结果 (JSON 文件)
     - HTML 报告 (复用 testplat-new 的报告生成能力)
     - 截图文件 (UI 测试时)
     - 日志文件 (HTTP 请求/响应日志)
  
  2. **目录结构**:
     ```
     /var/testplat/
     ├── data/
     │   └── testplat.db  # SQLite 数据库
     ├── runs/
     │   ├── 2026-10-09/
     │   │   ├── run_abc123/
     │   │   │   ├── result.json       # 结构化结果
     │   │   │   ├── report.html       # HTML 报告
     │   │   │   ├── screenshots/      # UI 截图
     │   │   │   │   ├── step1_search.png
     │   │   │   │   └── step2_verify.png
     │   │   │   └── logs/
     │   │   │       └── http.log      # HTTP 日志
     │   │   └── run_def456/
     │   └── 2026-10-10/
     ```
  
  3. **SQLite 表结构**:
     ```sql
     CREATE TABLE execution_records (
         id TEXT PRIMARY KEY,
         project_id TEXT NOT NULL,
         env TEXT NOT NULL,
         asset_type TEXT NOT NULL,  -- 'api' | 'case'
         asset_id TEXT NOT NULL,
         status TEXT NOT NULL,       -- 'success' | 'failed'
         executed_by TEXT NOT NULL,
         started_at TIMESTAMP NOT NULL,
         completed_at TIMESTAMP,
         duration REAL,              -- 秒
         result_path TEXT NOT NULL,  -- 指向 result.json
         report_path TEXT,           -- 指向 report.html
         error_message TEXT,
         created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
     );
     
     CREATE INDEX idx_project_env ON execution_records(project_id, env);
     CREATE INDEX idx_status ON execution_records(status);
     CREATE INDEX idx_started_at ON execution_records(started_at);
     CREATE INDEX idx_executed_by ON execution_records(executed_by);
     ```

- **✅ MVP 查询功能**:
  
  1. **列表查询 API**:
     ```python
     GET /api/executions?
         project_id=xxx&
         env=test&
         status=failed&
         executed_by=user1&
         start_date=2026-10-01&
         end_date=2026-10-09&
         limit=50&
         offset=0
     ```
  
  2. **支持的筛选条件**:
     - 按项目筛选
     - 按环境筛选 (test/staging/prod)
     - 按状态筛选 (success/failed)
     - 按执行人筛选
     - 按日期范围筛选
     - 按资产名称搜索 (模糊匹配)
  
  3. **详情查询**:
     ```python
     GET /api/executions/{run_id}
     # 返回完整的执行结果,包括:
     # - 元数据
     # - 步骤详情
     # - 截图列表
     # - 日志文件路径
     ```

- **✅ MVP 清理策略**:
  - **不自动清理**
  - 由用户手动删除旧记录
  - 前端提供"删除"按钮 (单条或批量)
  - 删除记录时同时删除文件系统中的目录

- **✅ Phase 2 增强功能** (后期实现):
  
  1. **运行趋势页面**:
     - 某个 case 最近 N 次的执行趋势 (成功率、耗时趋势)
     - 项目整体的执行趋势 (每日执行次数、失败率)
     - 可视化图表 (折线图、柱状图)
  
  2. **自动清理策略**:
     ```yaml
     # config/cleanup.yaml
     retention:
       days: 30  # 保留 30 天
       # 或
       count: 1000  # 保留最近 1000 条
     ```
  
  3. **执行对比**:
     - 对比同一 case 两次执行的差异
     - 高亮变化的步骤

**Decision - 前端列表页设计**:

```
┌───────────────────────────────────────────────────────────┐
│  执行历史                    [删除选中] [导出]              │
├───────────────────────────────────────────────────────────┤
│  筛选: [项目: 全部 ▼] [状态: 全部 ▼] [执行人: 全部 ▼]     │
│        日期: [2026-10-01] ~ [2026-10-09]  [搜索]         │
├───────────────────────────────────────────────────────────┤
│  □  运行ID     资产名称      状态    执行人  开始时间  耗时 │
│  □  abc123    user.login   ✓ 成功  admin  14:30    2.3s │
│  □  def456    order.create ❌ 失败  user1  14:25    5.1s │
│  □  ghi789    order.search ✓ 成功  admin  14:20    3.2s │
│                                                           │
│  [上一页]  1 / 10  [下一页]                               │
└───────────────────────────────────────────────────────────┘

点击某行 → 跳转到详情页 (显示完整报告)
```

**Decision - 文件清理 API**:

```python
@router.delete("/executions/{run_id}")
async def delete_execution(run_id: str):
    """删除执行记录
    
    1. 从数据库删除记录
    2. 删除文件系统中的目录
    """
    record = db.get_execution(run_id)
    if not record:
        raise HTTPException(404, "Record not found")
    
    # 删除文件
    result_dir = Path(record.result_path).parent
    if result_dir.exists():
        shutil.rmtree(result_dir)
    
    # 删除数据库记录
    db.delete_execution(run_id)
    
    return {"message": "Deleted"}

@router.post("/executions/cleanup")
async def cleanup_old_executions(days: int = 30):
    """批量清理旧记录 (Phase 2)"""
    cutoff = datetime.now() - timedelta(days=days)
    old_records = db.query_executions(before=cutoff)
    
    deleted_count = 0
    for record in old_records:
        # 删除文件和数据库记录
        ...
        deleted_count += 1
    
    return {"deleted_count": deleted_count}
```

**Implications**:
- 磁盘空间管理:
  - MVP 阶段需要定期手动清理
  - 建议文档中说明磁盘空间评估 (每次执行约 1-5MB)
  
- 报告访问:
  - HTML 报告通过静态文件服务访问
  - 支持脱敏开关 (前端渲染时动态处理)
  
- 数据导出:
  - 支持导出执行记录为 CSV/Excel (便于分析)
  - 支持下载单次执行的完整归档 (ZIP 文件)

**User insight**:
> "后期会在专门的页面展示运行趋势"
- 说明用户有长期监控和分析的需求
- 需要保留足够的历史数据支持趋势分析
- Phase 2 的趋势分析功能优先级较高

### Q11 — 资产文件的管理方式

**Asked**: 资产文件是只读展示还是可编辑?本地编辑后如何同步到服务器?

**Captured**:
- **✅ MVP 采用方案 A: 本地编辑 + 上传**
  
  **实现方式**:
  1. **Web 界面**:
     - 资产列表展示 (只读)
     - 资产内容查看 (只读,提供代码高亮)
     - 不支持 Web 在线编辑
  
  2. **本地编辑工作流**:
     ```
     用户在本地编辑器修改 YAML
         ↓
     配置文件目录结构:
     project-config/
     ├── assets/
     │   ├── api/
     │   ├── scenarios/
     │   └── pages/
     ├── config/
     └── data/
         ↓
     通过 Web 界面或 CLI 上传到服务器
         ↓
     刷新 Web 页面看到最新内容
     ```
  
  3. **上传方式** (两种):
     - **方式 1: Web 界面批量上传**
       ```
       [选择目录] [上传] → 选择本地文件夹 → 上传全部文件
       ```
     - **方式 2: CLI 上传**
       ```bash
       testplat-control upload \
         --local-dir /path/to/project-config \
         --server http://testplat.internal:8000 \
         --project project-a \
         --env test
       ```

- **✅ 优点**:
  - 用户可以用熟悉的 IDE (VS Code/PyCharm) 编辑 YAML
  - 支持本地 git 版本控制
  - 实现简单,无需复杂的文件锁机制
  - 无需 NFS 挂载
  
- **⚠️ 约束**:
  - 不支持多人同时编辑同一文件 (冲突需要手动解决)
  - 本地和服务器的文件容易不同步

**Decision - 上传 API 设计**:

```python
@router.post("/assets/upload")
async def upload_assets(
    project_id: str,
    files: List[UploadFile],
    validate: bool = True,
):
    """批量上传资产文件
    
    Args:
        project_id: 目标项目
        files: 上传的文件列表
        validate: 是否校验文件格式
    
    Returns:
        {
            "uploaded": 5,
            "failed": 0,
            "errors": [],
            "details": [
                {"path": "api/user/login.yaml", "status": "ok"},
                ...
            ]
        }
    """
    project = await db.get_project(project_id)
    assets_root = Path(project.assets_root)
    
    results = {
        "uploaded": 0,
        "failed": 0,
        "errors": [],
        "details": [],
    }
    
    for file in files:
        try:
            # 保存文件到指定位置
            # 根据 file.filename 的路径结构恢复目录
            target_path = assets_root / file.filename
            target_path.parent.mkdir(parents=True, exist_ok=True)
            
            content = await file.read()
            
            # 校验 YAML 语法
            if validate and file.filename.endswith('.yaml'):
                try:
                    yaml.safe_load(content)
                except yaml.YAMLError as e:
                    raise ValueError(f"Invalid YAML: {e}")
            
            # 写入文件
            target_path.write_bytes(content)
            
            results["uploaded"] += 1
            results["details"].append({
                "path": file.filename,
                "status": "ok",
            })
        
        except Exception as e:
            results["failed"] += 1
            results["errors"].append(str(e))
            results["details"].append({
                "path": file.filename,
                "status": "error",
                "error": str(e),
            })
    
    return results
```

**Decision - CLI 上传工具**:

```python
# testplat-control/cli/upload.py

import click
from pathlib import Path
import requests

@click.command()
@click.option('--local-dir', required=True, help='本地项目目录')
@click.option('--server', required=True, help='服务器地址')
@click.option('--project', required=True, help='目标项目ID')
@click.option('--env', required=True, help='目标环境')
@click.option('--validate', is_flag=True, help='是否校验文件格式')
def upload(local_dir, server, project, env, validate):
    """上传项目配置到服务器"""
    
    local_path = Path(local_dir)
    if not local_path.exists():
        click.echo(f"Error: {local_dir} not found", err=True)
        return
    
    # 收集所有需要上传的文件
    files_to_upload = []
    for yaml_file in local_path.rglob('*.yaml'):
        relative_path = yaml_file.relative_to(local_path)
        with open(yaml_file, 'rb') as f:
            files_to_upload.append((relative_path, f.read()))
    
    if not files_to_upload:
        click.echo("No YAML files found")
        return
    
    # 上传到服务器
    url = f"{server}/api/assets/upload"
    
    with click.progressbar(
        files_to_upload,
        label=f"Uploading {len(files_to_upload)} files"
    ) as bar:
        for relative_path, content in bar:
            files = {'files': (str(relative_path), content)}
            params = {
                'project_id': project,
                'validate': validate,
            }
            
            response = requests.post(url, params=params, files=files)
            if response.status_code != 200:
                click.echo(f"Failed to upload {relative_path}: {response.text}", err=True)
    
    click.echo(f"✓ Upload complete")
```

**Decision - Phase 2 在线编辑** (保留设计):

```yaml
# 未来支持 Web 编辑时
features:
  web_editor: false  # MVP 关闭
  
# Phase 2 时改为:
features:
  web_editor: true
  editor_config:
    type: monaco  # VS Code 编辑器
    language: yaml
    theme: vs-dark
```

**Implications**:
- MVP 用户工作流:
  1. 本地 IDE 修改 assets 文件
  2. 通过 Web UI 或 CLI 上传到服务器
  3. 刷新 Web 页面查看最新资产
  4. 点击运行测试

- 文件管理最佳实践:
  - 用 git 管理本地项目配置
  - 上传前 commit 到 git
  - 服务器上的文件由 Web 界面管理 (用户不直接修改)

- 未来升级时:
  - 支持 Web 在线编辑 (Phase 2)
  - 需要实现文件锁和冲突检测
  - 自动 git commit 每次编辑

### Q12 — 环境隔离与多项目支持

**Asked**: 多项目、多环境如何隔离?有无冲突风险?

**Captured**:
- **✅ 采用推荐方案: 严格隔离**

**1. 项目隔离机制**:
```python
# 配置加载时强制绑定 project
class ProjectContext:
    project_id: str      # 必填,不能为空或通配符
    env: str             # 必填
    
    def __init__(self, project_id: str, env: str):
        if not project_id:
            raise ValueError("project_id 不能为空")
        if not env:
            raise ValueError("env 不能为空")
        
        # 验证 project + env 组合的合法性
        if not self._is_valid_combination(project_id, env):
            raise ValueError(f"无效的 project/env 组合: {project_id}/{env}")

# 执行前检查
def execute_case(case_id: str, project_id: str, env: str):
    # 1. 检查 case 所属项目是否匹配
    case = load_case(case_id)
    if case.project != project_id:
        raise ValueError(
            f"Case 所属项目 {case.project} "
            f"与运行环境 {project_id} 不匹配"
        )
    
    # 2. 检查环境是否存在
    if not config_manager.env_exists(env, project_id):
        raise ValueError(f"环境 {env} 在项目 {project_id} 中不存在")
    
    # 3. 加载配置 (严格按 project+env 隔离)
    config = config_manager.load_project_config(project_id, env)
    context = ProjectContext(project_id, env)
    
    # 4. 执行
    result = engine.execute_case(case_id, context, config)
```

**2. 环境隔离机制**:
```yaml
# 配置文件层级确保环境隔离
config/envs/
├── _global.yaml              # 跨环境共享 (只含identities/profiles/routes)
├── test.yaml                 # test 环境专属
│   api_hosts:
│     main: https://api-test.example.com
├── staging.yaml              # staging 环境专属
│   api_hosts:
│     main: https://api-staging.example.com
├── prod.yaml                 # prod 环境专属
│   api_hosts:
│     main: https://api-prod.example.com
└── test/
    └── project-a.yaml        # test/project-a 组合
        # 此项目在 test 环境的配置

# 确保不会混淆:
# - project-a 的 test 环境用 test.yaml + test/project-a.yaml
# - project-a 的 prod 环境用 prod.yaml + prod/project-a.yaml
```

**3. 用户隔离机制**:
```python
# 执行上下文隔离
class Context:
    """执行上下文 - 单次运行独立隔离"""
    
    def __init__(self, project_id: str, env: str, user: str):
        self.project_id = project_id
        self.env = env
        self.user = user
        
        # 独立的变量命名空间
        self._variables: Dict[str, Any] = {}
        
        # 独立的 HTTP 会话
        self.http_client = HttpClient()
        
        # 独立的浏览器实例 (如果需要)
        self.browser = None
    
    def set_variable(self, name: str, value: Any):
        """设置变量 (仅当前执行上下文有效)"""
        self._variables[name] = value
    
    def get_variable(self, name: str) -> Any:
        """获取变量"""
        return self._variables.get(name)

# 多个用户同时运行时
# 用户 A 的上下文: Context(project_id="a", env="test", user="userA")
# 用户 B 的上下文: Context(project_id="b", env="prod", user="userB")
# 完全隔离,不会互相干扰
```

**4. 配置加载隔离**:
```python
class ConfigManager:
    """配置管理器"""
    
    def load_project_config(self, project_id: str, env: str) -> dict:
        """加载项目配置
        
        流程:
        1. 加载 _global.yaml
        2. 加载 {env}.yaml
        3. 加载 {env}/{project_id}.yaml
        4. 合并 (优先级: project > env > global)
        
        确保返回的配置只包含当前 project+env 的数据
        """
        
        # 验证文件存在性
        env_file = self.config_root / f"{env}.yaml"
        project_file = self.config_root / f"{env}/{project_id}.yaml"
        
        if not env_file.exists():
            raise FileNotFoundError(f"环境配置不存在: {env_file}")
        
        if not project_file.exists():
            raise FileNotFoundError(f"项目配置不存在: {project_file}")
        
        # 加载并合并
        global_config = self._load_yaml(self.config_root / "_global.yaml")
        env_config = self._load_yaml(env_file)
        project_config = self._load_yaml(project_file)
        
        merged = self._deep_merge(
            self._deep_merge(global_config, env_config),
            project_config
        )
        
        # 标记来源 (调试用)
        merged["__project_id__"] = project_id
        merged["__env__"] = env
        
        return merged
```

**5. 执行记录隔离**:
```python
# 执行记录表结构 - 天然隔离
class ExecutionRecord:
    id: str
    project_id: str        # ← 隔离键 1
    env: str              # ← 隔离键 2
    executed_by: str      # ← 隔离键 3
    asset_type: str
    asset_id: str
    status: str
    ...

# 查询时强制加上隔离键
def get_executions(project_id: str, env: str):
    """查询某个项目在某个环境的执行记录"""
    return db.query(
        ExecutionRecord,
        project_id=project_id,
        env=env,
    )

# 防止跨项目数据泄露
@router.get("/executions")
async def list_executions(
    project_id: str,  # 必填,来自 URL 参数
    env: str,         # 必填,来自 URL 参数
):
    """必须指定 project_id 和 env,不能为空或通配符"""
    if not project_id or not env:
        raise HTTPException(400, "project_id 和 env 必填")
    
    return db.query_executions(project_id, env)
```

**Decision - 隔离验证清单**:

在设计和实现中必须遵守:

- ✅ 所有配置加载都需要 `(project_id, env)` 二元组
- ✅ 不支持跨项目的 case 或 flow 引用
- ✅ 每次执行创建独立的 `Context` 对象
- ✅ HTTP 会话、浏览器实例按用户隔离
- ✅ 执行记录必须包含 `project_id + env + user` 组合
- ✅ 数据库查询必须显式过滤 `project_id` 和 `env`
- ✅ API 不支持"全部项目"或"全部环境"的批量操作

**Implications**:
- 同一个人可以:
  - 在 test 环境下运行项目 A
  - 在 prod 环境下运行项目 B
  - 在 staging 环境下运行项目 C
  - 不会互相干扰

- 多人并发时:
  - 用户 A 在 test/project-a 运行
  - 用户 B 在 prod/project-b 运行
  - 完全隔离,无冲突

- 未来扩展时:
  - 如实现细粒度权限控制,可基于这个隔离机制
  - 如实现跨环境对比,需要显式加载两个环境的配置

## Open flags (pending input)
