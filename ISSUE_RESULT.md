# Vibe Coding Plan 模式研究报告

## 📌 执行摘要

本报告深入分析了当前主流 AI 编程工具（Cursor、OpenAI Codex）的 Plan 模式实现细节，为团队开发自己的 AI 编程助手提供参考。

---

## 一、Plan 模式的核心价值

### 1.1 为什么需要 Plan 模式？

**问题背景**：
- 直接执行代码修改风险高
- 用户无法预知 AI 要做什么
- 错误难以追溯和回滚
- 缺乏透明度和可控性

**Plan 模式的解决方案**：
- ✅ **事前审查** - 执行前展示完整计划
- ✅ **分步确认** - 关键节点需要用户批准
- ✅ **透明执行** - 每步操作都有日志
- ✅ **可验证** - 自动运行测试提供证据

---

## 二、Cursor 的 Plan 模式详解

### 2.1 工作流程图

```
┌─────────────┐
│  用户输入   │
│  需求描述   │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  AI 分析    │
│ - 读取项目  │
│ - 理解结构  │
│ - 识别模式  │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  生成计划   │
│ (Markdown)  │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  用户审查   │
│  确认/修改  │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  分步执行   │
│  显示进度   │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  测试验证   │
│  生成 diff  │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  用户确认   │
│  提交代码   │
└─────────────┘
```

### 2.2 计划的结构化格式

Cursor 生成的计划遵循特定格式：

```markdown
## Plan

### [步骤序号] [步骤名称]

**目标**: 清晰描述这一步要达成什么

**涉及文件**:
- `path/to/file1.py` - 新建/修改/删除
- `path/to/file2.js` - 新建/修改

**具体操作**:
1. 创建 User 模型类
2. 添加 email 字段（唯一索引）
3. 添加 password_hash 字段
4. 实现 __str__ 方法

**依赖**:
- 需要先完成步骤 1
- 需要安装 bcrypt 库

**测试计划**:
- 运行 test_user.py 验证模型创建
- 检查数据库迁移是否正确
```

### 2.3 关键特性

#### 2.3.1 上下文感知
- 读取 `.cursor/rules` 配置文件
- 理解项目代码风格
- 遵循现有架构模式

#### 2.3.2 智能依赖分析
```python
# AI 会识别这种依赖关系
class UserService:
    def __init__(self, db: Database):  # 依赖 Database
        self.db = db
    
    def create_user(self, email: str):  # 依赖 User 模型
        user = User(email=email)
        self.db.save(user)
```

#### 2.3.3 风险评估
AI 会对高风险操作进行标记：
- ⚠️ **数据库迁移** - 可能影响现有数据
- ⚠️ **API 修改** - 可能破坏客户端兼容性
- ⚠️ **核心逻辑** - 需要额外测试覆盖

---

## 三、OpenAI Codex 的 Plan 模式

### 3.1 架构设计

```
┌─────────────────────────────────────┐
│         用户 (ChatGPT 界面)          │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│      Codex Agent (云端沙箱)         │
│  ┌───────────────────────────────┐  │
│  │  计划生成器                    │  │
│  │  - 分析 AGENTS.md             │  │
│  │  - 理解任务需求               │  │
│  │  - 制定执行策略               │  │
│  └───────────────────────────────┘  │
│  ┌───────────────────────────────┐  │
│  │  执行引擎                      │  │
│  │  - 文件读写                   │  │
│  │  - 命令执行                   │  │
│  │  - 测试运行                   │  │
│  └───────────────────────────────┘  │
│  ┌───────────────────────────────┐  │
│  │  验证系统                      │  │
│  │  - 测试断言                   │  │
│  │  - 日志记录                   │  │
│  │  - 证据收集                   │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

### 3.2 AGENTS.md 规范

```markdown
# AGENTS.md

## 项目概述
这是一个基于 FastAPI 的用户管理系统

## 目录结构
```
src/
  ├── models/      # 数据模型
  ├── services/    # 业务逻辑
  ├── routes/      # API 端点
  └── tests/       # 测试文件
```

## 开发环境
- Python 3.11+
- 使用 Poetry 管理依赖
- 虚拟环境在 `.venv/`

## 测试命令
```bash
poetry install          # 安装依赖
poetry run pytest       # 运行测试
poetry run pytest -v    # 详细输出
poetry run coverage     # 覆盖率报告
```

## 代码规范
- 遵循 PEP 8
- 使用 Black 格式化
- 类型注解必需
- 函数 docstring 必需

## 提交规范
- Conventional Commits
- 格式：`type(scope): description`
- 示例：`feat(auth): add login endpoint`

## 特殊规则
- 数据库迁移前必须备份
- API 修改必须更新文档
- 所有公共函数必须有测试
```

### 3.3 执行证据系统

Codex 会提供详细的执行证据：

```markdown
## 执行证据

### 文件修改
```diff
--- a/src/models/user.py
+++ b/src/models/user.py
@@ -1,5 +1,8 @@
 class User:
     def __init__(self, email: str):
         self.email = email
+        self.password_hash = None
+        self.created_at = datetime.now()
```

### 测试输出
```
$ poetry run pytest tests/test_user.py -v
===================== test session starts =====================
test_create_user PASSED
test_email_unique PASSED
test_password_hash PASSED
===================== 3 passed in 0.5s =======================
```

### 终端日志
```
$ poetry install
Installing dependencies...
✓ bcrypt installed
✓ PyJWT installed
```
```

---

## 四、对比分析

### 4.1 功能对比表

| 特性 | Cursor | Codex | 我们的实现建议 |
|------|--------|-------|---------------|
| **执行环境** | 本地 IDE | 云端沙箱 | 本地 + 可选云端 |
| **计划展示** | Markdown 文本 | 结构化 UI | Markdown + UI |
| **用户干预** | 每步确认 | 异步执行 | 可配置干预级别 |
| **测试验证** | 本地运行 | 沙箱运行 | 本地 + CI 集成 |
| **代码审查** | Git diff | PR 自动创建 | Git diff + MR |
| **上下文理解** | 项目扫描 | AGENTS.md | 配置文件 + 扫描 |
| **错误处理** | 暂停询问 | 自动重试 | 分级错误处理 |
| **进度追踪** | 实时显示 | 日志查看 | 进度条 + 日志 |

### 4.2 优劣势分析

#### Cursor 优势
- ✅ 实时交互，反馈快
- ✅ 本地执行，隐私好
- ✅ 深度 IDE 集成

#### Cursor 劣势
- ❌ 依赖本地环境配置
- ❌ 无法并行处理多任务
- ❌ 错误可能影响本地环境

#### Codex 优势
- ✅ 云端沙箱，安全隔离
- ✅ 可并行处理多任务
- ✅ 不依赖本地环境
- ✅ 完整的执行证据链

#### Codex 劣势
- ❌ 延迟较高（1-30 分钟）
- ❌ 无法实时干预
- ❌ 需要上传代码到云端

---

## 五、实现建议

### 5.1 架构设计

```
┌─────────────────────────────────────────┐
│           用户界面层                     │
│  - 计划展示组件                         │
│  - 确认/修改交互                        │
│  - 进度监控面板                         │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│           计划引擎层                     │
│  - 需求分析器                           │
│  - 任务分解器                           │
│  - 依赖分析器                           │
│  - 风险评估器                           │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│           执行引擎层                     │
│  - 文件操作模块                         │
│  - 命令执行模块                         │
│  - 测试运行模块                         │
│  - 日志记录模块                         │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│           验证层                         │
│  - 测试断言                             │
│  - 代码检查                             │
│  - 差异生成                             │
└─────────────────────────────────────────┘
```

### 5.2 计划格式规范

```yaml
plan:
  version: "1.0"
  task: "添加用户登录功能"
  
  steps:
    - id: 1
      name: "创建用户模型"
      description: "定义 User 数据模型"
      files:
        - path: "src/models/user.py"
          action: "create"
      commands: []
      tests:
        - "pytest tests/test_user.py::test_user_creation"
      dependencies: []
      risk_level: "low"
      
    - id: 2
      name: "实现认证服务"
      description: "创建登录/注册逻辑"
      files:
        - path: "src/services/auth.py"
          action: "create"
      commands:
        - "pip install bcrypt"
      tests:
        - "pytest tests/test_auth.py"
      dependencies: [1]
      risk_level: "medium"
      
    - id: 3
      name: "创建 API 端点"
      description: "暴露认证接口"
      files:
        - path: "src/routes/auth.py"
          action: "create"
        - path: "src/main.py"
          action: "modify"
      commands: []
      tests:
        - "pytest tests/test_api.py"
      dependencies: [1, 2]
      risk_level: "high"
  
  rollback_plan:
    - "git revert HEAD~3..HEAD"
    - "pip uninstall bcrypt"
```

### 5.3 配置文件格式

```markdown
# AI_ASSISTANT.md

## 项目信息
- 名称：希望公司项目
- 语言：Python 3.11, TypeScript 5.0
- 框架：FastAPI, React

## 代码规范
- 遵循 PEP 8 和 ESLint
- 必须写类型注解
- 函数必须有 docstring

## 测试要求
- 单元测试覆盖率 > 80%
- 所有公共 API 必须有集成测试
- 测试命令：`pytest --cov=src`

## 安全规则
- 密码必须 hashing
- API 必须认证
- 数据库操作必须参数化

## 禁止操作
- 不删除生产数据
- 不修改配置文件
- 不提交敏感信息

## 审查级别
- 低风险：自动执行
- 中风险：执行前确认
- 高风险：需要人工审查
```

---

## 六、总结

### 6.1 核心洞察

1. **透明度是关键** - 用户需要清楚知道 AI 要做什么
2. **可控性很重要** - 提供不同级别的干预选项
3. **验证不可少** - 自动测试 + 人工审查双重保障
4. **证据链完整** - 所有操作都可追溯

### 6.2 我们的差异化机会

1. **混合执行模式** - 本地快速迭代 + 云端安全执行
2. **智能风险分级** - 基于项目上下文自动评估风险
3. **团队协作** - 计划可共享、可评论、可协作审查
4. **知识沉淀** - 每次执行的经验自动学习优化

---

## 附录

### A. 参考资料
- [Cursor 官方文档](https://docs.cursor.com)
- [OpenAI Codex 文档](https://platform.openai.com/docs/codex)
- [AGENTS.md 规范](https://github.com/openai/codex)

### B. 术语表
- **Plan 模式**: AI 在执行前生成详细计划的工作模式
- **沙箱环境**: 隔离的执行环境，防止影响主机
- **AGENTS.md**: 给 AI 的项目配置和规范文件

---

**报告生成时间**: 2026-03-22  
**作者**: 美娜  
**版本**: 1.0
